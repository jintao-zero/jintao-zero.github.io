---
layout: post
title: "SSH常用设置(免密、别名登录、隧道转发)"
date: 2017-03-30 19:48:56 +0800
comments: true
categories: tool
tags: ssh
---
作为程序员每天都需要使用ssh登录到远程主机进行部署调试，本篇将总结ssh工具几个常用方法：  

* [免密登录](#password-less)    
* [别名登录](#alias)    
* [隧道转发](#tunnel)  

## <a name="password-less"> </a>免密登录
1. 在本地机器上生成pubkey
2. 将pubkey拷贝到远程主机~/.ssh/authorized_keys文件中

## <a name="alias"/> </a>别名登录

在免密登录的基础上， 可以方便实现别名登录
在本地~/.ssh/config文件中配置远程主机相关信息：  

	Host vps
 		HostName 45.78.61.64
		Port 22
 		User root
 本地可以使用

 	ssh vps
 这样的形式直接登录目标主机
 
## <a name="tunnel"> </a> 隧道转发
远端主机可能设置了防火墙，只开放少数端口，比如远端机器只开放了22 ssh端口，如果本地想连接远端3306端口的话，会被防火墙拒绝，这个时候我们可以利用ssh的隧道功能，将本地流量通过ssh tunnel转发远端目标端口，在~/.ssh/config配置文件中添加如下配置：  
	
	Host tunnel
   		HostName 45.78.61.64
    	Port 27764
    	LocalForward 3306 127.0.0.1:3306
    	User root
 上面的意思是将本地3306端口流量隧道转发到45.78.61.64 3306端口
	添加配置后，需要运行下面命令：  

	ssh -f -N tunnel 
使用lsof -i:3306我们可以看到ssh程序正在监听这个端口，当有数据从这个端口进来时，ssh程序会将数据转发到远端sshd服务端程序，服务端程序会将流量转发到远端本地3306端口，ssh隧道数据是加密的，我上一篇开发的[http proxy](http://jintao-zero.github.io/blog/2017/03/29/golang-shi-xian-httpdai-li/)我们可以利用ssh可以利用这个功能躲避GFW

