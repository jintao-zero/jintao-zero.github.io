---
layout: post
title: "Golang Reflect介绍"
date: 2017-03-06 17:44:02 +0800
comments: true
categories: program
tags: Golang
---
Golang [reflect](https://golang.org/pkg/reflect/)包实现了运行时反射，允许程序管理任意类型对象。典型应用是使用`TypeOf`从静态类型`interface{}`中抽取动态类型信息。  
使用`ValueOf`返回一个`Value`类型变量代表运行时数据。  
[The Laws of Reflection](https://golang.org/doc/articles/laws_of_reflection.html)介绍了Golang reflect机智中的几条规则：  
##从interface反射到对象
反射基本功能是用来检查interface变量中保存的`类型`和`数据`对。[package reflect](http://golang.org/pkg/reflect/)中提供了两种类型：[Type](http://golang.org/pkg/reflect/#Type)和[Value](http://golang.org/pkg/reflect/#Value)。这两种类型提供了反问interface内部数据的机制。调用`reflect.TypeOf`和`reflect.ValueOf`函数分别返回`reflect.Type`和`reflect.Value`变量：  

	ackage main

	import (
    "fmt"
    "reflect"
	)

	func main() {
    	var x float64 = 3.4
    	fmt.Println("type:", reflect.TypeOf(x))
	}
程序输出如下：  
	
	type: float64

`reflect.TypeOf`函数定义如下：  

	// TypeOf returns the reflection Type of the value in the interface{}.
	func TypeOf(i interface{}) Type

当我们调用`TypeOf(x)`时，x首先存在一个空interface中，之后作为参数，reflect.TypeOf解析参数获取实际类型。    
调用`Value(x)`时，从接口参数中获取实参值。  

	var x float64 = 3.4
	fmt.Println("value:", reflect.ValueOf(x))
打印结果如下：  
	
	value: <float64 Value>
	
##从反射对象逆转回接口值
给定一个`reflect.Value`对象，调用`Interface`方法可以恢复一个接口值。实际上，`Interface`方法将Value对象的值和类型信息打包到一个接口值中并返回：  
	
	// Interface returns v's value as an interface{}.
	func (v Value) Interface() interface{}

接上面的例子：   

	y := v.Interface().(float64) // y will have type float64.
	fmt.Println(y)
将反射对象v代表的float64值打印。  
##如果修改一个反射对象，value必须可设置  
`reflect.ValueOf`返回的Value值，并不是所有情况下都是可修改的。下面示例：  

	var x float64 = 3.4
	v := reflect.ValueOf(x)
	v.SetFloat(7.1) // Error: will panic.
执行代码时，报如下panic错误：  
	
	panic: reflect.Value.SetFloat using unaddressable value
反射对象v不是可设置的。可设置是`reflect.Value`类型对象的一个属性，不是所有`Value`对象有此属性。  
Value对象`CanSet`方法返回Value对象是否可以设置，在我们的例子中：  
	
	var x float64 = 3.4
	v := reflect.ValueOf(x)
	fmt.Println("settability of v:", v.CanSet())
将会输出：  
	
	settability of v: false

`Settability`是反射对象的一个属性，它反映的是是否能够修改创建这个反射对象的原对象。`Settability`由反射对象是否持有原对象决定。  
	
	var x float64 = 3.4
	v := reflect.ValueOf(x)
当我们执行上面代码时，根据x生成一个interface对象作为是实参调用`reflect.Value`，interface对象保存的是x值拷贝。  
	
	v.SetFloat(7.1)
上面代码修改的并不是x值。  
对于函数调用，如果想修改实参，那必须传递实参地址到函数中。  

	var x float64 = 3.4
	p := reflect.ValueOf(&x) // Note: take the address of x.
	fmt.Println("type of p:", p.Type())
	fmt.Println("settability of p:", p.CanSet())
上面代码，我们创建了一个x地址的反射对象，代码的输出如下：  
	
	type of p: *float64
	settability of p: false

反射对象p是不可设置的，我们不是要修改p，而是要修改p指向的x对象。  调用`Elem`方法可以获取p所指向的原对象：  

	v := p.Elem()
	fmt.Println("settability of v:", v.CanSet())
现在v对象就是可以修改的了：  
	
	settability of v: true
现在v对象代表了原对象x，现在可以对v对象进行设置：  
	
	v.SetFloat(7.1)
	fmt.Println(v.Interface())
	fmt.Println(x)
结果输出如下：  
	7.1
	7.1
###Structs
当我们拥有struct结构体对象地址时，可以对结构体对象进行修改  
接下来是一个小的示例，使用结构体对象地址创建一个反射对象，然后使用reflect包中提供的结构体相关的方法，对结构体对象进行遍历和修改。  

	type T struct {
   	 	A int
    	B string
	}
	t := T{23, "skidoo"}
	s := reflect.ValueOf(&t).Elem()
	typeOfT := s.Type()
	for i := 0; i < s.NumField(); i++ {
    	f := s.Field(i)
    	fmt.Printf("%d: %s %s = %v\n", i,
        typeOfT.Field(i).Name, f.Type(), f.Interface())
	}
上面代码的输出如下：  
	
	0: A int = 23
	1: B string = skidoo
因为s是可设置的反射对象，我们可以修改原对象的字段值：  
	
	s.Field(0).SetInt(77)
	s.Field(1).SetString("Sunset Strip")
	fmt.Println("t is now", t)

结果如下：  

	t is now {77 Sunset Strip}

##案例

参考：  

1、https://golang.org/pkg/reflect/     
2、https://blog.golang.org/laws-of-reflection



	




