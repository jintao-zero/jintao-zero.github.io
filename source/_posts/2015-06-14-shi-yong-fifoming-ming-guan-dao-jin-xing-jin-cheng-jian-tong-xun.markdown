---
layout: post
title: "使用FIFO命名管道进行进程间通讯"
date: 2015-06-14 15:56:00 +0800
comments: true
categories: Unix
tags: Unix
---
管道是进程间通讯的一种方式。使用`pipe`，`popen`，`pclose`，等接口我们可以使用管道来进行进程间通讯。但是这一种管道只能由相关进程使用，这些相关进程的共同祖先进程创建了管道。通过FIFO，也称为命名管道，不相关的进程也可以进行通讯。
#创建命名管道
调用mkfifo接口创建命名管道：  

	#include <sys/types.h>
	#include <sys/stat.h>
	int mkfifo(const char *path, mode_t mode);
	
创建FIFO类似于创建文件。FIFO的路径名存在于文件系统中。FIFO是一种文件类型。stat结构成员st_mode的编码指明文件是否是FIFO类型。可以用S_ISFIFO宏对此进行测试。 mkfifo函数创建一个路径名为path的fifo。mode和umask掩码规定了命名管道的访问权限。  
参数：  
path是管道在文件系统中的存在路径。  
mode于open函数中定义的mode一样。  

返回值：  
成功返回0。失败返回-1，同时设置errno错误码。  

错误原因：  
	
* ENOTSUP		系统内核不支持fifo  
* ENOTDIR		path前缀不是合法目录  
* ENAMETOOLONG	path路径太长  
* EACCESS		用户对于path中的目录没有权限访问  
* EROFS			path指定的目录在只读文件系统  
* EEXIST		path指定文件已经存在  

	<!-- more -->

#注意事项
我们可以像访问普通文件一样访问命名管道文件。一般的文件I/O函数都可以用于FIFO。  
当打开一个FIFO，非阻塞标志（O_NONBLOCK）产生下列影响：  
  
* 在一般情况下（没有指定O_NOBLOCK），只读open要阻塞到某个其他进程为写而打开此FIFO。类似地，只写open要阻塞到某个进程为读而打开此FIFO。  
* 如果指定了O_NONBLOCK，则只读open立即返回。但是，如果没有进程已经为读而打开一个FIFO，那么只写open将出错返回-1，其errno是ENXIO。  

当多个写进程同时向同一FIFO进行写操作时，如果保证来自不同程序的数据块不相互交错，每个写操作必须原子话，FIFO的长度是需要考虑的一个很重要因素。系统对任一时刻在一个FIFO中可以存在的数据长度是有限制的。它由#define PIPE_BUF定义，在头文件limits.h中。在Linux和许多其他类UNIX系统中，它的值通常是4096字节，Red Hat Fedora9下是4096，但在某些系统中它可能会小到512字节。

* 当要写入的数据量不大于PIPE_BUF时，Linux将保证写入的原子性。如果此时管道空闲缓冲区不足以容纳要写入的字节数，则进入睡眠，直到当缓冲区中能够容纳要写入的字节数时，才开始进行一次性写操作。即写入的数据长度小于等于PIPE_BUF时，那么或者写入全部字节，或者一个字节都不写入，它属于一个一次性行为，具体要看FIFO中是否有足够的缓冲区。
* 当要写入的数据量大于PIPE_BUF时，Linux将不再保证写入的原子性。FIFO缓冲区一有空闲区域，写进程就会试图向管道写入数据，写操作在写完所有请求写的数据后返回。

#FIFO示例
创建FIFO

	ret = mkfifo(FIFO_NAME,0777);
	if ((-1 == ret) && (errno != EEXIST)) {
		perror("create fifo fail.\n");
		exit(1);
	}

读FIFO进程代码
	
	// create fifo
	int ret = mkfifo(FIFO_NAME,0777);
	if ((-1 == ret) && (errno != EEXIST)) {
		perror("create fifo fail.\n");
		exit(1);
	}
	
	int fd = open(FIFO_NAME, O_RDONLY);
	// int fd = open(FIFO_NAME, O_RDWR);
	if (-1 == fd) {
		printf("open fifo %s fail\n", FIFO_NAME);
		exit(1);
	}
	printf("open fifo %s for read suc\n", FIFO_NAME);

	char buf[1024];
	while (1 ) {
		ssize_t size = read(fd, buf, sizeof(buf));
		if (size > 0) {
			buf[size] = '\0';
			printf("read from fifo: %s\n", buf);
		} else if(size == 0) {
			printf("write end of fifo exit\n");
			break;
		} else {
			perror("read fail from fifo:");
			break;
		}
	}
	close(fd);
	return 0;
	
写FIFO进程代码

	// create fifo
	int ret = mkfifo(FIFO_NAME,0777);
	if ((-1 == ret) && (errno != EEXIST)) {
		perror("create fifo fail.\n");
		exit(1);
	}

	int fd = open(FIFO_NAME, O_WRONLY);
	if (-1 == fd) {
		printf("open fifo %s fail\n", FIFO_NAME);
		exit(1);
	}
	printf("open fifo %s for write suc", FIFO_NAME);

	while (1) {
		char buf[1024] = {0};
		if (gets(buf) == NULL) {
			break;
		}
		int len = strlen(buf);
		if (0 == len)
		  break;
		printf("%d\n", len);
		ssize_t size = write(fd, buf, len);
		if (size >= 0) {
			printf("write to fifo: %s, msg: %s, size:%ld \n", FIFO_NAME, buf, size);
		} else {
			perror("write fail\n");
			break;
		}
	}
	close(fd);
	return 0;


	

