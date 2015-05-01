---
layout: post
title: "tcp server使用kevent进行io复用"
date: 2015-05-01 08:40:25 +0800
comments: true
categories: ['网络']
---
# 概述
kqueue是FreeBSD系统中引入的可扩展事件通知接口，NetBSD，OpenBSD，DragonflyBSD和OS X这些系统也支持此接口。传统的io复用接口如select和poll存在以下两个缺点：  

*  支持的文件描述符数量有限，select接口支持的文件描述个数与进程能够打开的最大文件个数有关
*  当大量描述符需要复用时，select和poll的效率会降低，内核是采用轮询方式判断描述符是否可用

kqueue是针对传统传统的select/poll处理大量的文件描述符性能较低效而开发出来的。注册一批描述符到kqueue以后，当其中的描述符状态发生变化时，kqueue将一次性通知应用程序哪些描述符可读、可写或者发生错误。

kqueue支持多种类型的文件描述符，包括socket、信号、定时器、AIO、VNODE、PIPE。

#接口

##kqueue

	int kqueue(void);
	
kqueue系统调用创建一个新的内核事件队列并返回该队列描述符。调用fork时，子进程不继承该描述符。  
失败时，系统调用返回-1,同时设置errno

##kevent
	
	int kevent(int kq, const struct kevent *changelist, int nchanges, struct kevent *eventlist, int nevents, const struct timespec *timeout);
使用kevent系统调用向队列注册事件和返回已经触发的事件。  

* kq  
  kq为kqueue系统调用返回的队列描述符
* changelist  
  changelist指向一个kevnet结构体数组，kevent定义在<sys/event.h>头文件中，这个数组描述了需要对kq队列中的事件进行修改，包括添加、删除、修改等操作
* nchanges  
  nchanges是changelist参数指向数组的大小
* evnetlist  
  指向一个kevnet结构体数组，用于返回已经触发的事件
* nevents  
  定义eventlist数组大小
* timeout
  timeout是非NULL指针时，定义系统调用超时间隔。timeout为NULL时，kevent系统调用无限阻塞等待事件发生。timeout非NULL，但是值为0时，kevent立即返回
  
  <!-- more -->

##EV_SET
	
	EV_SET(&kev, ident, filter, flags, fflags, data, udata);
	struct kevent {
             uintptr_t       ident;          /* identifier for this event */
             int16_t         filter;         /* filter for event */
             uint16_t        flags;          /* general flags */
             uint32_t        fflags;         /* filter-specific flags */
             intptr_t        data;           /* filter-specific data */
             void            *udata;         /* opaque user data identifier */
     };
EV_SET宏用来初始化一个kevent结构体。下面详细介绍结构体各个字段

* ident
  ident用来标识一个事件，ident值的准确含义由filter字段进行解释，ident值通常代表一个文件描述符
* filter
  内核filter定义的过滤器处理发生在ident上的事件。预定义的系统过滤器如下，通过kevent结构体中的flags和data字段传递参数  
  
  EVFILT_READ   
  文件描述符作为标识符，当描述符可读时返回。根据文件描述符类型不同，触发该过滤器的情况不同：
  
  * sockets:  
  处于监听状态的socket套接字，当存在待处理的建立连接请求（已经完成三次握手）时，该套接字可读，kevent结构体的data字段包含监听队列中待处理的请求个数。  
  非监听状态的套接字缓冲区中有可读数据时，触发该过滤器，每个套接字对应的缓冲区可读低水位可以通过传递fflags和data参数进行设置。
  * Vnodes  
  当文件指针没有达到文件末尾时，触发该过滤器。
  * Fifos, Pipes  
  当管道可读时，触发该过滤器。data字段返回可读字节数。
  
  EVFILT_WRITE    
  文件描述符作为标识符。当文件描述符可写时，触发该过滤器。对于sockets、pipes和fifos，data字段返回缓冲区中可写字节数。    
  EVFILT_AIO  
  尚不支持  
  EVFILT_VNODE    
  文件描述符作为标识符，fflags字段定义需要监测的事件类型：  
  * NOTE_DELETE    
  unlink系统调用操作文件描述符指向的文件时，事件发生。
  * NOTE_WRITE    
  对文件描述指向的文件进行写操作
  * NOTE_EXTEND  
  对文件描述符指向的文件进行extened
  * NOTE_ATTRIB  
  文件描述符指向的文件属性发生了变化
  * NOTE_LINK  
  文件描述指向的文件链接数发生了变化
  * NOTE_RENAME  
  文件描述符指向的文件发生了重命名
  * NOTE_REVOKE  
  
  kevent系统调用返回时，fflags包含触发过滤器的事件
  
  EVFILT_PROC
  进程id作为标识符。监测的事件定义在fflags字段中：  
  * NOTE_EXIT    
  进程退出
  * NOTE_EXITSTATUS  
  进程已经退出，退出状态  
  * NOTE_FORK  
  进程调用fork或者其他类似接口创建子进程
  * NOTE_EXEC  
  进程调用execve或者其他类似接口执行一个新进程
  * NOTE_SIGNAL  
  进程被发送一个信号。  
  kevent系统调用返回时，fflags包含触发过滤器的事件。
  
  EVFILT_SIGNAL  
  将信号值作为标识符，当该信号发送到进程时，触发过滤器。   
   
  EVFILT_TIMER
  设置一个ident标识的定时器。当添加一个定时器时，data字段指定定时器超时事件，fflags字段可以为以下设置：  
  * NOTE_SECONDS   
  data字段值单位为秒
  * NOTE_USECONDS  
  data字段值单位为毫米
  * NOTE_NSECONDS  
  data字段值单位为纳秒  
  * NOTE_ABSOLUTE
  data字段值为绝对时间

* flags  
 flags表示对该事件做的操作。
 	* EV_ADD  
 	 添加该事件到kqueue队列
 	* EV_ENABLE
 	 如果该事件触发，kevent返回该事件
 	* EV_DISABLE
 	 如果该事件触发，kevent不返回该事件
 	* EV_DELETE
 	 从kevent队列中删除该事件
 	* EV_RECEIPT
 	 对kqueue进行批量修改，而不返回任何待处理事件。
 	* EV_ONESHOT
 	 过滤器第一次触发后，返回该事件，然后将该事件从kqueue队列中删除。
 	* EV_CLEAR
 	 用户获取事件后，重置事件状态。
 	* EV_ERROR
 	 系统处理事件出错时，返回该事件event，将flags设置为EV_ERROR  
 	 
* fflags  
  过滤器相关的标志
* data  
  过滤器相关的数值  
* udata
  用户定义的值，当事件返回时，内核将该值带给用户

将[tcp server 使用select进行并发](http://jintao-zero.github.io/blog/2015/04/12/tcp-servershi-yong-selectjin-xing-i-slash-ofu-yong/)示例改为使用kevent进行i/o多路复用，主要涉及以下方面的修改：  

1、套接字注册  

	int Register(int kq, int fd)
	{
     	struct kevent ke;
     	EV_SET(&ke, fd, EVFILT_READ, EV_ADD, 0, 0, NULL);
     	return kevent(kq, &ke, 1, NULL, 0, NULL);
 	}
 目前只是对监听套接字和客户端连接套接字注册可读过滤器
 
2、循环调用kevent处理套接字事件  

	struct kevent events[MAX_EVENT_COUNT];
	for ( ; ; ) {
		int nevent = kevent(kq, NULL, 0, events, MAX_EVENT_COUNT, NULL);
	}
	
	void HandleEvent(int kq, struct kevent * events, int nevents) 
	{
		for (int i = 0; i < nevents; i++) {
        	int sock = events[i].ident;
        	int data = events[i].data;
         	if (sock == serv_sock) {
            	Accept(kq, data);
          	} else {
            	Receive(sock, data);
          	}
      	}
  	}
	
	void Accept(int kq, int conn_size)
  	{
    	for (int i = 0; i < conn_size; i++) {
          int client = accept(serv_sock, NULL, NULL);
          Register(kq, client);
      	}
  	}
 
  	void Receive(int sock, int buf_size)
  	{
    	if (0 == buf_size)
        	close(sock); // close a file descriptor removing any 			kevents that reference the descriptor
 
      	char *buf = malloc(buf_size);
      	if (NULL == buf)
        	return;
 
      	recv(sock, buf, buf_size, 0);
      	for (int i = 0; i < buf_size; i++)
      		buf[i] = toupper(buf[i]);
      	send(sock, buf, buf_size, 0);
 		free(buf);
  	}	 
  	
#结束语
这篇文章只是简单介绍了kevent I/O多路复用模型的简单用法，上述例子中涉及的一些细节，如错误处理等都尚须完善  
参考[使用 kqueue 在 FreeBSD 上开发高性能应用服务器](http://www.ibm.com/developerworks/cn/aix/library/1105_huangrg_kqueue/index.html#resources)

  
  
  
  
  	

  
  

