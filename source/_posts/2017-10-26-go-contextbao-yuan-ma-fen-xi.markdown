---
layout: post
title: "Go context包源码分析"
date: 2017-10-26 10:38:30 +0800
comments: true
categories: dev
tags: golang 
---
Go标准库`context`定义了一个`Context`类型，Context类型包含超时时间，取消信号和请求范围内的变量值。  
context包提供了`WithCancel`，`WithDeadline`和`WithTimeout`函数来分别返回不同类型的Context接口。    
下面对context包的源码进行分析：  

* [Context接口](#Context)
* [空Context接口对象](#emptyCtx)
* [cancelCtx](#cancelCtx)
* [WithDeadline](#WithDealine)
* [WithValue](#WithValue)

<!-- more -->

## <span id="Context">Context接口</span>
	
	type Context interface {
		// Dealine 返回任务完成的最后期限时间，如果没有设置Deadline，则ok返回false。
		Deadline() (deadline time.Time, ok bool)
		
		// Done返回一个channel，当这个Conetext代表的任务应该取消时，这个channel将会被关闭。如果这个context不应该被取消，那么Done返回nil。
		Done() <- chan struct{}
		
		// 如果Done返回的channel没有被关闭，Err返回nil。
		// 如果Done返回的channel被关闭了，Err返回一个非nil error表示原因：Canceled表示被取消，DeadlineExceeded表示达到超时时间
		Err() error
		
		// Value返回context中key对应的值。如果没有key对应的值则返回nil
		Value(key interface{}) interface{}
	}  	
	
## <span id="emptyCtx">空Context接口对象</span>
context包提供了两个函数`TODO`和`Background`来返回一个非nil、空Context对象。  
	
	var (
		background = new(emptyCtx)
		todo       = new(emptyCtx)
	)
	func Background() Context {
		return background	
	}
	func TODO() Context {
		return todo
	}
	
`emptyCtx`类型定义如下：  
	
	// 一个emptyCtx类型Context不可以取消，没有值，没有超时时间。它不是一个空struct{}类型，因为我们需要不同类型的emptyCtx变量拥有不同地址。  
	type emptyCtx int
	
   // 下面是实现的Context接口方法
	func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
		return
	}

	func (*emptyCtx) Done() <-chan struct{} {
		return nil
	}

	func (*emptyCtx) Err() error {
		return nil
	}

	func (*emptyCtx) Value(key interface{}) interface{} {
		return nil
	}

	func (e *emptyCtx) String() string {
		switch e {
			case background:
				return "context.Background"
			case todo:
				return "context.TODO"
		}
		return "unknown empty Context"
	}

## <span id="cancelCtx">cancelCtx</span>
使用`WithCancel`创建可取消Context接口  

	// WithCancel返回一个新Context接口对象和一个CancelFunc函数，当parent对象Donechannel激活或者调用CancelFunc函数时，新产生的Context对象Done返回的channel通道被激活
	func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
		c := newCancelCtx(parent)
		propagateCancel(parent, &c)
		return &c, func() { c.cancel(true, Canceled) }
	}

	// newCancelCtx 返回一个初始化的cancelCtx对象.
	func newCancelCtx(parent Context) cancelCtx {
		return cancelCtx{
			Context: parent,
			done:    make(chan struct{}),
		}
	}
	
	
	// cancelCtx可以被取消. 当cancelCtx被取消时，它同时也会取消孩子中那些实现了canceler接口的孩子
	type cancelCtx struct {
		Context
		done chan struct{} // 调用取消函数时，通道被关闭

		mu       sync.Mutex
		children map[canceler]struct{} // 第一次调用cancel函数时，children设置为nil
		err      error                 // 调用cancel函数时，设置为非空
	}
	
	
propagateCancel方法用来向父Context添加一个canceler孩子，这样当父对象结束时，同时会将孩子结束。  

	func propagateCancel(parent Context, child canceler) {
		if parent.Done() == nil {
			return // 如果parent的Done为空，则parent不可以取消，比如默认的context.BackGround和context.TODO返回的接口对象
		}
		// 使用parentCancelCtx查找context链上的父cancelContext对象
		if p, ok := parentCancelCtx(parent); ok {
			p.mu.Lock()
			if p.err != nil {
				// 父对象已经被取消，那么也就没有必要再向父对象添加，此时取消子对象
				child.cancel(false, p.err)
			} else {
				// 向父对象中添加子对象
				if p.children == nil {
					p.children = make(map[canceler]struct{})
				}
				p.children[child] = struct{}{}
			}
			p.mu.Unlock()
		} else {
		// 如果父对象不是可取消对象，那么新建goroutine等待父对象结束后，取消子对象，在目前已知的集中Context接口实现，好像不能走到此流程
			go func() {
				select {
				case <-parent.Done():
					child.cancel(false, parent.Err())
				case <-child.Done():
				}
			}()
		}
	}
	
	// 此函数判断查找父cancelCtx对象
	func parentCancelCtx(parent Context) (*cancelCtx, bool) {
		for {
			switch c := parent.(type) {
			case *cancelCtx:
				return c, true
			case *timerCtx:
				return &c.cancelCtx, true
			case *valueCtx:
				parent = c.Context
			default:
				return nil, false
			}
		}
	}

cancelCtx实现：  

	// cancelCtx结构体定义
	type cancelCtx struct {
		Context
		done chan struct{} // closed by the first cancel call.
		mu       sync.Mutex
		children map[canceler]struct{} // set to nil by the first cancel call
		err      error                 // set to non-nil by the first cancel call
	}
	
	// 实现Context接口Done方法，返回channel
	func (c *cancelCtx) Done() <-chan struct{} {
		return c.done
	}

   // 实现Context接口Err方法，返回错误原因
	func (c *cancelCtx) Err() error {
		c.mu.Lock()
		defer c.mu.Unlock()
		return c.err
	}

	// 以字符串形式打印对象
	func (c *cancelCtx) String() string {
		return fmt.Sprintf("%v.WithCancel", c.Context)
	}

    // cacel关闭c.done通道，取消所有子Context对象，并且从父对象中删除
	func (c *cancelCtx) cancel(removeFromParent bool, err error) {
		if err == nil {
			panic("context: internal error: missing cancel error")
		}	
		c.mu.Lock()
		if c.err != nil {
			c.mu.Unlock()
			return // already canceled
		}
		c.err = err
		close(c.done)
		for child := range c.children {
			// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err)
		}
		c.children = nil
		c.mu.Unlock()
		
		if removeFromParent {
			removeChild(c.Context, c)
		}
	}

## <span id="WithDeadline">WithDealine</span>
WithDeadline创建一个具有超时时间的Conetext对象

	func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc) {
		if cur, ok := parent.Deadline(); ok && cur.Before(deadline) {
		//如果父Context的超时时间更早
			return WithCancel(parent)
		}
		c := &timerCtx{
			cancelCtx: newCancelCtx(parent),
			deadline:  deadline,
		}
		propagateCancel(parent, c)
		d := time.Until(deadline)
		if d <= 0 {
			c.cancel(true, DeadlineExceeded) // 超时时间已过
			return c, func() { c.cancel(true, Canceled) }
		}
		c.mu.Lock()
		defer c.mu.Unlock()
		if c.err == nil {
			c.timer = time.AfterFunc(d, func() {
				c.cancel(true, DeadlineExceeded)
			})
		}
		return c, func() { c.cancel(true, Canceled) }
	}

## <span id="WithValue"> WithValue </span>

	
	//返回一个Context接口对象，这个对象包含一个key、value对	// key必须是可比较的
	func WithValue(parent Context, key, val interface{}) Context {
		if key == nil {
			panic("nil key")
		}
		if !reflect.TypeOf(key).Comparable() {
			panic("key is not comparable")
		}
		return &valueCtx{parent, key, val}
	}
	
	// valueCtx包含一个键值对，它实现了Value接口方法
	// 它将接口的其他方法委托给Context父对象
	type valueCtx struct {
		Context
		key, val interface{}
	}

	
	func (c *valueCtx) String() string {
		return fmt.Sprintf("%v.WithValue(%#v, %#v)", c.Context, c.key, c.val)
	}

	func (c *valueCtx) Value(key interface{}) interface{} {
		if c.key == key {
			return c.val
		}
		return c.Context.Value(key)
	}
	
## 参考
[context](https://blog.golang.org/context)