---
layout: post
title: "tcp client 使用select进行I/O复用"
date: 2015-04-06 11:49:13 +0800
comments: true
categories: ['Network']
tags: Network
---
在[tcp客户端示例](http://jintao-zero.github.io/blog/2015/02/10/tcp-ke-hu-duan-shi-li/)中，存在以下两种情况时，示例代码不能更好的处理：  

* 目前客户端输入一条数据并发送，等待服务端返回后，才能继续输入数据
   这样的设计不利于客户端批量处理数据
   
* 当服务端出现异常（如服务进程退出或者服务端主机崩溃等异常情况），客户端也许正在等待输入，不能够及时感知到服务端的异常，只有从send返回错误时，才能感知到服务端故障

# select 函数介绍
上面实现的客户端示例中，客户端只能处理一个描述符，我们需要修改客户端使其可以同时处理标准输入和套接字描述符，达到I/O复用：  

	#include <sys/select.h>
	#include <sys/time.h>
	int select(int nfds, fd_set *restrict readfds, fd_set *restrict writefds, fd_set *restrict errorfds,
         struct timeval *restrict timeout);

* timeout参数，它告知内核等待所有描述符中的一个就绪可花多长时间  
	struct timeval {  
	    long tv_sec;  /\* seconds\*/  
	    long tv_usec; /\* microseconds\*/  
	}     
	(1) 永远等待下去：仅在有一个描述符准备好时返回。为此，将timeout设置为   空指针  
	(2) 等待一段固定时间：在有一个描述符准备好I/O时返回，但是不超过由该参数所指向的timeval结构中指定的秒数和微秒数  
	(3) 根本不等待：检查描述符后立即返回，这称为轮询。为此，该参数必须指向一个timeval结构，而且其中的定时器值必须为0

* readfds, writefds, errorfds参数，这三个参数指定了，需要内核测试读、写     和异常条件的描述符集合。
  系统提供了以下宏来对描述符进行操作：  
  void FD_CLR(fd, fd_set *fdset)  
  void FD_COPY(fd_set *fdest_orig, fd_set *fdset_copy)  
  int FD_ISSET(fd, fd_set *fdset)  
  void FD_SET(fd, fd_set *fdset)  
  void FD_ZERO(fd_set *fdset)   

* nfds参数，指定待测试的描述符个数，它的值是待测试的最大描述符加1

<!-- more -->

# 描述符就绪条件
* 满足下列四个条件中的任何一个时，一个套接字准备好读  
  a) 该套接字接收缓冲区中的数据字节数大于等于套接字接收缓冲区低水位标记的当前大小。对这样的套接字执行读操作不会阻塞并将返回一个大于0的值（也就是返回准备好读入的数据）。对于TCP和UDP套接字，其默认值为1。  
  b) 该连接的读半部关闭（也就是接收了FIN的TCP连接）。对这样的套接字的读操作将不阻塞并返回0（也就是返回EOF）  
  c) 该套接字是一个监听套接字且已完成的连接数大于0。对于这样的套接字调用accept通常不会阻塞。    
  d) 其上有一个套接字错误待处理。对这样的套接字的读操作将不阻塞并返回－1（也就是返回一个错误），同时把errno设置成确切的错误条件。  
 
* 下列四个条件中的任何一个满足时，一个套接字准备好写。  
  a) 该套接字发送缓冲区中的可用空间字节数大于等于套接字发送缓冲区低水位表姐的当前大小，并且或者该套接字已连接，或者该套接字不需要连接（如UDP套接字）。    
  b) 该连接的写半部关闭。对这样的套接字的写操作将产生SIGPIPE信号。  
  c) 使用非阻塞式connect的套接字已建立连接，或者connect已经以失败告终。  
  d) 其上有一个套接字错误待处理。
* 如果一个套接字存在带外数据或者仍处于带外标记，那么它有异常条件待处理。
  
# 修改tcp客户端程序
* 批量输入  
  a) 用户输入数据，标准输入可读，read返回读入字节数，发送到服务器  
  b) 用户输入Ctrl-D时，标准输入可读，read返回字节数为0，此时关闭套接字写端  
		
* 套接字增加处理情况：
  a) 如果对端TCP发送数据，那么该套接字变为可读，并且read返回一个大于0的值（即读入数据的字节数）。  
  b) 如果对端TCP发送一个FIN（对端进程终止），那么该套接字变为可读，并且read返回0（EOF）。  
  c) 如果对端TCP发送一个RST（对端主机崩溃并重新启动），那么该套接字变为可读，并且read返回－1，而errno中含有确切的错误码。  

以下是相关修改：  
1、将标准输入、套接字添加到客服描述符集合
	
	fd_set readfds;
	FD_ZERO(&readfds);
	FD_SET(sock_client, &readfds);
    FD_SET(STDIN_FILENO, &readfds);
    int maxfd = (STDIN_FILENO > sock_client ? STDIN_FILENO: sock_client) + 1;
    select(maxfd, &readfds, NULL, NULL, NULL)
    
2、标准输入可读

	if (FD_ISSET(STDIN_FILENO , &readfds)) {
		ssize_t rd_size = read(STDIN_FILENO , buf, sizeof(buf));
		if (rd_size > 0) {
			ssize_t sd_size = send(sock_client, buf, rd_size, 0);
			buf[rd_size] = '\0';
			printf("send msg(len:%ld):%s to server \n", sd_size, buf);
		} else if (rd_size == 0) {
		// client finish send msg to server, we close write part of the full-duplex connection
			shutdown(sock_client, SHUT_WR); // send FIN to server
			stdineof = 1;
			continue;
		} else {
		// client finish send msg to server, we close write part of the full-duplex connection
			printf("read fail from console, reason:%s \n", strerror(errno));
			break;
		}
	}  
	
3、套接字可读

	if (FD_ISSET(sock_client, &readfds)) {
		ssize_t rd_size = read(sock_client, buf, sizeof(buf));
		if (rd_size > 0) {
			printf("receive from server:%s", buf);
		} else if (rd_size == 0) { // client receive FIN
				printf("receive 0 from server\n");
				if (stdineof == 1) {
					printf("normal finish\n");
				} else {// server crash and reset, client receive RST
					printf("server terminate prematurely\n");
				}
				break;
			} else {
				printf("read fail.reason: %s\n", strerror(errno));
				break;
			}
		}
	}


下面是使用select进行I/O复用的tcp客户端代码：

	int main(int argc, char *argv[])
	{
		// judge the legal arguments
		if (2 > argc) {
			printf("please input a hostname \n");
			return -1;
		}

	// create a socket
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
			bzero(&servaddr, sizeof(servaddr));
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


			int stdineof = 0;
			for (; ; ) {
				fd_set readfds;
				FD_ZERO(&readfds);
				FD_SET(sock_client, &readfds);
				if (stdineof == 0)
					FD_SET(STDIN_FILENO, &readfds);
				int maxfd = (STDIN_FILENO > sock_client ? STDIN_FILENO: sock_client) + 1;
				if (select(maxfd, &readfds, NULL, NULL, NULL) > 0) {
					char buf[LINE_MAX]={0};
					if (FD_ISSET(STDIN_FILENO , &readfds)) {
						ssize_t rd_size = read(STDIN_FILENO , buf, sizeof(buf));
						if (rd_size > 0) {
							ssize_t sd_size = send(sock_client, buf, rd_size, 0);
							buf[rd_size] = '\0';
							printf("send msg(len:%ld):%s to server \n", sd_size, buf);
						} else if (rd_size == 0) {
							// client finish send msg to server, we close write part of the full-duplex connection
							shutdown(sock_client, SHUT_WR); // send FIN to server
							stdineof = 1;
							continue;
						} else {
							// client finish send msg to server, we close write part of the full-duplex connection
							printf("read fail from console, reason:%s \n", strerror(errno));
							break;
						}
					} else if (FD_ISSET(sock_client, &readfds)) {
						ssize_t rd_size = read(sock_client, buf, sizeof(buf));
						if (rd_size > 0) {
							printf("receive from server:%s", buf);
						} else if (rd_size == 0) { // client receive FIN
							printf("receive 0 from server\n");
							if (stdineof == 1) {
								printf("normal finish\n");
							} else {// server crash and reset, client receive RST
								printf("server terminate prematurely\n");
							}
							break;
						} else {
							printf("read fail.reason: %s\n", strerror(errno));
							break;
						}
					}
				}
			}
			close(sock_client);
		}
