---
layout: post
title: "crontab任务设置及路径、动态库、产生多个进程问题解决"
date: 2015-11-21 11:17:04 +0800
comments: true
categories: DevOps 
tags: DevOps 
---
crontab文件中包含了cron守护进程需要执行的指令：某个时间执行这条命令。每个用户拥有自己的crontab，crontab中的命令以crontab所属用户运行。
	
	crontab [ -u user ] file
    crontab [ -u user ] [ -i ] { -e | -l | -r }

命令第一种形式是用file中的命令设置crontab任务  
-l 参数是打印当前crontab任务到标准输出  
-e 参数是使用VISUAL或者EDITOR环境变量定义的编辑器编辑当前crontab定时任务  
-r 参数删除当前定时任务  
-i 与用户进行交互

## 任务设置
crontab文件中可以包含三种语句：  
1、注释 
以#开头的语句都是注释

	#this is a comment in crontab 
	
2、设置环境变量  
	
	LD_LIBRARY_PATH = /homt/lib/

3、设置定时任务  
每一行定义一条定时任务，包括五个时间、日期字段，时间日期后面的内容都是命令  
时间日期包括分、时、天、月和一周某天字段：
	
	ield          allowed values
    -----          --------------
    minute         0-59
    hour           0-23
    day of month   1-31
    month          1-12 (or names, see below)
    day of week    0-7 (0 or 7 is Sun, or use names)

字段可以是＊，代表范围内所有  
字段也可以是一个范围，比如hour字段8-10，代表8，9，10  
字段也可以是一个列表，比如hour字段8，9，10  
字段也可以是一个范围和列表的混合，比如hour字段1-7，9－12  
字段也可以为一个间隔，比如hour 1-8/2,代表1,3,5,7  

<!-- more -->

##路径问题
crontab任务是根据所属用户从/etc/passwd文件中获取bashshell工具类型、logname和当前路径的。  
代码中如果使用的是相对路径，那么在会导致程序找不到文件，可以在程序入口处利用chdir类似功能函数修改程序的当前工作路径：  

	int chdir(const char *path);
	
##动态库问题
如果程序使用了动态库，而动态库放置的位置没有在等标准路径下的话，程序会因为加载不到动态库而执行失败，可能报如下的错误：
	  
	/root/server/demo_linux/server: error while loading shared libraries: libTDFAPI30.so: cannot open shared object file: No such file or directory
	
解决办法有两个：  
1、链接程序时指定绝对路径：  

	-Wl,-rpath=/root/server/lib
2、在crontab中设置LD_LIBRARY_PATH环境变量	
	
	LD_LIBRARY_PATH=/root/server/lib
	
## 创建两个相关进程

	root     19400  0.0  0.0 113116  1208 ?        Ss   18:10   0:00 /bin/sh -c /root/test/cron 1> stdout 2> stderr
	root     19402  0.0  0.0  12620  1072 ?        S    18:10   0:00 /root/test/cron
	
看到两个进程是因为crontab使用/bin/sh来启动定时任务程序:

	[root@iZ235ne7v0iZ demo_linux]# ps -o ppid -p 19402
 	PPID
	19400
	
可以将crontab任务修改为下列情况：

	10 18  * * * exec /root/test/cron 1> stdout 2> stderr
或者

	10 18  * * * bash -c '/root/test/cron 1> stdout 2> stderr'
