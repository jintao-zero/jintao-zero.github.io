---
layout: post
title: "tcp 客户端示例"
date: 2015-02-10 19:07:27 +0800
comments: true
categories: ['Network']
tags: Network
---
实现了一个简单客户端，使用tcp与服务端进行通讯。本片文章主要介绍在tcp客户端网络编程中用到的几个基本函数：  
1、创建socket
  
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

失败时，函数返回－1，成功时，返回套接字文件描述符  

2、connect连接服务端  

	int connect(int socket, const struct sockaddr *address, socklen_t address_len);
connect函数在一个socket上面初始化一个连接

参数socket为要建立连接的套接字  
参数address为要对端连接的地址，不同协议域会有不同的方式解析这个参数内容
参数address_len为传入的address地址所对应地址结构体占据的内存大小

连接成功，函数返回0，失败，则返回－1，同时置错误码errno

<!-- more --> 

3、send发送内容

	ssize_t send(int socket, const void *buffer, size_t length, int flags);

send函数发送数据到对端套接字。使用send函数时需要socket处于连接状态。  

参数buffer为需要发送内容首地址  
参数length为需要发送的报文内容长度  
参数flags可能包含以下两种：  
		
		#define MSG_OOB        0x1  /* process out-of-band data */
    	#define MSG_DONTROUTE  0x4  /* bypass routing, use direct interface */
本例中，不需要以上两种标志，设置为0即可

发送成功，函数返回发送的字节数，发送失败，返回－1，设置errno错误码

4、recv接受内容  

	ssize_t recv(int socket, void *buffer, size_t length, int flags);
	
recv函数只可以在处于连接状态套接字使用。
默认情况下，如果没有数据可读，recv调用将会一直阻塞等待数据到达，除非socket被设置为非阻塞状态。通常情况下，recv会返回任何可读数据，最大达到请求的length数量，而不是一直等待接收到length数量才返回，可以通过设置套接字属性SO_RCVLOWAT和SO_RCVTIMEO来进行修改。
如果没有可接收数据并且对端已经执行关闭操作，那么recv返回0

参数flags，有以下几种类型：  
	
	MSG_OOB        process out-of-band data
    MSG_PEEK       peek at incoming message
    MSG_WAITALL    wait for full request or error
MSG_OOB标志请求接收带外数据。  
MSG_PEEK标志使recv只从头读取队列中的数据而不从队列中删除这些数据，这样接下来的读操作会读取相同内容。
MSG_WAITALL标志使recv操作阻塞直到读取的报文达到请求字节数。

recv函数返回接收到的字节数，返回－1表示发生错误，返回0表示对端已经关闭连接。

5、示例代码
示例完成功能：1、与服务端建立tcp连接 2、从控制台读取输入，发送到服务端 3、读取服务端返回数据 4、关闭tcp连接	

	#include <sys/socket.h>
	#include <netdb.h>
	#include <arpa/inet.h>
	#include <sys/errno.h>
	#include <stdio.h>
	#include <string.h>
	#include <unistd.h>

	#define SERVER_PORT 8000

	int main(int argc, char *argv[])
	{
		// judge the legal arguments
		if (2 > argc) {
			printf("please input a hostname \n");
			return -1;
		}

		// create a SOC_STREAM socket
		int sock_client = socket(AF_INET, SOCK_STREAM, 0);
		if (0 > sock_client) {
			perror("create socket error");
			return -1;
		}


		// construct server sockaddr
		struct hostent *p = gethostbyname(argv[1]);
		if (p == NULL) {
			printf("gethostbyname:%s fail.reason:%s \n", argv[1], strerror(errno));
			return -1;
		}
		struct sockaddr_in servaddr;
		servaddr.sin_family = AF_INET;
		servaddr.sin_addr = *(struct in_addr *)p->h_addr;
		servaddr.sin_port = htons(SERVER_PORT);

		// connect to server
		if (connect(sock_client, (struct sockaddr *)&servaddr, sizeof(servaddr)) == -1) {
			printf("connect to server:%s port:%d fail.reason:%s \n", inet_ntoa(servaddr.sin_addr), SERVER_PORT,
			strerror(errno));
			return -1;
		}
		printf("connect to server:%s port:%d success\n", inet_ntoa(servaddr.sin_addr), SERVER_PORT );

		// send and rev data with server
		char buf[1024];
		while(fgets(buf, sizeof(buf)-1,	stdin))
		{
			int len = strlen(buf);
			if (len == 1)
		  		break;
			buf[len-1] = '\0';
			printf("%d\n", len-1);
			ssize_t send_size = send(sock_client, buf, len, 0);
			if (send_size != len) {
				printf("send fail \n");
				break;
			}
			ssize_t rcv_size = recv(sock_client, buf, sizeof(buf), 0);
			if (rcv_size == -1) {
				printf("rcv fail\n");
				break;
			}
			printf("recv from server:%s \n", buf);
		}
		close(sock_client);
	}	

