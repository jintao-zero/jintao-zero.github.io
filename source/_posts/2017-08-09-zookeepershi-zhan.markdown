---
layout: post
title: "zookeeper实战"
date: 2017-08-09 15:57:12 +0800
comments: true
categories: ops
tags: zookeeper 
---
zookeeper实战记录

* [环境搭建](#setup)
* [命令行操作](#cmd)
* [编程](#code)

##<span id="setup">环境搭建</span>
搭建适合生产环境使用的zookeeper集群，最好是大于等于3台机器  

1. [下载](http://zookeeper.apache.org/releases.html)

2. 解压
	
	`tar xzf zookeeper***.tar.gz`

3. 修改配置文件  
	进入解压目录conf子目录  
	`cp zoo_sample.cfg zoo.cfg`  
	修改配置文件   
	设置相关参数：  
	dataDir为zookeeper持久化数据存放目录     
	dataDir=/data/zookeeper
	配置集群中三台机器
	server.1=10.174.176.156:2888:3888
	server.2=10.168.106.149:2888:3888
	server.3=10.168.37.52:2888:3888  
	需要在每台机器dataDir目录下建立myid文件，并将上面server后面对应的id写入myid文件中 
4. 启动程序  
	bin/zkServer.sh start  
	
5. 查看日志文件  
	tail -f zookeeper.out
	
<!-- more -->

##<span id="cmd">命令行操作</span>
`bin/zkCli.sh -server 127.0.0.1:2181`  
连接zookeeper服务端。  
查看`help`帮助命令：  

```
[zk: 127.0.0.1:2181(CONNECTED) 0] help  
ZooKeeper -server host:port cmd args
	stat path [watch]
	set path data [version]
	ls path [watch]
	delquota [-n|-b] path
	ls2 path [watch]
	setAcl path acl
	setquota -n|-b val path
	history
	redo cmdno
	printwatches on|off
	delete path [version]
	sync path
	listquota path
	rmr path
	get path [watch]
	create [-s] [-e] path data acl
	addauth scheme auth
	quit
	getAcl path
	close
	connect host:port
[zk: 127.0.0.1:2181(CONNECTED) 1]
```
新建节点：  

	[zk: 127.0.0.1:2181(CONNECTED) 2] create /test 	test_data
	Created /test
	[zk: 127.0.0.1:2181(CONNECTED) 3]
查看节点： 
 
```	
[zk: 127.0.0.1:2181(CONNECTED) 4] ls /test
[]
[zk: 127.0.0.1:2181(CONNECTED) 5] get /test
test_data
cZxid = 0x10000002a
ctime = Wed Aug 09 16:26:43 CST 2017
mZxid = 0x10000002a
mtime = Wed Aug 09 16:26:43 CST 2017
pZxid = 0x10000002a
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 9
numChildren = 0
[zk: 127.0.0.1:2181(CONNECTED) 6]
```

