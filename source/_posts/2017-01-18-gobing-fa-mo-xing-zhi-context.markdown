---
layout: post
title: "Go并发模型之Context"
date: 2017-01-18 15:26:37 +0800
comments: true
categories: program
tags: Golang
---
Go语言中go和channel是开发高并发程序的基础。我们使用channel进行goroutine程序通讯，传送数据和控制信号，[Pipelines and Cancelation](https://blog.golang.org/pipelines)向我们讲解了如何利用Done通道来向goroutine发送结束信号：  
	
	func main() {
        var done = make(chan struct {})
        go func(done chan struct{}) {
            select {
            case <- done:
                fmt.Println("receive exit signal")
            }
        }(done)

        done <- struct {}{}
        time.Sleep(3 * time.Second)
	}

在Go服务端，每一个请求都对应会有一个goroutine进行处理。处理函数通常又会开启另外的goroutine去访问后段服务，比如数据库和RPC服务。对应一个请求创建的这一系列的goroutine通常会需要访问请求相关的信息，比如终端用户名，授权token和请求的deadline等等。当一个请求被取消或者处理超时，与这个请求相关的一些列goroutine应该尽快退出，以便系统回收资源。  
Go中`context`包提供了这样的功能，利用context包可以进行请求范围内的数据传递，发送请求信号和deadline查看。本文简单示例如何使用`context`包进行goroutine并发编程。 

<!-- more -->

##Context
`Context`接口定义如下：  
	
	// A Context carries a deadline, cancelation signal, and request-scoped values
    // across API boundaries. Its methods are safe for simultaneous use by multiple
    // goroutines.
    type Context interface {
        // Done returns a channel that is closed when this Context is canceled
        // or times out.
        Done() <-chan struct{}

        // Err indicates why this context was canceled, after the Done channel
        // is closed.
        Err() error

        // Deadline returns the time when this Context will be canceled, if any.
        Deadline() (deadline time.Time, ok bool)

        // Value returns the value associated with key or nil if none.
        Value(key interface{}) interface{}
    }

`Context`接口包含四个方法。  
当Context被取消或者超时时，`Done`返回的通道将会被关闭。  
`Err`函数返回关闭原因。    
`Deadline`返回Context将要被取消的时间，如果设置了超时时间的话,程序可以利用Deadline的返回值判断是否需要有必要开始操作，或者利用返回值设置I/O超时时间。  
`Value`方法返回与key相对应的value数据。  
`Context`接口是并发安全的。可以将同一个Context对象传递给任意goroutine。    

###派生contexts
context包提供函数从已存在Conext接口对象派生新接口对象。这些对象形成了一个树，当一个Context被取消时，所有从此Context对象派生的对象都被取消。  
`Background`是任何Context树的根；Background永远不会被取消  

    // Background returns an empty Context. It is never canceled, has no deadline,
    // and has no values. Background is typically used in main, init, and tests,
    // and as the top-level Context for incoming requests.
    func Background() Context

`WithCancle`和`WithTimeout`返回一个可以比parent Context更早取消的Context接口对象。  
###WithCancle示例  
    
    func f1(ctx context.Context) {
        fmt.Println("f1 running...")
        withCancel, cancel := context.WithCancel(ctx)
        defer cancel()
        go f2(withCancel)
        // wait f2 to run
        time.Sleep(time.Second)
        fmt.Println("f1 cancel f2", )
        cancel()
        ticker := time.NewTicker(time.Second)
        for ; ;  {
            select {
            case <- ctx.Done():
                fmt.Println("f1, parent call canceled", ctx.Err())
            case t := <- ticker.C:
                fmt.Println("f1 running at ", t)
            }

        }
    }

    func f2(ctx context.Context)  {
        fmt.Println("f2 running...")
        ticker := time.NewTicker(time.Second)
        for ; ;  {
            select {
            case <- ctx.Done():
                fmt.Println("f2, parent call canceled", ctx.Err())
                return
            case t := <- ticker.C:
                fmt.Println("f2 running at", t)
            }
        }
    }

    func main()  {
        ctx, cancel := context.WithCancel(context.Background())
        defer cancel()

        go f1(ctx)
        for {
            time.Sleep(5 * time.Second)
        }
        //cancel()
        //time.Sleep(5 * time.Second)
    }

`WithCancel`从parent派生出一个新Context接口对象和一个CancelFunc函数。调用CancelFunc函数时关闭`Done`返回的通道，通知子goroutine退出。

###WithDeadline
`WithDeadline` 返回一个Context接口对象，当设置的deadline时间到达时，`Context`对象`Done`可读。  

    func main()  {
        n := time.Now()
        ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(5 * time.Second))
        defer cancel()
        ticker := time.NewTicker(1 * time.Second)
        for {
            select {
            case <-ticker.C:
                fmt.Println("ticker timeout")
            case <-ctx.Done():
                fmt.Println("context timeout", time.Since(n))
                return
            }
        }
    }                                                                                                }

###WithTimeout示例  
`WithTimeout`与`WithDeadline`类似，当设置的超时时间到达时，`Done`可读    
    func main()  {
        ctx, cancel := context.WithTimeout(context.Background(), 5 * time.Second)
        defer cancel()
        ticker := time.NewTicker(1 * time.Second)
        for {
            select {
            case <-ticker.C:
                fmt.Println("ticker timeout")
            case <-ctx.Done():
                fmt.Println("context timeout")
                return
            }
        }
    }

###WithValue示例
使用`WithValue`来传递请求相关数据

    func WithValue(parent Context, key, val interface{}) Context
    func main()  {
        ctx := context.WithValue(context.Background(), "test", "value")
        go func(ctx context.Context) {
            fmt.Println(ctx.Value("test"))
        }(ctx)
        time.Sleep(5 * time.Second)
    }

##参考
更多关于Go并发模型的讲解：  
[Go Concurrency Patterns: Pipelines and cancellation](https://blog.golang.org/pipelines)           
[Go Concurrency Patterns](http://talks.golang.org/2012/concurrency.slide#1)           
[Advanced Go Concurrency Patterns](http://blog.golang.org/advanced-go-concurrency-patterns)        
[Go Concurrency Patterns: Context](https://blog.golang.org/context)         
