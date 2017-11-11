---
layout: post
title: "使用logroate管理日志文件"
date: 2017-11-11 11:15:06 +0800
comments: true
categories: ops
tags: ops 
---
logrotate命令设计用来管理日志文件。它可以自动循环，压缩，删除和邮件发送日志文件。可以根据不同的周期（日、周、月）和大小来管理文件。  
通常情况下，logrotate作为cron天任务每天执行一次。如果logrotate一天之中被运行了多次，除非设置size作为日志循环处理条件或者使用`-f`或者`--force`参数，否则日志文件不会被多次处理。    

## logrotate如何工作
系统以通常是每天cron任务运行logrotate。CentOS系统中`logrotate`定时任务脚本位于`/etc/cron.daily`中。  
如果以更高的频率运行logrotate任务，那么可以将`logrotate`放置在`/etc/cron.hourly`目录中。  
logrotate运行时会读取/etc/logrotate.conf文件中的配置，根据文件中配置的信息对日志文件进行处理。  
logrotate cron任务执行脚本如下：  
	
	#!/bin/sh

	/usr/sbin/logrotate -s /var/lib/logrotate/logrotate.status /etc/logrotate.conf
	EXITVALUE=$?
	if [ $EXITVALUE != 0 ]; then
    	/usr/bin/logger -t logrotate "ALERT exited abnormally with [$EXITVALUE]"
	fi
	exit 0
	
<!-- more -->

## logrotate.conf  
logrotate主要的配置文件是`/etc/logrotate.conf`。  
这个配置文件中包含了一些logrotate在执行日志处理时的默认配置。  
在这个文件中包含了其他的配置文件：  
	
	# RPM packages drop log rotation information into this directory
	include /etc/logrotate.d
我们可以将应用相关配置文件放置在`/etc/logrotate.d/`目录下  

## logrotate.d
`ls /etc/logrotate.d/`可以查看系统中已经配置的应用相关配置文件：  
	
	[root@test logrotate.d]# ls -al /etc/logrotate.d/
	total 40
	drwxr-xr-x.  2 root root  4096 Nov 11 09:57 .
	drwxr-xr-x. 89 root root 12288 Nov 10 14:15 ..
	-rw-r--r--   1 root root   243 Oct 31  2016 nginx
	-rw-r--r--.  1 root root   136 Jun 10  2014 ppp
	-rw-r--r--   1 root root   127 Aug  5  2016 redis
	-rw-r--r--   1 root root   224 Nov  5  2016 syslog
	-rw-r--r--   1 root root   100 Mar  3  2017 wpa_supplicant
	-rw-r--r--   1 root root   100 Nov 15  2016 yum
	[root@test logrotate.d]#
配置文件中配置了关于如何对应用产生的日志文件进行处理的配置信息：  
	
	[root@test logrotate.d]# cat nginx
		/var/log/nginx/*log {
    	create 0644 nginx nginx
    	daily
    	rotate 10
    	missingok
    	notifempty
    	compress
    	sharedscripts
    	postrotate
        	/bin/kill -USR1 `cat /run/nginx.pid 2>/dev/null` 2>/dev/null || true
	    endscript
	}
下面会对配置文件中的配置命令进行详细说明。  

## 配置命令  
可以使用通过查看man手册获取logrotate命令的相信命令参数说明：  

	man logrotate
/etc/logrotate.conf中配置了默认设置，应用配置文件会继承并覆盖/etc/logrotate.conf中的配置方法。  
	
### Log files
在上面nginx相关的配置信息中，开头的`/var/log/nginx/*log`即定义了需要处理的日志文件路径，大括号中的配置命令是对日志文件进行的具体配置。   
配置日志文件路径时，可以使用通配符。在一个文件中可以包含多个这样的配置信息。  

### Rotate Count
`rotate`命令配置了旧日志文件保留个数。当保留的日志文件超过这个数目时，最老的日志文件将会被删除。

	rotate 10
这个命令高速logrotate保留10个日志文件。  

### Rotation interval   
可以通过以下命令设置日志文件处理周期：  
	
	daily
	weekly
	monthly
	yearly

如果配置中没有指定时间周期，每次logrotate运行时都会日志文件进行处理，除非配置了size信息。  
如果想以不同的周期运行logrotate工具，可以设置不同的cron任务。比如，每小时执行一次logrotate工具，可以在`/etc/cron.hourly`中放置执行脚本，包含以下执行命令：  
	
	/usr/sbin/logrotate /etc/logrotate.hourly.conf

在`/etc/logrotate.hourly.conf`配置文件包含需要每小时进行处理的日志配置信息。  

### Size
可以使用`size`命令配置一个文件大小，logrotate工具根据这个文件大小来判断是否对日志文件进行处理。  
	
	size 100k
	size 100M
	size 100G
第一个命令配置日志文件大于100k字节时处理文件，第二个命令配置日志文件大于100M字节时处理日志文件，第三个命令配置日志文件大于100G时处理日志文件。 通常情况不建议将size设置太大。  

需要注意的是，`size`命令与`时间周期`配置是互斥的，而且`size`具有更高的优先级，当配置了`size`时，配置的`时间周期`将不起作用。  

### Compression
如果希望对产生的旧日志文件进行压缩处理，通常可以在`/etc/logrotate.conf`中配置`compress`，如果作为全局配置，那么所有产生的日志文件都会被压缩（以gzip格式）：  
	
	compress
	
对于具体应用来说，如果不想对日志文件进行压缩处理，那么可以使用`nocompress`进行配置：  
	
	nocompress

### Postrotate
logrotate每次处理配置文件配置的日志文件后将会运行`postrotate`脚本。通常使用`postrotate`脚本在处理日志文件后重启应用，这样应用可以使用一个新日志文件。  
	
	postrotate
    	/usr/sbin/apachectl restart > /dev/null
	endscript

`postrotate`命令表示脚本开始，`endscript`命令表示脚本结束。  
### sharedscripts
`sharescripts`告诉logrotate只运行一次`postrotate`脚本。  

下面的脚本示例：  
	
	/data/l1/ss/test/trend/log.txt {
    	rotate 5    		# 保存5个旧日志文件
    	copytruncate		# 拷贝并清除日志文件
    	#size 20M
    	maxsize 20M		# 日志文件最大大小为20M，达到20M时即被清理
    	start 0			# 日志文件从0编号
    	weekly				# 每周清理一次
    	#dateext
    	#dateyesterday
    	createolddir	   # 如果日志文件保存路径不存在，则创建
   		olddir /data/l1/ss/test/trend/log	   # 旧日志文件保存路径
	}

## 测试 logrotate
如果怀疑日志文件没有被正确处理，或者添加了一个新配置，可以使用一些参数对logrotate进行调试。  
### Verbose
`-v`参数，告诉logrotate输出详细信息。当logrotate没有按照设想处理日志时，使用`-v`参数进行调试。  

### Ddbug
`-d`参数，告诉logrotate去检查日志文件，但是不实际执行处理动作。在测试新配置文件时，这个参数很有作用。  
debug参数可以帮助检查配置文件格式是否正确和是否能够发现日志文件。

### Force
`-f`参数，强制logrotate工具处理所有日志文件，不管当时日志文件是否满足处理条件。
### Combining flags
测试命令可以组合使用。让logrotate强制执行并且输出所有详细信息，但是不实际执行，可以使用如下命令：  
	
	/usr/sbin/logrotate -vdf /etc/logrotate.conf

我们将会看到它会列出logrotate将会做什么，包括哪些日志文件将会处理和处理过程。    

如果想测试所有的配置信息，可以强制执行：  
	
	/usr/sbin/logrotate -vf /etc/logrotate.conf
日志将会强制处理，并打印详细信息。  

## 状态信息 
查看logrotate状态信息：  
	
	cat /var/lib/logrotate.status
	logrotate state -- version 2
	"/var/log/acpid.log" 2010-6-18
	"/var/log/iptables.log" 2010-6-18
	"/var/log/uucp.log" 2010-6-29
	..



## 参考 
* man logrotate
* [Understanding logrotate utility](https://support.rackspace.com/how-to/understanding-logrotate-utility/)
