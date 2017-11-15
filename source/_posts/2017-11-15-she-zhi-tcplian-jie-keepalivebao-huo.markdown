---
layout: post
title: "设置TCP连接keepalive保活"
date: 2017-11-15 15:46:35 +0800
comments: true
categories: network  
tags: tcp/ip
---
一个tcp连接包含两端。如何确保连接两端的socket都在线呢，一个方法是应用程序定期发送一个心跳信息给对方，tcp/ip提供了一个系统层面的保活机制，设置了socket keepalive属性后，系统会定期给对端发送心跳报文，根据是否能收到回复来判断对端是否在线，当对端不在线时，对本段socket的读操作将会读到EOF。  
## keepalive 
三个属性决定keepalive如何工作，在Linux系统中，这三个属性位于`proc`文件系统中：  

* `tcp_keepalive_time`  
  连接空闲时间，超时后发送保活探测包，默认为`7200`秒
* `tcp_keepalive_intvl`  
  每天探测包之间的间隔时间，默认为：`75`秒 	
* `tcp_keepalive_probes`
  总的保护包数，超过则认为对端不在线，默认为`9`个  

keepalvie的工作过程如下：    

1. 客户端建立到服务端的连接
2. 当连接的空闲时间超过`tcp_keepalive_time`时，客户端向服务端发送一个tcp`ACK`包  
3. 服务端是否对刚刚的`ACK`包进行回复？
   * NO    
   		1. 等待`tcp_keepalive_intvl`超时，发送`ACK`包
   		2. 如果探测包个数达到`tcp_keepalive_probes`，则发送一个`RST`报文并关闭连接   		 
   * YES  
   		收到心跳包回复，则继续步骤2  

大多数系统中，默认是开启tcp连接保活的，当对端(7200+74*9)秒都没有回复的话，则会关闭本端连接。  

<!-- more -->

## 修改具体连接保活属性  
系统提供了默认值，在具体应用程序中，我们可以单独对连接设置不同的属性值：  
C/C++代码示例如下：  
	
	int keepalive_enabled = 1;
	int keepalive_idle = 10; // 10 second
	int keepalive_intvl = 5; // send keepalive every 5 second
	int keepalive_cnt = 3; //
	int ret = setsockopt(conn_sock, SOL_SOCKET, SO_KEEPALIVE, &keepalive_enabled, sizeof(int));
 	ret = setsockopt(conn_sock, IPPROTO_TCP, TCP_KEEPIDLE, &keepalive_idle, sizeof(int));
	ret = setsockopt(conn_sock, IPPROTO_TCP, TCP_KEEPINTVL, &keepalive_intvl, sizeof(int));     
	ret = setsockopt(conn_sock, IPPROTO_TCP, TCP_KEEPCNT, &keepalive_cnt, sizeof(int));


## 修改系统全局保活属性  
* 修改proc文件系统中的属性值
		
		echo 10 > /proc/sys/net/ipv4/tcp_keepalive_time
		echo 5  > /proc/sys/net/ipv4/tcp_keepalive_intvl
		echo 3  > /proc/sys/net/ipv4/tcp_keepalive_probes  
	这种方法修改的属性值，系统重启后属性值会恢复	
* sysctl命令修改  
		
		/sbin/sysctl -w net.ipv4.tcp_keepalive_time=10
		/sbin/sysctl -w net.ipv4.tcp_keepalive_intvl=5
		/sbin/sysctl -w net.ipv4.tcp_keepalive_probes=3
	这种方法，系统重启后属性值会恢复	
* 修改/etc/sysctl.conf配置文件

		net.ipv4.tcp_keepalive_time=10
		net.ipv4.tcp_keepalive_intvl=5
   		net.ipv4.tcp_keepalive_probes=3   
 
 ##参考
* [Custom Configuration of TCP Socket Keep-Alive Timeouts](http://coryklein.com/tcp/2015/11/25/custom-configuration-of-tcp-socket-keep-alive-timeouts.html)  
* [TCP Keepalive HOWTO](http://tldp.org/HOWTO/TCP-Keepalive-HOWTO/usingkeepalive.html)
