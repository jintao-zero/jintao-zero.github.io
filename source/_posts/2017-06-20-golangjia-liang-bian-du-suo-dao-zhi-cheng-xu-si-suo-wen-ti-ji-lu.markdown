---
layout: post
title: "golang加两遍读锁导致程序死锁问题分析"
date: 2017-06-20 21:42:04 +0800
comments: true
categories: dev
tags: golang
---
golang程序使用读写锁对共享数据进行互斥保护时，注意在同一程序调用栈不要对锁进行多次加锁，这样会导致程序死锁，如下的代码片段就会导致程序死锁：
  
	import (
    "sync"
    "fmt"
    "time"
    _ "net/http/pprof"
    "net/http"
	)

	var mutex sync.RWMutex

	func f()  {
    	fmt.Println("f begin to get Rlock ")
    	mutex.RLock()
    	defer mutex.RUnlock()
    	fmt.Println("f get Rlock suc")
	}

	func main() {
    	mutex.RLock()
    	fmt.Println("main get RLock suc")
    	defer mutex.RUnlock()

    	go http.ListenAndServe(":6060", nil)
    	go func() {
        	fmt.Println("other goroutine begin to get write lock")
        	mutex.Lock()
        	defer mutex.Unlock()
        	fmt.Println("other goroutine get write lock suc“)
    	}()
    	time.Sleep(time.Second)
    	f()
	}
程序执行结果如下：  
	
	MacBook-Pro-2:sync jintao$ go run main.go 
	main get RLock suc
	other goroutine begin to get write lock
	f begin to get Rlock
	
<!-- more -->
通过pprof工具查看相关goroutine调用栈：  
main goroutine阻塞在获取读锁的位置：  
	
	1 @ 0x102dfca 0x102e0ae 0x103e5a1 0x103e1a4 0x1060869 0x13119c6 0x1311bd7 0x102db7a 0x1059c01
	#	0x103e1a3	sync.runtime_Semacquire+0x33	/usr/local/go/src/runtime/sema.go:47
	#	0x1060868	sync.(*RWMutex).RLock+0x48	/usr/local/go/src/sync/rwmutex.go:43
	#	0x13119c5	main.f+0xa5			/Users/jintao/Project/test/Golang/src/goutil/sync/main.go:15
	#	0x1311bd6	main.main+0x136			/Users/jintao/Project/test/Golang/src/goutil/sync/main.go:33
	#	0x102db79	runtime.main+0x209		/usr/local/go/src/runtime/proc.go:185
子goroutine阻塞在获取写锁的位置：
	
	1 @ 0x102dfca 0x102e0ae 0x103e5a1 0x103e1a4 0x106098e 0x1311cb6 0x1059c01
	#	0x103e1a3	sync.runtime_Semacquire+0x33	/usr/local/go/src/runtime/sema.go:47
	#	0x106098d	sync.(*RWMutex).Lock+0x6d	/usr/local/go/src/sync/rwmutex.go:91
	#	0x1311cb5	main.main.func1+0xa5		/Users/jintao/Project/test/Golang/src/goutil/sync/main.go:28
通过上面的调用栈，我们发现main goroutine在第21行已经获取了读锁，当在调用的函数`f`第15行再次尝试获取读锁时，程序阻塞。创建的goroutine在第28行尝试获取写所时，协程阻塞。  
##分析
这个是读写锁的一个标准行为。在
[Wikipedia "Readers–writer lock"](https://en.wikipedia.org/wiki/Readers%E2%80%93writer_lock)词条中对读写锁进行了介绍，在`Priority policies`一节中对`reader`和`writer`加锁时的优先级策略进行了说明，不同的优先级策略会给并发和死锁带来不同的影响：  

* 读优先策略	
	读优先允许最大并发，但是如果读并发太多，可能导致写饥饿。[Concurrent Programming: Algorithms, Principles, and Foundations](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=3&cad=rja&uact=8&ved=0ahUKEwji0rOAh9bUAhUW12MKHYx9CvUQFggyMAI&url=http%3A%2F%2Fwww.beck-shop.de%2Ffachbuch%2Fleseprobe%2F9783642320262_Excerpt_001.pdf&usg=AFQjCNEskoEL2n3HKpHGYVWc_XpU4z90nw)
* 写优先策略  
	写优先策略可以避免上面读优先导致的写锁饥饿问题，如果有一个写锁着在等待，系统将会组织任何新reader加读锁成功，一旦当前已经获取的读锁释放完后，即会成功获取写锁。写优先策略相比较与读优先策略，会降低系统的并发性能。相对于读优先策略，写优先策略在实现上的效率要低，因为在获取或者释放读锁或者写锁时，都要操作多个互斥锁。  
* 不指定优先级策略
	这种策略在一定场景下的效率更高

从上面Wikipedia关于读写锁的说明看，golang中关于读写锁使用的是写优先策略，下面查看源码，对源码进行分析

##源码解析 
读写锁实现在sync/rwmutex.go文件中
	
	type RWMutex struct {
		w           Mutex  // 获取写锁时，必须获取的互斥量
		writerSem   uint32 // 写锁信号量
		readerSem   uint32 // 读锁信号量
		readerCount int32  // 目前获取的读锁个数
		readerWait  int32  // 需要等待reader释放锁个数
	}
`加载读锁：` 
 
	// RLock locks rw for reading.
	func (rw *RWMutex) RLock() {
		if race.Enabled {
			_ = rw.w.state
			race.Disable()
		}
		if atomic.AddInt32(&rw.readerCount, 1) < 0 {
			// A writer is pending, wait for it.
			runtime_Semacquire(&rw.readerSem)
		}
		if race.Enabled {
			race.Enable()
			race.Acquire(unsafe.Pointer(&rw.readerSem))
		}
	}
每次获取读锁时，都对readerCount计数器进行加1，如果加1后值为负，那么说明有人正在获取写锁。等待readerSem信号量变为正

`释放读锁：`  
	
	func (rw *RWMutex) RUnlock() {
		if race.Enabled {
			_ = rw.w.state
			race.ReleaseMerge(unsafe.Pointer(&rw.writerSem))
			race.Disable()
		}
		if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
			if r+1 == 0 || r+1 == -rwmutexMaxReaders {
				race.Enable()
				panic("sync: RUnlock of unlocked RWMutex")
			}
			// A writer is pending.
			if atomic.AddInt32(&rw.readerWait, -1) == 0 {
				// The last reader unblocks the writer.
				runtime_Semrelease(&rw.writerSem)
			}
		}
		if race.Enabled {
			race.Enable()
		}
	}
释放写锁时，对readerCount减1，说明获取释放一个读锁，如果最新值为负，那么说明有人正在尝试获取写锁，readerWait记录了写锁需要等待的读锁个数，此时将readerWait个数减1，如果个素减为0，说明写锁正在等待的所有reader都已经释放读锁，此时可以释放写锁信号量，让程序获取写锁。 
 
`获取写锁：`  
	
	func (rw *RWMutex) Lock() {
		if race.Enabled {
			_ = rw.w.state
			race.Disable()
		}
		// First, resolve competition with other writers.
		rw.w.Lock()
		// Announce to readers there is a pending writer.
		r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
		// Wait for active readers.
		if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
			runtime_Semacquire(&rw.writerSem)
		}
		if race.Enabled {
			race.Enable()
			race.Acquire(unsafe.Pointer(&rw.readerSem))
			race.Acquire(unsafe.Pointer(&rw.writerSem))
		}
	}
获取写锁时，首先获取互斥锁，这样其他写锁必须等待。然后设置读锁个数为负值，这样后面的reader就无法获取到读锁。如果读锁没有释放，则等待写锁信号量变为正。  

`释放写锁：`
	
	func (rw *RWMutex) Unlock() {
		if race.Enabled {
			_ = rw.w.state
			race.Release(unsafe.Pointer(&rw.readerSem))
			race.Release(unsafe.Pointer(&rw.writerSem))
			race.Disable()
		}

		// Announce to readers there is no active writer.
		r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
		if r >= rwmutexMaxReaders {
			race.Enable()
			panic("sync: Unlock of unlocked RWMutex")
		}
		// Unblock blocked readers, if any.
		for i := 0; i < int(r); i++ {
			runtime_Semrelease(&rw.readerSem)
		}
		// Allow other writers to proceed.
		rw.w.Unlock()
		if race.Enabled {
			race.Enable()
		}
	}
释放写锁时，将等待获取读锁的个数变为实际值，根据等待获取读锁的个数，释放读锁信号量。

##结论
根据上面对读写锁源码的分析，我们可以看到golang中读写锁是使用写锁优先策略的，博文开头的例子中，main主协程在已经获取读锁的情况，又尝试第二次获取读锁，因为已经创建了一个子协程在获取写锁，那么第二次获取读锁操作将被阻塞，这样就导致主协程无法释放第一次获取的读锁，从而子协程获取写锁失败，这样整个程序两个写成处于死锁状态。  

## 参考

* [Lock re-accquision in sync forbiden](https://groups.google.com/forum/#!topic/golang-nuts/4sx5pPp8gFw)
* [Package sync](https://golang.org/pkg/sync/#RWMutex.Lock)
* [Readers–writer lock](https://en.wikipedia.org/wiki/Readers%E2%80%93writer_lock)
* [goroutine blocks when calling RWMutex RLock twice after an RWMutex Unlock](https://stackoverflow.com/questions/30547916/goroutine-blocks-when-calling-rwmutex-rlock-twice-after-an-rwmutex-unlock)
* [sync: document that double RLock isn't safe #15418](https://github.com/golang/go/issues/15418)
