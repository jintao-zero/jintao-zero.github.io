---
layout: post
title: "Golang 通过http服务获取程序性能统计数据"
date: 2017-01-08 22:55:07 +0800
comments: true
categories: program
tags: Golang
---
Golang标准库`net/http/pprof`通过HTTP服务对外提供性能统计数据，以便`pprof`这样的可视化工具使用。  

引入此`pprof`库向http服务注册路径处理函数。此库注册的http服务路径都是以`/debug/pprof`开头。  
##使用方法
###方式引入库   
	
	import _ "net/http/pprof"

###开启HTTP服务
如果程序中已经运行了一个或者多个HTTP服务，那么就无需再开启HTTP服务，否则需要像如下代码片段，开启一个HTTP服务：  

	go func() {
		log.Println(http.ListenAndServe("localhost:6060", 	nil))
	}()

其中地址也可以不绑定到localhost,可以选择监听所有端口,比如:  
	
	go func() {
		log.Println(http.ListenAndServe(":6060", 	nil))
	}()

###获取性能数据 
1、查看堆栈性能数据  
	
	go tool pprof http://localhost:6060/debug/pprof/heap
	
2、查看30s CPU性能数据  
	
	go tool pprof http://localhost:6060/debug/pprof/profile

3、查看goroutine阻塞性能数据 
	
	go tool pprof http://localhost:6060/debug/pprof/block
	
4、查看5s执行栈  
	
	wget http://localhost:6060/debug/pprof/trace?seconds=5

5、查看所有可用性能数据
	
	浏览器打开  http://localhost:6060/debug/pprof/

参考：  
[Golang官博](https://golang.org/pkg/net/http/pprof/)
