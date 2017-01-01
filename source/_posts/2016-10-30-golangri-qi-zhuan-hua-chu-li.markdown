---
layout: post
title: "Golang日期转化处理"
date: 2016-10-30 22:13:12 +0800
comments: true
categories: program
tags: golang
---

golang中time包提供了时间操作函数。  
##获取时间
 	
 	type Time struct {
        // contains filtered or unexported fields
	}
 	
 	func Date(year int, month Month, day, hour, min, sec, nsec int, loc *Location) Time
    func Now() Time

一个Time结构体代表一个纳米精度的时间实例。now函数可以获取当前时间：  
	
	now := time.Now()
    fmt.Println(now)
   
结果如下：

	2016-11-27 22:46:17.666418725 +0800 CST	
<!-- more -->

##格式化时间

	func (t Time) Format(layout string) string

Format方法可以按照layout定义的格式格式化时间字符串。  
Golang中以2006 01 02 03 04 05 分别定义年、月、日 时、分、秒字段，03的24表示法为15。  
比如将当前时间格式化为YYYY-mm-dd hh:MM:ss，则layout为`2006-01-02 15:04:05`
	
	now := time.Now()
    fmt.Println(now.Format("2006-01-02 15:04:05"))
 输出为：

 	2016-11-27 22:56:40

控制精度的方法为在秒后面加0, 000、000000、000000000分别代表毫秒、微秒、纳秒：
	
	now := time.Now()
    fmt.Println(now.Format("2006-01-02 15:04:05"))
    fmt.Println(now.Format("2006-01-02 15:04:05.000"))
    fmt.Println(now.Format("2006-01-02 15:04:05.000000"))
    fmt.Println(now.Format("2006-01-02 15:04:05.000000000"))
输出：

	now := time.Now()
    fmt.Println(now.Format("2006-01-02 15:04:05"))
    fmt.Println(now.Format("2006-01-02 15:04:05.000"))
    fmt.Println(now.Format("2006-01-02 15:04:05.000000"))
    fmt.Println(now.Format("2006-01-02 15:04:05.000000000"))

##解析时间

    func Parse(layout, value string) (Time, error)
    func ParseInLocation(layout, value string, loc *Location) (Time, error)

Parse函数将格式化字符串解析为一个时间实例。`layout`定义了时间字符串的表现格式，比如：

	Mon Jan 2 15:04:05 -0700 MST 2006
当待解析时间字符串中没有时区时，`Parse`默认按照`UTC`时区解析时间。`ParseInLocation`按照loc参数指定的的location解析时间。

##定时器
定时器通过`NewTimer`或者`AfterFunc`创建，用AfterFunc创建的定时器超时后，在定时器goroutine中调用func函数，其他定时器则通过C通道发送一个时间对象。  
  
	type Timer struct {
        C <-chan Time
        // contains filtered or unexported fields
	}

	func AfterFunc(d Duration, f func()) *Timer  
	func NewTimer(d Duration) *Timer
	
定时器在使用过程中需要注意`Reset`和`Stop`方法的使用：  

	func (t *Timer) Stop() bool
Stop方法用来停止定时器触发。停止定时器则返回true，如果定时器已经超时或者已经被停止则返回false。Stop方法不关闭定时器中的channel。  
为了防止调用Stop方法后，定时器仍然触发，需要检查方法返回值并且读取定时器时间channel，代码片段：  
	
	if !t.Stop() {
		<-t.C
	}

	func (t *Timer) Reset(d Duration) bool
**需要特别注意的是，不要并发执行上面的代码。**    
	
Reset修改定时器的超时时间为d。修改成功则返回true，如果定时器已经超时或者被停止则返回false。     

	func (t *Timer) Reset(d Duration) bool
为了重复一个已经激活的定时器，需要先调用定时器`Stop`方法，如果定时器已经超时，则例如一下代码片段，先读取channel中的时间：  
	
	if !t.Stop() {
		<-t.C
	}
	t.Reset(d)
**需要特别注意的是，不要并发执行上面的代码。**    
在读取通道和定时器超时之间存在竞态条件，所以几乎不可能正确使用Reset方法的返回值。Reset方法应该总是与Stop方法一起使用。   


