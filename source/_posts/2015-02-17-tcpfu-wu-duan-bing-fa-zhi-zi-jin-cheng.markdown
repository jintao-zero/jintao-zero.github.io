---
layout: post
title: "tcp服务端并发之子进程"
date: 2015-02-17 14:56:56 +0800
comments: true
categories: ['tcp/ip']
---
上一篇博客中实现的tcp服务端每次只能处理一个客户端连接，如果有多个客户端同时与服务端进行通讯的话，那只能进行等待上一个客户端通讯完成才可以，那么如何修改服务端为并发服务器程序呢？Unix中编写并发服务器程序最简单的方法就是fork一个子进程来服务每个客户。以下为一个典型的并发服务器的轮廓：

	pid_t pid;
	int listenfd, connfd;
	listenfd = Socket(...);
		/* fill in sockaddr_in{} with server's well-known port */
	Bind*(listenfd, ...);
	Listen(listenfd, LISTENQ);
	for ( ; ; ) {
		connfd = Accept(listenfd, ...);  /* probably blocks */
		if ((pid = Fork()) == 0) {
			Close(listenfd); 	/* child close listening socket */
			doit(connfd); 		/* process the request */
			close(connfd); 		/* child terminates */
			exit(0);			/* child terminate */
		}
		Close(connfd); 			/* parent closes connected socket */
	}

fork函数
	
	pid_t fork(void);

fork创建一个新进程。新进程是当前运行进程的一个拷贝。fork函数一次调用，两次返回。对于子进程返回值为0，对于父进程，返回值为子进程id。  

以下为根据上面的并发服务器轮廓代码修改的tcp客户端代码：

<!-- more -->

	for (; ; ) {

		if ((connsock = accept(sock_server, (struct sockaddr *)&clientaddr, &clientaddr_len)) < 0) {
			if (errno == EINTR) {
				continue;
			}
			else {
				printf("accept fail. reason:%s \n", strerror(errno));
				close(sock_server);
				exit(0);
			}
		}
		printf("new client connected, %s:%d \n", inet_ntoa(clientaddr.sin_addr), clientaddr.sin_port);

		int child = fork();
		if (child == 0) { // child process

			close(sock_server);
			printf("child process:%d \n", getpid());
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
			exit(0);
		}

		// parent process
		close(connsock);
	}
	
以上tcp服务端程序可以并发处理多个客户端请求，但是在通讯结束，子进程调用exit函数退出后，会产生僵死进程, 使用ps命令可以看到多个server进程状态为Z+：  
{% img /images/zombie.png %}
设置僵死状态的目的是维护子进程的信息，以便父进程在以后某个时候获取。这些信息包括子进程的进程ID、终止状态以及资源利用信息（CPU时间、内存使用量等等）。如果一个进程终止，而该进程有子进程处于僵死状态，那么它的所有僵死子进程的富进程ID将被重置为1（init进程）。继承这些子进程的init进程将清理它们。  
僵死进程占用内核中的空间，最终可能导致我们耗尽进程资源，因此我们需要及时的清理僵死子进程，无论何时我们fork子进程都得wait它们，以防它们变成僵死进程。每个子进程在退出时，内核都会向父进程发送SIGCHLD信号，因此父进程需要建立一个俘获SIGCHLD信号的信号处理函数，在函数体中调用wait。  

wait函数

	pid_t wait(int *stat_loc);
wait函数阻塞调用进程直到获取到一个退出子进程的信息，成功返回后，stat_loc返回退出子进程的退出信息。SIGCHILD信号处理函数为：

	void chld_sig_handler(int signo)
	{
        int stat_loc;
        wait(&stat_loc);
        return ;
   	}

sigaction函数
	
	struct  sigaction {
             union __sigaction_u __sigaction_u;  /* signal handler */
             sigset_t sa_mask;               /* signal mask to apply */
             int     sa_flags;               /* see signal options below */
     };

     union __sigaction_u {
             void    (*__sa_handler)(int);
             void    (*__sa_sigaction)(int, struct __siginfo *,
                            void *);
     };

	#define sa_handler      __sigaction_u.__sa_handler
    #define sa_sigaction    __sigaction_u.__sa_sigaction

	int sigaction(int sig, const struct sigaction *restrict act, struct sigaction *restrict oact);
sigaction函数用来修改进程对于某个信号的处理函数
参数sig为需要修改处理函数的信号id
参数act说明了该信号处理函数，以及进入处理函数时的信号掩码
	
	// register SIGCHLD process handler
    struct sigaction act;
    struct sigaction oact;
    act.sa_handler = chld_sig_handler;
    sigemptyset(&act.sa_mask);
    act.sa_flags = 0;
    if (sigaction(SIGCHLD, &act, &oact) < 0) {
    	printf("set SIGCHLD handler fail. reason:%s \n", strerror(errno));
    	close(sock_server);
    	return -1;
    }
在tcp服务代码中添加了SIGCHLD信号处理函数后，重新测试，但是发现仍然会出现僵死子进程：  
{% img /images/zombie_wait.png %}  
Unix信号处理机制中，  
1、当一个信号达到后，进入该信号的处理函数后，进程将屏蔽该信号   
2、多个同样类型信号同时到达时，信号处理函数只执行一次，其他的信号不进行排队

这样当多个客户端连接同时结束时，会导致服务端子进程同时退出，多个SIGCHLD信号同时产生递送到服务端父进程，那么一种可能的情况是，在信号处理函数被调用之前，信号都到达，那么只有一个信号被处理，其他信号不会排队，将会被丢弃，这样仍然产生僵死子进程。  
为了解决这样问题，可以在信号处理函数中多次执行wait操作，获取已经结束子进程的退出信息。但是上面的wait接口是阻塞等待、直到有新退出进程，这样的话就会影响进程的正常执行，下面介绍另外一个

	pid_t waitpid(pid_t pid, int *stat_loc, int options);

参数pid指定需要等待的进程集。pid为-1时，等待任何子进程。pid为0时，等待调用者进程组里面的任何子进程。pid大于0时，等待进程号为pid的子进程。
参数stat_loc用来保存进程退出状态。
参数options,为WNOHANG、WUNTRACED选项的或。指定WNOHANG操作时，如果尚没有退出的子进程，你们waitp函数不阻塞等待，会返回0。指定WUNTRACED选项时，如果子进程是由于SIGTTIN,SIGTTOU,SIGTSTP,SIGSTOP信号而处于stopped状态，那么wait操作也会获取到这些进程的状态信息。  
需要修改SIGCHLD信号处理函数，使用waitp函数，设置options位WNOHANG来读取所有已经退出子进程的进程状态，防止变为僵尸进程。  

	void chld_sig_handler(int signo)
	{
		int stat_loc;
		int options = WNOHANG;
		int pid;
		while ((pid = waitpid(-1, &stat_loc, options)) > 0) {
			printf("wait pid:%d suc\n", pid);	// normally we should call i/o func in signal handler
		}
	return ;
	}
	
对于服务器主进程而言，大部分的时间都是阻塞在accept系统调用上，这一类的系统调用也称为慢系统调用（slow system call），适用与慢系统调用的一个规则是：当阻塞与某个慢系统调用的一个进程捕获某个信号且相应信号处理函数返回时，该系统调用可能返回一个EINTR错误。有些内核自动重启某些被中断的系统调用。为了便于移植，当我们编写捕获信号的程序时（多数并发服务器捕获SIGCHLD），我们必须对慢系统调用返回EINTR有所准备。修改服务器代码为：  
	
	if ((connsock = accept(sock_server, (struct sockaddr *)&clientaddr, &clientaddr_len)) < 0) {
		if (errno == EINTR) {
			continue;
		} else {
			printf("accept fail. reason:%s \n", strerror(errno));
			close(sock_server);
			exit(0);
		}
	}

本片文章的目的时总结我们在网络编程时可能会遇到的三种情况：
(1) 当fork子进程时，必须捕获SIGCHLD信号
(2) 当捕获信号时，必须处理被中断的系统调用
(3) SIGCHLD的信号处理函数必须正确编写，应适用waitpid函数以免留下僵尸进程





