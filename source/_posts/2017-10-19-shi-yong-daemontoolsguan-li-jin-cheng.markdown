---
layout: post
title: "使用daemontools管理进程"
date: 2017-10-19 11:23:50 +0800
comments: true
categories: ops 
tags: daemon 
---
本片博客介绍如何使用[daemontools](http://cr.yp.to/daemontools.html)管理在后台一直运行的进程，当程序异常退出时，deemontools可以将进程重新拉起。 
## 安装
deemontools只可以运行在UNIX系统。  
步骤：  

* 创建`/package`目录

        mkdir -p /package  
        chmod 1755 /package  
        cd /package  

* 下载[daemontools-0.76.tar.gz](http://cr.yp.to/daemontools/daemontools-0.76.tar.gz)
  
        cd /package  
        wget http://cr.yp.to/daemontools/daemontools-0.76.tar.gz  
        tar xzvf daemontools-0.76.tar.gz   
        cd admin/daemontools   

* 编译安装

        package/install

  在CentOS 7上编译时报错：  

        /usr/bin/ld: errno: TLS defini  tion in /lib/libc.so.6 section .tbss mismatches non-TLS reference in envdir.o
  解决方法如下：  
        
        将admin/daemontools-0.76/src/error.h中的extern int errno;替换为#include <errno.h>
  安装完成后会自动创建`/command  /service`两个目录，command目录下面包含各种daemontools工具 service目录包含daemontools管理的服务

<!-- more -->

## 配置daemontools开机启动
CentOS 7使用systemd来管理服务，新建/etc/systemd/system/daemontools.service文件，将如下内容放到新建文件中：  
    
        [Unit]
        Description=daemontools Start supervise
        After=getty.target
        [Service]
        Type=simple
        User=root
        Group=root
        Restart=always
        ExecStart=/command/svscanboot /dev/ttyS0
        TimeoutSec=0
        [Install]
        WantedBy=multi-user.target

* 启动服务

        systemctl start deamontools.service

* 查看服务状态
        
        systemctl status deamontools.service

* 设置开机启动
        
        systemctl enable daemontools.service

## 使用daemontools管理进程
   daemontools.service服务的启动进程为/command/svscanboot，查看svscanboot程序的进程树：  

        [root@hq-test command]# ps aux | grep svs
        root     19658  0.0  0.0 113116  1428 ?        Ss   Oct18   0:00 /bin/sh /command/svscanboot /dev/ttyS0
        root     19660  0.0  0.0   4344   480 ?        S    Oct18   0:00 svscan /service
        root     30566  0.0  0.0 112640   956 pts/0    R+   14:35   0:00 grep --color=auto svs
        [root@hq-test command]# pstree -ap 19658
        svscanboot,19658 /command/svscanboot /dev/ttyS0
          ├─readproctitle,19661 service errors:...
            └─svscan,19660 /service
                  └─supervise,20625 test
                            └─test,22043
        
   通过进程树我们看到svscanboot进程启动svscan进程，svscan读取/service目录，svscan启动supervise进程管理我们的应用进程test
   下面讲解如何向daemontools添加一个应用进程:  
   应用进程所在目录为：`/tmp/test`
    
        [root@hq-test test]# ls
        test
        [root@hq-test test]#

   * 在应用进程所在文件夹新建run文件 
    
        [root@hq-test test]# cat run
        #!/bin/sh
        exec /tmp/test/test 1>>/tmp/test/1.out 2>&1
        [root@hq-test test]# pwd
        /tmp/test
        [root@hq-test test]# ls
        run  test
        [root@hq-test test]#

    * 向svscanboot注册服务
        
        [root@hq-test test]# ln -s /tmp/test /service/test
        [root@hq-test test]# ls /service/
        test
        [root@hq-test test]# ls -al /service/
        total 8
        drwxr-xr-x   2 root root 4096 Oct 19 17:23 .
        drwxr-xr-x. 23 root root 4096 Oct 18 19:25 ..
        lrwxrwxrwx   1 root root    9 Oct 19 17:23 test -> /tmp/test
        [root@hq-test test]#
    
       在/service目录中建立到应用程序目录的软连接后，svscan将会新建supervise进程，启动应用进程:
        
        [root@hq-test test]# pstree -ap 19658
        svscanboot,19658 /command/svscanboot /dev/ttyS0
          ├─readproctitle,19661 service errors:...
            └─svscan,19660 /service
                  └─supervise,5821 test
                            └─test,5822

    * 查看服务状态 
        svstat services
        services 为/service目录下的软链接
         
        [root@hq-test test]# svstat /service/test/
        /service/test/: up (pid 5822) 303 seconds

    * 管理服务  

       svc opts services 
       opts包含如下选项：  
            
            -u: Up。 如果服务不在运行，启动服务。如果服务停止了，重启服务。  
            -d: Down。 如果服务在运行，发送一个TERM信号，之后发送CONT信号。服务停止后，不重启服务。  
            -o: Once。如果服务不在运行，则启动服务。服务停止后，不在启动服务。  
            -p: Pause。向服务发送STOP信号。  
            -c: Continue。向服务发送一个CONT信号。  
            -h: Hangup。向服务发送一个HUP信号。  
            -a: Alarm。向服务发送一个ALARM信号。  
            -i: Interrupt。向服务发送一个INT信号。  
            -t: Terminate。向服务发送一个TERM信号。  
            -k: Kill。 向服务发送一个KILL信号。  
            -x: Exit。服务进程退出后supervise将会退出。  
      停止服务进程：  
            
            [root@hq-test test]# pstree -pa 19658
            svscanboot,19658 /command/svscanboot /dev/ttyS0
              ├─readproctitle,19661 service errors:...
                └─svscan,19660 /service
                      └─supervise,10630 test
                                └─test,10631
            [root@hq-test test]# svstat /service/test/
            /service/test/: up (pid 10631) 67 seconds
            [root@hq-test test]# svc -d /service/test/
            [root@hq-test test]# svstat /service/test/
            /service/test/: down 5 seconds, normally up
            [root@hq-test test]#

     重启服务进程：  
           
           [root@hq-test test]# svc -u /service/test/
           [root@hq-test test]# svstat /service/test/
           /service/test/: up (pid 10750) 2 seconds
           [root@hq-test test]# pstree -pa 19658
           svscanboot,19658 /command/svscanboot /dev/ttyS0
             ├─readproctitle,19661 service errors:...
               └─svscan,19660 /service
                     └─supervise,10630 test
                               └─test,10750
           [root@hq-test test]# svstat /service/test/
           /service/test/: up (pid 10750) 12 seconds
           [root@hq-test test]#

    删除服务：  
           先删除/service目录中的服务链接，
           svc 命令中使用服务的目标链接

           [root@hq-test test]# rm /service/test
           rm: remove symbolic link ‘/service/test’? y
           [root@hq-test test]# svc -dx /tmp/test
           [root@hq-test test]# pstree -pa 19658
           svscanboot,19658 /command/svscanboot /dev/ttyS0
             ├─readproctitle,19661 service errors:...
               └─svscan,19660 /service
           [root@hq-test test]#
## 参考

    * [daemontools](http://cr.yp.to/daemontools/svc.html)
    
