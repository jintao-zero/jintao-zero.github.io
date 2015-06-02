---
layout: post
title: "Linux/Unix设置SUID"
date: 2015-06-02 13:56:27 +0800
comments: true
categories: ['Unix相关']
tags: [Unix]
---
#非root用户运行tcpdump
有些时候，我们希望以非root用户运行一些程序，比如以普通用户运行tcpdump命令进行抓包时，系统报如下异常信息：

Mac OS 系统：  

	MacBook-Pro:plugins jintao$ tcpdump
	tcpdump: ioctl(SIOCIFCREATE): Operation not permitted
	MacBook-Pro:plugins jintao$

Ubuntu系统： 
 
	[app@mp5 ~]$ tcpdump
	tcpdump: no suitable device found
	[app@mp5 ~]$

为什么会出现上述异常信息呢？是否是因为普通用户没有执行权限？下面是系统中tcpdump命令的权限信息  
Mac OS 系统：
	
	MacBook-Pro:plugins jintao$ which tcpdump
	/usr/sbin/tcpdump
	MacBook-Pro:plugins jintao$ ls -l /usr/sbin/tcpdump
	-rwxr-xr-x 1 root wheel 749040  9 10  2014 /usr/sbin/tcpdump*
	MacBook-Pro:plugins jintao$
	
Ubuntu系统：
	
	[app@mp5 ~]$ which tcpdump
	/usr/sbin/tcpdump
	[app@mp5 ~]$ ls -l /usr/sbin/tcpdump
	-rwxr-xr-x. 1 root root 742080 3月  26 2012 /usr/sbin/	tcpdump
	[app@mp5 ~]$
	
根据tcpdump的权限位-rwxr-xr-x我们发现对于普通用户jintao、app是拥有可读和可执行权限的。以普通用户执行tcpdump时，tcpdump进程的实际运行用户和有效用户都是普通用户，tcpdump运行过程中需要获取系统网卡资源信息，而获取这些信息需要root权限，这样的话系统就禁止了tcpdump程序的运行。  
解决这样的问题，我们可以尝试以下三种方法： 

  <!-- more -->
 

* su切换到root用户，然后再执行tcpdump，这一解决方案与我们以非root用户运行不符
* 以root用户编辑/etc/sudoers文件，赋予普通用户执行tcpdump命令的权限
* 设置tcpdump程序的用户ID权限位SUID    

```
	MacBook-Pro:C++ jintao$ sudo chmod u+s /usr/sbin/tcpdump
	Password:
	MacBook-Pro:C++ jintao$ ls -l /usr/sbin/tcpdump
	-rwsr-xr-x 1 root wheel 749040  9 10  2014 /usr/sbin/tcpdump*
	MacBook-Pro:C++ jintao$ tcpdump
	tcpdump: data link type PKTAP
	tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
```
下面着重介绍第三种方法：
#什么是SUID

SUID(Set owner User ID)是赋予文件的一种特殊权限位。在UNIX系统中，特权（例如能改变当前日期的表示法以及访问控制（例如，能否读、写一特定文件））是基于用户和组ID的。用户运行某程序时，程序访问资源的权限基于运行此程序的用户来决定。SUID被用来赋予程序访问资源的权限与程序的拥有者相同，而不是实际执行者。一言以蔽之，运行设置SUID权限位的程序时，普通用户将会获取文件拥有者的用户ID(UID)和组ID(GID)。  
##Example1:passwd command
我们运行passwd来修改我们的登陆密码，passwd命令拥有者是root用户。passwd命令会尝试修改系统配置文件，比如/etc/passwd, /etc/shadow等。其中的一些文件，只有root权限才能访问，非root用户无法打开。如果去掉SUID权限位，只是设置passwd command文件的所有权限位，那这个命令无法访问/etc/shadow等文件，程序会返回权限禁止错误或者其他措施。所以passwd被设置SUID权限位来赋予非root用户更新/etc/shadow等文件的权限。  

##Example2: ping command
当我们运行ping命令时，ping命令会打开socket文件和端口用来发送和接收IP数据包。非root用户无法获取权限打开socket和端口。所以对ping命令文件设置SUID权限位，任何执行ping命令的用户都可以获取ping命令文件拥有者（root用户）的权限。当执行ping命令时就拥有root权限来打开socket和端口port  

#如何设置文件SUID权限位
与设置文件其他的读、写、可执行权限位类似，可以用以下方法设置  
	
	* 字符命令方式（s代表Set）
	* 八进制数字（4）
	
对file1.sh设置SUID  

字符模式：
	
	chmod u+s file1.sh

设置owner可执行权限位为SUID位

数字模式：  
	
	chmod 4750 file1.sh
	
4750，4代表设置SUID权限位，7代表owner可读、可写、可执行权限位，5代表组拥有可读、可执行权限位，0代表其他不会不具有任何权限。

对于file1.sh文件，设置SUID前权限位：

	MacBook-Pro:C++ jintao$ ls -l file1.sh
	-rwxr--r-- 1 jintao staff 0  6  2 15:57 file1.sh*
	MacBook-Pro:C++ jintao$
	
设置SUID权限位后：
	
	MacBook-Pro:C++ jintao$ ls -l file1.sh
	-rwsr--r-- 1 jintao staff 0  6  2 15:57 file1.sh*
	MacBook-Pro:C++ jintao$

#何时使用SUID
1）当需要root用户登录来执行某些命令/程序/脚本  
2）不想赋予某些特殊用户信任，但是想以owner用户运行某些命令时  
3）不想使用SUDO命令而执行某些文件、脚本时  

#参考
http://www.linuxnix.com/2011/12/suid-set-suid-linuxunix.html




