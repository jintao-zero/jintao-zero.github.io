---
layout: post
title: "使用strace调试进程"
date: 2017-09-04 18:44:00 +0800
comments: true
categories: ops
tags: linux ops
---
`Strace`调试工具可以帮助定位问题。    
`Strace`工具可以用来记录指定程序运行过程中涉及的系统调用和信号。在没有获取程序源代码的情况下，strace工具可以有效帮助对程序的调试。strace工具提供一个二进制程序从开始到结束的执行顺序。    
下面几个例子展示如何使用`strace`工具：  

<!-- more -->
1. 跟踪可执行程序的执行  
可以使用strace工具跟踪任何可执行程序的执行。下面例子是使用strace跟踪linux ls命令的执行过程：  

	```
$  strace ls
execve("/bin/ls", ["ls"], [/* 21 vars */]) = 0
brk(0)                                  = 0x8c31000
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
mmap2(NULL, 8192, PROT_READ, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0xb78c7000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
open("/etc/ld.so.cache", O_RDONLY)      = 3
fstat64(3, {st_mode=S_IFREG|0644, st_size=65354, ...}) = 0
...
...
...
	```


2. 使用-e参数跟踪特定系统调用  
strace默认跟踪执行程序的所有系统调用。可以使用`-e`参数跟踪特定系统调用：  

	```
$ strace -e open ls
open("/etc/ld.so.cache", O_RDONLY)      = 3
open("/lib/libselinux.so.1", O_RDONLY)  = 3
open("/lib/librt.so.1", O_RDONLY)       = 3
open("/lib/libacl.so.1", O_RDONLY)      = 3
open("/lib/libc.so.6", O_RDONLY)        = 3
open("/lib/libdl.so.2", O_RDONLY)       = 3
open("/lib/libpthread.so.0", O_RDONLY)  = 3
open("/lib/libattr.so.1", O_RDONLY)     = 3
open("/proc/filesystems", O_RDONLY|O_LARGEFILE) = 3
open("/usr/lib/locale/locale-archive", O_RDONLY|O_LARGEFILE) = 3
open(".", O_RDONLY|O_NONBLOCK|O_LARGEFILE|O_DIRECTORY|O_CLOEXEC) = 3
Desktop  Documents  Downloads  examples.desktop  libflashplayer.so 
Music  Pictures  Public  Templates  Ubuntu_OS  Videos
	```
如果想执行多个系统调用，可以使用`-e trace=`选项。下面例子跟踪open和read系统调用：  

	```
	$ strace -e trace=open,read ls /home
open("/etc/ld.so.cache", O_RDONLY)      = 3
open("/lib/libselinux.so.1", O_RDONLY)  = 3
read(3, "\177ELF\1\1\1\3\3\1\260G004"..., 512) = 512
open("/lib/librt.so.1", O_RDONLY)       = 3
read(3, "\177ELF\1\1\1\3\3\1\300\30004"..., 512) = 512
..
open("/lib/libattr.so.1", O_RDONLY)     = 3
read(3, "\177ELF\1\1\1\3\3\1\360\r004"..., 512) = 512
open("/proc/filesystems", O_RDONLY|O_LARGEFILE) = 3
read(3, "nodev\tsysfs\nnodev\trootfs\nnodev\tb"..., 1024) = 315
read(3, "", 1024)                       = 0
open("/usr/lib/locale/locale-archive", O_RDONLY|O_LARGEFILE) = 3
open("/home", O_RDONLY|O_NONBLOCK|O_LARGEFILE|O_DIRECTORY|O_CLOEXEC) = 3
bala
	```
3. 保存调用过程到文件  
下面例子使用`-o`参数保存调用过程到文件中：  

	```
	$ strace -o output.txt ls
	Desktop  Documents  Downloads  examples.desktop  libflashplayer.so
Music  output.txt  Pictures  Public  Templates  Ubuntu_OS  Videos

	$ cat output.txt 
execve("/bin/ls", ["ls"], [/* 37 vars */]) = 0
brk(0)                                  = 0x8637000
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
mmap2(NULL, 8192, PROT_READ, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0xb7860000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
open("/etc/ld.so.cache", O_RDONLY)      = 3
fstat64(3, {st_mode=S_IFREG|0644, st_size=67188, ...}) = 0
...
...
	```
4. 使用`-p`参数跟踪正在运行的进程  
	先使用ps命令获取正在运行进程的id  
	```
	$ ps -C firefox-bin
	  PID TTY          TIME CMD
 	1725 ?        00:40:50 firefox-bin
	```
	使用`-p`指定strace需要跟踪进程的id
	
	```
	$ sudo strace -p 1725 -o firefox_trace.txt

	$ tail -f firefox_trace.txt
	```
	进程系统调用将会被写入到文件中，使用tail命令可以查看最近执行序列。  
	
5. 使用`-t`参数为每行输出显示时间戳  

	```
	$ strace -t -e open ls /home
20:42:37 open("/etc/ld.so.cache", O_RDONLY) = 3
20:42:37 open("/lib/libselinux.so.1", O_RDONLY) = 3
20:42:37 open("/lib/librt.so.1", O_RDONLY) = 3
20:42:37 open("/lib/libacl.so.1", O_RDONLY) = 3
20:42:37 open("/lib/libc.so.6", O_RDONLY) = 3
20:42:37 open("/lib/libdl.so.2", O_RDONLY) = 3
20:42:37 open("/lib/libpthread.so.0", O_RDONLY) = 3
20:42:37 open("/lib/libattr.so.1", O_RDONLY) = 3
20:42:37 open("/proc/filesystems", O_RDONLY|O_LARGEFILE) = 3
20:42:37 open("/usr/lib/locale/locale-archive", O_RDONLY|O_LARGEFILE) = 3
20:42:37 open("/home", O_RDONLY|O_NONBLOCK|O_LARGEFILE|O_DIRECTORY|O_CLOEXEC) = 3
bala
	```
6. 使用`-r`参数输出每个系统调用执行时间  

	```
	$ strace -r ls 
     0.000000 execve("/bin/ls", ["ls"], [/* 37 vars */]) = 0
     0.000846 brk(0)                    = 0x8418000
     0.000143 access("/etc/ld.so.nohwcap", F_OK) = -1 ENOENT (No such file or directory)
     0.000163 mmap2(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0xb787b000
     0.000119 access("/etc/ld.so.preload", R_OK) = -1 ENOENT (No such file or directory)
     0.000123 open("/etc/ld.so.cache", O_RDONLY) = 3
     0.000099 fstat64(3, {st_mode=S_IFREG|0644, st_size=67188, ...}) = 0
     0.000155 mmap2(NULL, 67188, PROT_READ, MAP_PRIVATE, 3, 0) = 0xb786a000
     ...
     ...
	```
7. 使用`-c`参数显示系统调用统计报告  
	使用`-c`参数可以输出程序执行期间系统调用的统计数据：包括执行时间占比，执行时间，执行次数，错误数:  
	
		$ strace -c ls /home
		bala
		% time     seconds  usecs/call     calls    errors syscall
		------ ----------- ----------- --------- --------- ----------------
  		-nan    0.000000           0         9           read
  		-nan    0.000000           0         1           write
  		-nan    0.000000           0        11           open
  		
 	 		
  		-nan    0.000000           0        13           close
  		-nan    0.000000           0         1           execve
  		-nan    0.000000           0         9         9 access
  		-nan    0.000000           0         3           brk
  		-nan    0.000000           0         2           ioctl
  		-nan    0.000000           0         3           munmap
  		-nan    0.000000           0         1           uname
  		-nan    0.000000           0        11           mprotect
  		-nan    0.000000           0         2           rt_sigaction
  		-nan    0.000000           0         1           rt_sigprocmask
  		-nan    0.000000           0         1           getrlimit
  		-nan    0.000000           0        25           mmap2
  		-nan    0.000000           0         1           stat64
  		-nan    0.000000           0        11           fstat64
  		-nan    0.000000           0         2           getdents64
  		-nan    0.000000           0         1           fcntl64
  		-nan    0.000000           0         2         1 futex
  		-nan    0.000000           0         1         set_thread_area
  		-nan    0.000000           0         1           set_tid_address
  		-nan    0.000000           0         1           statfs64
  		-nan    0.000000           0         1           	set_robust_list
		------ ----------- ----------- --------- --------- ----------------
		100.00    0.000000                   114        10 total


