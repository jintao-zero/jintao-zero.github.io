---
layout: post
title: "grep命令"
date: 2016-10-05 20:48:14 +0800
comments: true
categories: DevOps
tags: DevOps
---
#grep命令
grep, egrep, fgrep, zgrep, zegrep, zfgrep 文件模式搜索

```
grep [-abcdDEFGHhIiJLlmnOopqRSsUVvwxZ] [-A num] [-B num] [-C[num]] [-e pattern] [-f file] [--binary-files=value] [--color[=when]]
          [--colour[=when]] [--context[=num]] [--label] [--line-buffered] [--null] [pattern] [file ...]
```
grep搜索输入文件，输出匹配一个或者多个模式的文件行。通常情况，不包括结尾换行符的文件行匹配指定模式中的表达式即为匹配成功。空表达式匹配所有行。匹配行将会打印到标准输出。  
grep命令用于简单模式和基本正则表达式（BRES）；egrep命令可以处理扩展正则表达式。查看re_format(7）关于正则表达式的详细信息。fgre较grep和egrep更快，但是只能处理固定模式（比如，它不能解析正则表达式）。匹配模式可以由一行或者多行组成，以便于匹配输入内容的一部分。  
zgrep，zegrep，zfgrep分别与grep，egrep，fgrep功能类似，只是可以接收由compress或者gzip压缩工具压缩过的文件作为输入。  
##参数

	-A num, --after-context=num
打印匹配行后面的num行文本，与-B和-C选项功能类似

	-a, --text
	
将所有输入当作文本文件处理。如果文件包含二进制字符，grep简单打印`Binary file ... matches`。使用这个参数强制grep输出匹配行。    

	-B num, --before-context=num
打印每个匹配行前面num行文本。  

	-b, --byte-offset  
打印匹配模式在文件中的字节偏移数

	-C[num, --context=num]
指定打印匹配行前后文本行数。默认为2，相当于设置-A 2 -B 2

	-c, --count
只向标准输出打印匹配文本行的行数

	--colour=[when, --color=[when]]
打印时，用GREP_COLOR环境变量里面保存的颜色设置标记匹配到的模式

	-D action, --devices=action
为devices，FIFOS和sockets指定动作。默认动作是读，把他们当作普通文件处理。如果action指定为skip，这些设备将会被跳过。
	
	-d action, --directories=action
为目录指定特殊的动作。默认是读动作，把目录当作普通文件处理。`skip`动作是默默跳过目录，`recurse`动作是循环读取目录。

	-E, --extended-regexp
按照扩展正则表达式来解析pattern模式（强制grep表现为egrep）

	-e pattern, --regexp=pattern
指定匹配用的模式。可以多次使用-e选项来指定多个模式串
	
	--exclude
使用此选项将匹配参数模式的文件去除，不尽兴模式串的匹配。  
	
	--exclude-dir
将某些目录去除，不尽兴模式匹配

	-F, --fixed-strings
将模式解析为固定字符串集合（强制grep为fgrep）

	-f file, --file=file
从file文件中读取匹配用的模式串

	-G, --basic-regexp
按照基本正则表达式解析模式串（强制grep工作模式为传统grep）
	
	-H
在匹配文本行前头打印所属文件名

	-h, --no-filename
在匹配文本行前头不打印文件名

	--help
打印简要帮助信息
	
	-I
忽略二进制文件

	-i, --ignore-case
进行大小无关匹配，grep默认是大小相关
	
	--include
指定符合文件名模式的文件进行模式匹配
	
	--include-dir
如果指定-R参数，只有符合模式的目录才进行模式匹配
	
	-J, --bz2decompress
进行文本匹配之前，解压bzip压缩过的文件
	
	-L, --files-without-match
只打印不包含指定模式文本行文件的文件名

	-l, --files-with-matches
只打印包含指定模式文本行文件的文件名，不打印文本行

	--mmap
使用mmap代替read来读取输入，在某些环境下能够达到更高性能，也可能引起为定义行为
	
	-m num, --max-count=num
达到`num`处匹配后，停止继续读取输入
	
	-n, --line-number
输出结果时，每行前面打印匹配行在文件中的行数。

	--null
打印文件名时，后面跟零字节
	
	-O
8888
	
	-o, --only-matching
只打印每行中匹配的部分
	
	-p
如果指定`-R`选项，不跟随符号链接，这个是默认选项
	
	-q, --quiet, --silent






	



