---
layout: post
title: "Golang Defer Panic 和Recover介绍"
date: 2017-03-07 11:08:41 +0800
comments: true
categories: program
tags: Golang
---
Golang语言提供了正常的流程控制：if、for、switch和goto。还提供了go表达式创建goroutine协程。接下来将要介绍的是：`defer`、`panic`和`recovery`  
##defer
defer表达式将一个函数放到一个list中。包裹defer函数的函数return时，这个defer函数将会被执行。`Defer`通常被用来简化资源清理动作。    
下面的代码片段，完成的功能是打开两个文件，将一个文件的内容拷贝到另外一个文件中：  

	func CopyFile(dstName, srcName string) (written int64, err error) {
        src, err := os.Open(srcName)
    	if err != nil {
        	return
    	}

 	    dst, err := os.Create(dstName)
   	 	if err != nil {
   	        return
    	}

 	    written, err = io.Copy(dst, src)
        dst.Close()
    	src.Close()
    	return
	}
这个片段有bug。当调用os.Createa失败时，函数返回，但是没有将已经打开的源文件关闭。在第二个return语句之前执行src.Close可以修复bug。但是如果存在多个资源需要关闭或者多个return语句时，这样就比较繁琐。使用defer表达式，可以方便确保资源关闭：  
   
    func CopyFile(dstName, srcName string) (written int64, err error) {
        src, err := os.Open(srcName)
        if err != nil {
            return
        }
        defer src.Close()

        dst, err := os.Create(dstName)
        if err != nil {
            return
        }
        defer dst.Close()

        return io.Copy(dst, src)
    }

Defer表达式会提醒我们在打开文件之后关闭资源，并且确保无论多少return语句都会将资源关闭。  
defer表达式的行为是简单和可预测的。下面有三条简单规则：    

1. 注册defer函数时，计算deferred函数的参数
下面的例子中，defer表达式注册fmt.Println函数时确定了入参为0，下面函数执行return语句时，fmt.Println将会打印出0

    	func a() {
        	i := 0
        	defer fmt.Println(i)
        	i++
        	return
    	}
2. 注册函数按照先进后出的的顺序进行执行   
    
        func b() {
            for i := 0; i < 4; i++ {
                defer fmt.Print(i)
             }
        }

	打印结果是“3210”

3. Deferred函数可以读写包裹函数的命名返回值
下面的例子中，包裹函数返回后，deferred函数修改了命名返回值i。因此，包裹函数的返回值为2 
	
		func c() (i int) {
        	defer func() { i++ }()
        	return 1
   		}

	这个特点可以用来修改函数返回的error返回值

<!-- more -->

##Panic
`panic`是Golang内建函数，它会停止当前流程，开始`panicking`。当函数F调用panic时，F将会停止执行正常流程，执行F函数中注册的defer函数，然后F函数返回。对于F函数的调用函数，F函数表现为panic。F函数的调用函数按照F函数panic时的行为一样，执行deferred函数后返回，这样沿着沿着调用栈，直到最上层，之后程序崩溃。`Panic`可以由程序调用`panic`函数后触发，也会有运行时错误触发，比如slice越界访问。  

##Recover
`recover`内建函数，可以用来从处于`panic`状态的goroutine中重新获取控制。`recover`只有在defer函数中才有用。在正常状态下，调用recover函数返回nil，不会有其他作用。如果当前goroutine处于`panicking`状态，那么调用`recover`函数，可以获取`panic`函数的参数，并终止`panicking`状态，进入正常状态。
下面的例子演示如何利用`defer`、`panic`和`recover`：  

	package main

	import "fmt"

	func main() {
    	f()
    	fmt.Println("Returned normally from f.")
	}

	func f() {
   		defer func() {
   			if r := recover(); r != nil {
      			fmt.Println("Recovered in f", r)
    		}
    	}()
    	fmt.Println("Calling g.")
    	g(0)
    	fmt.Println("Returned normally from g.")
	}

	func g(i int) {
   		if i > 3 {
       	fmt.Println("Panicking!")
        	panic(fmt.Sprintf("%v", i))
    	}
   		defer fmt.Println("Defer in g", i)
   		fmt.Println("Printing in g", i)
   		g(i + 1)
	}

程序输出如下： 

	Calling g.
	Printing in g 0
	Printing in g 1
	Printing in g 2
	Printing in g 3
	Panicking!
	Defer in g 3
	Defer in g 2
	Defer in g 1
	Defer in g 0
	Recovered in f 4
	Returned normally from f.
	
如果我们删除F函数中的defer函数，那么panic将不会得到恢复，一直到到goroutine调用栈的最顶层，终止程序执行。这样程序的输出如下：  

	Calling g.
	Printing in g 0
	Printing in g 1
	Printing in g 2
	Printing in g 3
	Panicking!
	Defer in g 3
	Defer in g 2
	Defer in g 1
	Defer in g 0
	panic: 4
 
	panic PC=0x2a9cd8
	[stack trace omitted]
	
##实践
使用`defer`和`recover`构造一个带recover的goroutine包裹函数

	func defaultPanicHandler(e interface{}) {
		fmt.Println(e)
		debug.PrintStack()
		// log here
	}

	var PanicHandler func(interface{}) = defaultPanicHandler

	func withRecover(fn func()) {
		defer func() {
			handler := PanicHandler
			if handler != nil {
				if err := recover(); err != nil {
					handler(err)
				}
			}
		}()
		fn()
	}

	func main () {
		go withRecover(
			func () {
				fmt.Println("aaaa")
				panic("panicking")
			}
		)
		for {
			time.Sleep(3 * time.Second)
		}
	}




参考：  

1. https://blog.golang.org/defer-panic-and-recover
