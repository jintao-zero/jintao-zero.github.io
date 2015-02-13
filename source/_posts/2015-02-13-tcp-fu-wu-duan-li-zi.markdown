---
layout: post
title: "tcp 服务端例子"
date: 2015-02-13 11:04:43 +0800
comments: true
categories: ['tcp/ip']
---
实现了一个简单服务端，使用tcp与客户端进行通讯。本片文章主要介绍在tcp服务端网络编程中用到的几个基本函数：  
1、创建套接字socket函数

	int socket(int domain, int type, int protocol)

socket函数创建一个通讯端并返回这个终端的文件描述符。  

参数domain指定发生整个通讯过程所在的协议域。协议域定义在<sys/socket.h>头文件中，常用协议域名为：
	
	PF_LOCAL 主机内部通讯协议，之前名为PF_UNIX
	PF_UNIX  主机内部通讯协议，废弃，使用PF_LOCAL
	PF_INET  IPV4通讯协议
	PF_INET6 IPV6通讯协议
	。。。
本例子中使用PF_INET ipv4通讯协议域  

参数type指定socket类型，目前定义类型为：  
	
	SOCK_STREAM									     
	SOCK_DGRAM
	SOCK_RAW
	SOCK_SEQPACKET
	SOCK_RDM
SOCK_STREAM 类型提供序列化、可靠的、全双工字节流套接字
SOCK_DGRAM 类型提供无连接的、不可靠的，有报文最大限制的数据报通讯套接字
SOCK_RAW 类型提供读取内部网络协议和网卡接口的原始套接字，需要有超级用户权限  
本例子中使用SOCK_STREAM流套接字类型  

失败时，函数返回-1，成功时，返回套接字文件描述符

2、bind函数

	int bind(int socket, const struct sockaddr *address, socklen_t address_len);

bind函数把一个本地协议地址赋予一个套接字。
参数socket，为需要赋予地址的套接字。
参数address，为赋予到套接字的地址结构，有ip和端口号标识。
参数address_len, 为该地址结构大小

对于服务端来讲，在调用监听函数之前，都需要bind到特定地址和端口  
<!-- more -->

3、listen函数

	int listen(int socket, int backlog);
  
listen函数仅由TCP服务器调用，它做两件事情：  
(1) 当socket函数创建一个套接字时，它被假设为一个主动套接字，也就是说，它是一个将调用connect发起连接的客户套接字。	listen函数把一个未连接的套接字转换成一个被动套接字，指示内核应接受指向该套接字的连接请求。根据tcp状态转换图，调用listen导致套接字从CLOSED状态转换到LISTEN状态。  
(2) 本函数的第二个参数规定了内核应该为相应套接字排队的最大连接个数。
本函数通常应该在调用socket和bind这两个函数之后，并在调用accept函数之前调用。
内核为任何一个给定的监听套接字维护两个队列：  
(1)未完成连接队列，每个这样的SYN分节对应其中一项：已由某个客户发出并到达服务器，而服务器正在等待完成相应的TCP三路握手过程。这些套接字处于SYN_RCVD状态  
(2)已完成连接队列，每个已完成TCP三路握手的客户对应其中一项。这些套接字处于ESTABLISHED状态。
参数backlog规定了以上两个队列的总和大小

4、accept函数

	int accept(int socket, struct sockaddr *restrict address, socklen_t *restrict address_len);
	
accept函数由TCP服务器调用，用于从已完成连接队列对头返回下一个已完成连接。如果已完成连接队列未空，那么进程被投入睡眠（假定套接字为默认的阻塞方式）。
参数address和address_len返回已连接的对端进程的协议地址。
如果accept成功，那么其返回值是由内核自动生产的一个全新描述符，代表与所返回客户的TCP连接。

5、下面是服务端源代码
此服务端小程序主要完成功能：1、监听服务端口，接收连接 2、接收客户端数据并处理后发回服务端

	#include <stdio.h>
	#include <unistd.h>
	#include <sys/socket.h>
	#include <sys/errno.h>
	#include <string.h>
	#include <netinet/in.h>
	#include <arpa/inet.h>
	#include <ctype.h>

	#define SERVER_PORT 8000

	int main()
	{
		// create socket
		int sock_server = socket(AF_INET, SOCK_STREAM, 0);
		if (-1 == sock_server) {
			printf("create socket fail.reason:%s\n", 				strerror(errno));
			return -1;
		}

		// bind socket to local addr and port
		struct sockaddr_in servaddr;
		servaddr.sin_family = AF_INET;
		servaddr.sin_addr.s_addr  = INADDR_ANY;
		servaddr.sin_port = htons(SERVER_PORT);
		if (-1 == bind(sock_server, (struct sockaddr *)&servaddr, (socklen_t)sizeof(servaddr))) {
			printf("bind to local addr fail.%s \n", strerror(errno));
			return -1;
		}
		printf("bind to local addr success\n");

		// make socket to listen
		if (-1 == listen(sock_server, 128)) {
			printf("listen fail.%s\n", strerror(errno));
			return -1;
		}

		struct sockaddr_in clientaddr;
		socklen_t clientaddr_len = sizeof(clientaddr);
		int connsock;

		// accept a new client connection
		while ((connsock = accept(sock_server, (struct sockaddr *)&clientaddr, &clientaddr_len)) != -1) {
			
			printf("new client connected, %s:%d \n", inet_ntoa(clientaddr.sin_addr), 			clientaddr.sin_port);

			// recv data from client
			char buf[1024];
			ssize_t rcv_len;
			while ((rcv_len = recv(connsock, buf, sizeof(buf), 0)) > 0) {
				printf("recv from client:%s\n", buf);
				for (int i = 0; i < rcv_len; i++)
			  		buf[i] = toupper(buf[i]);
				send(connsock, buf, rcv_len, 0);
				printf("send to  client:%s\n", buf);
			}
			printf("len:%ld \n", rcv_len);
			close(connsock);
		}
		close(sock_server);
	}	
	
运行效果：
{% img /images/server.png %}	