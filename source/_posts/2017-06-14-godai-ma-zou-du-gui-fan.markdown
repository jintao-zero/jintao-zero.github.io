---
layout: post
title: "Go代码走读规范"
date: 2017-06-14 19:27:11 +0800
comments: true
categories: dev
tags: Golang
---
Go代码走读时注意点，非代码规范：  

* [Gofmt](#gofmt)
* [Comment Sentences](#Comment Sentences)

## <span id="Gofmt">Gofmt</span>
使用[gofmt](https://golang.org/cmd/gofmt/)工具修复大多数代码风格问题。机会所有Go代码都是用`gofmt`进行格式化。剩下的非机器可以格式化的代码风格将在本篇文章中进行说明。  
一个替代工具是`goimports`，它是`gofmt`工具的一个超集，它会自动增加或者删除引入包行。  

## <span id="Comment Sentences">Comment Sentences</span>
查看[https://golang.org/doc/effective_go.html#commentary](https://golang.org/doc/effective_go.html#commentary)，声明的注释应该占完整行，这样在将注释抽取到godoc文档时可以更好的格式化。注释应该以被注释声明开始，存在结束：  

	// Request represents a request to run a command.
	type Request struct { ...

	// Encode writes the JSON encoding of req to w.
	func Encode(w io.Writer, req *Request) { ...

## <span id="Contexts">Contexts</span>
`context.Context`类型实例在API和进程边界中传递安全证书、跟踪信息、超时期限和取消信号。Go程序在从RPC或者HTTP请求开始到输出请求的整个函数调用链过程中显示传递Context实例。    
大部分函数使用Context作为第一个参数：  

	func F(ctx context.Context, /* other arguments */) {}

如果一个函数不是请求特化的话，可以使用context.Background，即使你觉得不必要也宁可传递一个Context实例。默认情况是传递一个Context上下文，除非有更好的理由来使用`context.Background`。  
不要在结构体类型中添加`Context`成员，在结构体类型的所有需要使用Context的方法中添加一个ctx参数，用来传递Context。一个特殊情况是如果这些方法需要满足标准库或者第三方库中的接口。  
不要在函数声明中使用特定Context类型或者使用非Context接口。  
除非真的需要，否则通过参数，全局变量或者其他方法传递应用程序数据。  
Context是不可变的，所以可在多个调用中传递同一个ctx共享同样的最后期限、取消信号、资格认证和跟踪等等。  

## <span id="Copying"> Copying</span>
避免非预期的别名使用，当从其他包拷贝结构体时需要小心。比如，bytes.Buffer类型包含一个`[]byte`数组，当保存的字符串比较小时，内部`[]byte`数组就比较小。如果拷贝一个`Buffer`，新旧Buffer类型变量中的数组就发生了混淆，这样接下来对于两个Buffer对象的使用将产生不可预知的情况。  
总的来说，如果一个类型`T`，它的方法都与`*T`想对象，那就不要复制这个类型的变量。  

## <span id="Declaring Empty Slices"> Declaring Empty Slices</span>
声明一个空切片时，使用下面方式：  
	
	var t []string
而不是  
	
	t := []string{}

前者声明一个nil切片，后者定义一个长度为0的非nil切片。两种方式定义的切片`len`和`cap`的值都为0，建议使用前者。  

一些场景下建议使用后者定义空切片，比如将对象序列化为JSON格式时，`nil`序列化为`null`，`[]string{}`序列化为`[]`。  

设计接口时，避免nil切片，非nil切片和长度为0的切片产生差异，否则会产生微妙的错误。  

更多关于Go中nil的讨论见：[Understanding Nil](https://www.youtube.com/watch?v=ynoY2xz-F8s)  

## <span id="Crypto Rand">Crypto Rand</span>  
即使只使用一次，也不要使用`math/rand`生成key。没有种子，生成是可预测的。使用`time.Nanoseconds()`设置种子，只有一点熵。可替换方法是，使用`crypto/rand`，如果需要文本，打印为十六进制或者base64编码。  

	import (
    "crypto/rand"
    // "encoding/base64"
    // "encoding/hex"
    "fmt"
	)

	func Key() string {
    	buf := make([]byte, 16)
    	_, err := rand.Read(buf)
    if err != nil {
        panic(err)  // out of randomness, should never happen
    }
    return fmt.Sprintf("%x", buf)
    // or hex.EncodeToString(buf)
    // or base64.StdEncoding.EncodeToString(buf)	
	}

## <span id="Doc Comments">Doc Comments</span>
所有顶级，对外提供的名称应该有文档注释，一些重要的非对外提供的类型和函数定名也需要有注释。更多关于注释的信息参考：[https://golang.org/doc/effective_go.html#commentary](https://golang.org/doc/effective_go.html#commentary)


## <span id="Don't Panic">Don't Panic</span>
关于错误处理参考：[https://golang.org/doc/effective_go.html#errors](https://golang.org/doc/effective_go.html#errors)。不要使用`panic	`处理普通错误。使用error和多返回值。  

## <span id="Error Strings">Error Strings</span>
错误字符串不应该大写，除非是以名词或者缩写开头，因为错误信息通常跟在其他信息后面。使用`fmt.Errorf("something bad")`，而不是`fmt.Errorf("Something bad")`，所以`log.Printf("Reading %s: %v", filename, err)`格式化的信息中不会在中间出现一个可以的大写字符。

## <span id="Examples">Examples</span>
当添加一个新包时，包含使用示例：一个可运行示例，或者一个简单测试演示一个完整调用顺序。  
更多信息参考：[testable Example() functions](https://blog.golang.org/examples)。  

## <span id="Goroutine Lifetimes">Goroutine Lifetimes</span>
当创建goroutine时，需要对它是否退出和什么时间退出非常清楚。    

goroutine阻塞在发送或者接收通道时可能会导致泄漏：垃圾收集不会结束这样的goroutine。  

即使这些goroutine没有泄漏，这样的goroutine有可能导致微妙的不易定位的问题。向已经关闭的通道发送数据会导致panic。

保持并发执行的代码足够简单，这样goroutine的声明周期更明显。 如果不能这样的话，对goroutine的退出时机进行注释。  

## <span id="Handle Errors">Handle Errors</span>
查看错误处理[https://golang.org/doc/effective_go.html#errors](https://golang.org/doc/effective_go.html#errors)，不要使用`_`丢弃错误。如果一个函数返回错误，对返回值进行检查。处理错误，返回。如果是异常情况，则panic。  

## <span id="Imports">Import</span>  


