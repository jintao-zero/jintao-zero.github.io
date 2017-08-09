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
* [Contexts](#Contexts)
* [Copying](#Copying)
* [Declaring Empty Slices](#Declaring Empty Slices)
* [Crypto Rand](#Crypto Rand)
* [Doc Comments](#Doc Comments)
* [Don't Panic](#Don't Panic)
* [Error Strings](#Error Strings)
* [Examples](#Examples)
* [Goroutine Lifetimes](#Goroutine Lifetimes)
* [Handle Errors](#Handle Errors)
* [Import](#Import)
* [Import Dot](#Import Dot)
* [In-Band Errors](#In-Band Errors)
* [Indent Error Flow](#Indent Error Flow)
* [Initialisms](#Initialisms)
* [Interfaces](#Interfaces)
* [Mixed Caps](#Mixed Caps)
* [Named Result Parameters](#Named Result Parameters)
* [Package Comments](#Package Comments)
* [Package Names](#Package Names)
* [Pass Values](#Pass Values)
* [Receiver Names](#Receiver Names)
* [Receiver Type](#Receiver Type)
* [Synchronous Functions](#Synchronous Functions)

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
除了避免命名冲突，不要重命名引入的包。好的包命名不需要重命名。万一出现冲突，优先考虑重命名本地包或者项目相关的引入。  

引入的包成组组织，以空行分隔。标准库总是在第一组。  

```
	package main

	import (
		"fmt"
		"hash/adler32"
		"os"

		"appengine/foo"
		"appengine/user"

		"code.google.com/p/x/y"
		"github.com/foo/bar"
	)
```
[goimports](https://godoc.org/golang.org/x/tools/cmd/goimports)工具会完成上面的格式化。  

## <span id="Import Dot">Import Dot</span>
进行测试时，因为循环依赖，不能将测试用例放入被测包中时，可以用`import .`解决：  

```
package foo_test

import (
	"bar/testutil" // also imports "foo"
	. "foo"
)
```
这个例子中，测试文件不能放入`foo`包中，因为它使用`bar/testutil`，`bar/testutil`包引入了`foo`包。 我们使用`import .`使测试文件假装在包`foo`中，实际上不在。除了这一个特殊情况，不要使用`import .`。它使程序无法更好阅读，因为不方便区分一个大写开头的方法是当前包还是引入包中的顶级标识符。  

## <span id="In-Band Errors">In-Band Errors</span>
C或者其他类似语音中，函数返回-1或者null来表示错误是很常见的：  

```
// Lookup returns the value for key or "" if there is no mapping for key.
func Lookup(key string) string

// Failing to check a for an in-band error value can lead to bugs:
Parse(Lookup(key))  // returns "parse failure for value" instead of "no value for key"
```
Go中的多返回值机制提供了一个更好的方案。函数直接放回一个错误值来表示函数执行结果是否正确。
	
```
// Lookup returns the value for key or ok=false if there is no mapping for key.
func Lookup(key string) (value string, ok bool)

```
这种格式可以阻止不正确的使用函数返回值：  

```
// Lookup returns the value for key or ok=false if there is no mapping for key.
func Lookup(key string) (value string, ok bool)
```
一个更健壮和可阅读的风格如下：  

```
value, ok := Lookup(key)  
if !ok  {  
    return fmt.Errorf("no value for %q", key)
}
return Parse(value)
```
这个规则对于导出函数和非导出函数都是有用的。  

## <span id="Indent Error Flow">Indent Error Flow</span>
保持正常流程最小缩进，缩进异常处理流程。这有助于提高可读性，可以快速查阅程序正常流程，不建议如下风格：  

```
if err != nil {
	// error handling
} else {
	// normal code
}
```
建议如下风格：  

```
if err != nil {
	// error handling
	return // or continue, etc.
}
// normal code
```
像下面这样`if`表达式中带初始化操作的语句：   

```
if x, err := f(); err != nil {
	// error handling
	return
} else {
	// use x
}
```
建议将声明挪到外面去，如下所示：

```  
x, err := f()
if err != nil {
	// error handling
	return
}
// use x
```
## <span id="Initialisms"> Initialisms</span>
命名中的首字母缩写应该保持同一大小写风格。比如，`URL`应该写为`URL`或者`url`（`URLPony`或者`urlPony`）,不应该为`Url`。又一个例子：应该为`ServeHTTP`而不应该为`ServeHttp`。  
这个规则也适用于`identifier`的缩写`ID`，应该命名为`appID`而不是`appid`。  
protocol buffer编译生成的代码可以不遵守这个规则。人类编写的程序应该比机器坚持更高标准。  


##<span id="Interfaces"> Interfaces</span>
Go 接口通常定义在使用接口的包中，而不是在实现接口的包中。实现的包应该返回具体类型（通常指针类型或者结构体）：这种方法，可以方便的在实现类型上增加新方法。  

不要因为模拟，在具体实现这边定义接口类型。  

不要在使用之前定义接口类型：没有使用的实际用例，对于判断一个接口是否需要是困难的，不要去想接口应该包含哪些方法。  

```
package consumer  // consumer.go

type Thinger interface { Thing() bool }

func Foo(t Thinger) string { … }
```

```
package consumer // consumer_test.go

type fakeThinger struct{ … }
func (t fakeThinger) Thing() bool { … }
…
if Foo(fakeThinger{…}) == "x" { … }
```

```
// DO NOT DO IT!!!
package producer

type Thinger interface { Thing() bool }

type defaultThinger struct{ … }
func (t defaultThinger) Thing() bool { … }

func NewThinger() Thinger { return defaultThinger{ … } }
```
定义一个具体类型，让消费者模拟生产者实现。

```
package producer

type Thinger struct{ … }
func (t Thinger) Thing() bool { … }

func NewThinger() Thinger { return Thinger{ … } }

```

##<span id="Mixed Caps"> Mixed Caps</span>
查看[mixed-caps](https://golang.org/doc/effective_go.html#mixed-caps)，这打破了其他语言中惯例。比如一个非导出常量命名`maxLength`而不是`MaxLength`和`MAX_LENGTH`。

##<span id="Named Result Parameters"> Named Result Parameters</span>

下面这样的定义：  

```
func (n *Node) Parent1() (node *Node)
func (n *Node) Parent2() (node *Node, err error)
```
最好定义成如下方式：  

```
func (n *Node) Parent1() *Node
func (n *Node) Parent2() (*Node, error)
```
其他情况，如果一个函数返回两个或者三个相同类型的值，或者一个返回值的意义不是很明确，那么应该给返回值命名。

	func (f *Foo) Location() (float64, float64, error)
没有下面命名更清晰：  
	
	// Location returns f's latitude and longitude.
	// Negative values mean south and west, respectively.
	func (f *Foo) Location() (lat, long float64, err error)

##<span id="Package Comments"> Package Comments</span>
包注释，与其他被godoc工具展示的注释类似，必须紧邻着package语句，不需要空白行。
  
```
// Package math provides basic constants and mathematical functions.  
package math
```

```
/*
Package template implements data-driven templates for generating textual
output such as HTML.
....
*/
package template
```

对于`package main`可以使用一些其他风格的注释，比如，在`seedgen`目录中的`package main`包，可以采用如下注释：  
	
	// Binary seedgen ...
	package main
或者
	
	// Command seedgen ...
	package main

或者 
	
	// Program seedgen ...
	package main
或者
	
	// The seedgen command ...
	package main

或者
	// The seedgen program ...
	package main

或者
	
	// Seedgen ..
	package main

上面例子的其他形式也是可以接受的。  
对于包注释，不应该以小写字母开头，因为这些注释会被公开查看，应该按照英语规则书写。
查看[https://golang.org/doc/effective_go.html#commentary](https://golang.org/doc/effective_go.html#commentary)关于注释的详细说明。  

##<span id="Package Names"> Package Names</span>
外界需要使用报名来引用包中的命名，所以可以在命名中将包名去掉。比如，如果`package chubby`中定义`File`，不需要定义成`ChubbyFile`，因为访问时就是这样`chubby.ChubbyFile`。直接定义成`File`，这样访问时就会是`chubby.File`。避免无意义包名，如`util, common,misc,api,types,interfaces`。查看[http://golang.org/doc/effective_go.html#package-names](http://golang.org/doc/effective_go.html#package-names)和[http://blog.golang.org/package-names](http://blog.golang.org/package-names)详细信息。  

##<span id="Pass Values"> Pass Values</span>
不要只是为了节省几个字节而传递指针作为函数地址。如果一个函数都是以`*x`的形式使用参数`x`，那么就参数`x`就不应该是指针形式。比如以传递一个字符串的指针`*string`或者一个指向接口的类型`*io.Reader`作为函数参数，在这两种场景中，字符串和接口类型值都是固定大小，可以直接传递。这个建议对于大的结构体类型并不适用。  

##<span id="Receiver Names"> Receiver Names</span>
方法接受者名称应该反映它的实体类型。通常是类型的一个或者两个字母缩写（比如用c和cl代表Client）。不要使用`me`，`this`或者`self`这样的面向对象语言中使用的标识符。

##<span id="Receiver Type"> Receiver Type</span>
对于Go新手来说，定义方法时使用值还是指针作为接收者是困难的。如果有所疑虑，使用指针。下面是一些指导意见：  

* 如果接受者是`map``func`或者`chan`时，不要使用指针。如果接受者是一个`slice`并且方法中并没有重新分配slice，不要使用指针。  
* 如果方法需要修改接受者，必须使用指针。  
* 如果接受者是一个结构体并且包含sync.Mutex或者类似同步元语字段，接受者必须是指针类型避免拷贝。  
* 如果接受者是一个大的结构体或者数组，指针类型接受者更有效率。

##<span id="Synchronous Functions"> Synchronous Functions</span>

##参考

* [CodeReviewComments](https://github.com/golang/go/wiki/CodeReviewComments#receiver-names)
