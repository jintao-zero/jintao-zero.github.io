---
layout: post
title: "golang 工程代码结构"
date: 2017-06-11 12:19:45 +0800
comments: true
categories: program
tags: golang
---
golang工程常用代码目录结构

	github.com/user/project
		pkg
			p1
				*.go
			p2
				*.go 
		cmd
			cmdline
				main.go
		web
				main.go
		vendor
			github/*/*
		examples
		docs


##参考
* [Project structure for a tool with multiple UIs](https://stackoverflow.com/questions/32634837/project-structure-for-a-tool-with-multiple-uis/32635264#32635264)

* [How should I structure packages for a multiple-binary web application?](https://forum.golangbridge.org/t/how-should-i-structure-packages-for-a-multiple-binary-web-application/665)
* [kpass](https://github.com/seccom/kpass)
