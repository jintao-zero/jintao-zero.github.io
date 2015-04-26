---
layout: post
title: "tcp server使用select进行I/O复用"
date: 2015-04-12 15:30:03 +0800
comments: true
categories: ['网络']
---
在[tcp服务端示例](http://jintao-zero.github.io/blog/2015/02/13/tcp-fu-wu-duan-li-zi/)中，接受一个客户端连接，调用accept函数返回后就对本客户端连接进行业务处理，如果业务逻辑比较复杂，那么服务端有可能就阻塞在某个客户端上，而不能处理其他客户端连接，解决此问题有多种模型可以采用：  
1）多进程，为每个客户端连接创建一个子进程进行处理  
2）多线程，多个线程并行处理客户端连接  
3）I/O多路复用，服务器采用单线程模型，使用I/O多路复用模型，并行处理多个客户端文件描述符  
在上一篇示例[tcp client 使用select进行I/O复用](http://jintao-zero.github.io/blog/2015/04/06/tcp-client-shi-yong-selectjin-xing-i-slash-ofu-yong/)中介绍的select函数同样适用与对tcp server端的I/O多路复用改造，改造主要设计几下几个方面：  

<!-- more -->

1）将server监听套接字添加到readfds套接字集合中，监听套接字可读时，调用accept接受一个新的客户端连接，并将添加到readfds套接字集合中  

	if (FD_ISSET(sock_server, &rfds)) {
		struct sockaddr_in clientaddr;
		socklen_t clientaddr_len = sizeof(clientaddr);
		int clientsock = accept(sock_server, (struct sockaddr *)&clientaddr, &clientaddr_len);
		if (clientsock < 0) {
			printf("accept fail. reason:%s\n", strerror(errno));
			continue;
		} else {
			printf("new client connected, %s:%d \n", inet_ntoa(clientaddr.sin_addr), clientaddr.sin_port);
			int i;
			for (i = 0; i < FD_SETSIZE; i++) {
				if (clientfds[i] == -1) {
					clientfds[i] = clientsock;
					break;
				}
			}
			if (i == FD_SETSIZE) {
				printf("too many clients, ignore this one\n");
				continue;
			}
			FD_SET(clientfds[i], &allfds);
			if (clientfds[i] > maxfd)
				maxfd = clientfds[i];
		}
	}  
	
2）当每个客户端套接字可读时，对套接字调用read操作后，继续执行select判断是否有套接字可读，而不是处理一个客户端连接直到完成。

	for (int i = 0; i < FD_SETSIZE; i++) {
		if (readyfds == 0)
			break;
		if (clientfds[i] < 0)
			continue;
		if (FD_ISSET(clientfds[i], &rfds)) {
			readyfds--;
			char buf[LINE_MAX];
			ssize_t rcv_len;
			rcv_len = recv(clientfds[i], buf, sizeof(buf), 0);
			if (rcv_len > 0) {
				buf[rcv_len] = '\0';
				printf("recv from client(%d), msg: %s\n", clientfds[i], buf);
				for (int i = 0; i < rcv_len; i++)
					buf[i] = toupper(buf[i]);
					send(clientfds[i], buf, rcv_len, 0);
					printf("send to client:%s", buf);
				} else if (rcv_len == 0) {
					printf("recv_len == 0 from client:%d\n", clientfds[i]);
					FD_CLR(clientfds[i], &allfds);
					close(clientfds[i]);
					clientfds[i] = -1;
				} else {
					printf("recv from clientfd(%d) fail. reason:%s\n", clientfds[i], strerror(errno));
					continue;
				}
			}
		}
	}

