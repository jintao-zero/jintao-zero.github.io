---
layout: post
title: "hadoop环境搭建"
date: 2015-09-19 15:41:51 +0800
comments: true
categories: CloudComputing
tags: Hadoop
---
最近开始进行Hadoop分布式计算相关内容的学习，先从搭建开发环境开始。下面记录一下搭建Hadoop开发环境的过程，以及注意点。
本节介绍如何快速搭建和配置一个单节点Hadoop环境，便于使用Hadoop MapReduce和Hadoop Distributed File System(HDFS)。
## 依赖条件
支持的操作系统平台    
GNU/Linux支持作为Hadoop的开发和生产环境，Hadoop已经证明可以在GNU/Linux集群上运行2000节点。本文使用Ubuntu操作系统做为实验环境。
	
	Linux node8 3.16.0-30-generic #40~14.04.1-Ubuntu SMP Thu Jan 15 17:43:14 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
	
Windows系统也可以搭建Hadoop集群，本文暂时没有涉及，可以参考[wiki page](http://wiki.apache.org/hadoop/Hadoop2OnWindows)

依赖的软件  

* Java环境  
	安装方法：
	
	1、到oracle官网下载[JDK二进制压缩包](http://download.oracle.com/otn-pub/java/jdk/8u60-b27/jdk-8u60-linux-x64.tar.gz)  
	2、将压缩包解压到/usr/local/lib/目录下，安装目录也可以换到其他路径    
		sudo tar xzvf jdk-8u45-linux-x64.tar.gz -C /usr/local/lib/  
	3、修改/etc/profile文件配置JAVA_HOME、PATH、CLASSPATH环境变量  
		JAVA_HOME=/usr/local/lib/jdk1.8.0_45  
		PATH=$JAVA_HOME/bin:$PATH  
		CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar   
	4、运行java -version查看是否显示java版本信息
	
* ssh远程控制软件  
	如果系统没有运行sshd服务，则Ubuntu下运行apt-get命令安装ssh服务端
	sudo apt-get install openssh-server
	
##下载Hadoop软件
本文采用的是Hadoop最新的稳定版本[2.7.1](http://apache.fayea.com/hadoop/common/current/hadoop-2.7.1.tar.gz)

<!-- more -->  

##准备运行集群
1、将压缩包解压到一个安装目录下  

	sudo tar xzvf hadoop-2.7.1.tar.gz -C /home/jintao/toolkit/
	
2、打开安装目录下etc/hadoop/hadoop-env.sh

	添加一句 export JAVA_HOME=/usr/local/lib/jdk1.8.0_45
	
3、运行bin/hadoop命令，查看hadoop命令使用帮助

##单机模式
默认情况，Hadoop运行在非分布式模式，一个单独Java进程。这种模式调试时很有用。下面的例子，拷贝hadoop目录下面的etc/hadoop/*.xml作为输入，运行hadoop打印文件中匹配正则表达式的内容，输出到指定的output目录。

	$ mkdir input  
	$ cp etc/hadoop/*.xml input
	$ bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.1.jar grep input output 'dfs[a-z.]+'
	$ cat output/*
	
##伪分布式
Hadoop可以在单个节点运行在伪分布式模式，每个Hadoop守护进程运行在不同的Java进程。
###配置
编辑以下文件：
etc/hadoop/core-site.xml

	<configuration>
    	<property>
        	<name>fs.defaultFS</name>
        	<value>hdfs://localhost:9000</value>
    	</property>
	</configuration>

etc/hadoop/hdfs-site.xml
	
	<configuration>
    	<property>
        	<name>dfs.replication</name>
        	<value>1</value>
    	</property>
	</configuration>
### 设置ssh无密码登录
查看当前是否可以无密码登录：
	
	$ ssh localhost

如果不可以无密码登录，则执行一下命令：

	$ ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa
	$ cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys

### 执行
以下指令在本地运行一个MapReduce任务。  
1、格式化hdfs文件系统	
	
	$ bin/hdfs namenode -format
	
默认namenode存储路径：/tmp/hadoop-jintao/dfs/name

2、运行NameNode和DataNode守护进程：

	$ sbin/start-dfs.sh
	
hadoop守护进程日志输出到$HADOOP_LOG_DIR目录（默认在$HADOOP_HOME/logs安装目录的logs目录下）。使用jps查看java进行：
	
	jintao@node8:~/toolkit/hadoop-2.7.1$ jps
	5248 DataNode
	5473 SecondaryNameNode
	5105 NameNode
	13380 Jps

3、通过web接口查看NameNode信息；默认路径：
	
	http://localhost:50070/

4、创建执行MapReduce任务需要的HDFS路径：/user/<username\>

	$ bin/hdfs dfs -mkdir /user	
	$ bin/hdfs dfs -mkdir /user/jintao

5、将试验用的数据放到hdfs上input目录下：
	
	$ bin/hdfs dfs -put etc/hadoop input

6、运行hadoop自带的grep任务：

	$ bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.1.jar grep input output 'dfs[a-z.]+'
	
7、将结果文件从hdfs上获取到本地目录outpu中，并查看结果：
	
	$ bin/hdfs -get output output
	$ cat output/*

或者直接查看hdfs上面的结果文件：
	
	$ bin/hdfs -cat output/*
	
8、关闭hadoop后台进程
	
	$ sbin/stop-dfs.sh
	
###伪分布式运行YARN
设置一些参数，可以在伪分布式环境下运行YARN框架管理MapReduce任务，YARN框架会启动ResourceManager和NodeManager守护进程。  
在上面1到4步完成的基础上，进行如下设置：  
1、配置etc/hadoop/mapred-site.xml，如果不存在mapred-site.xml，则拷贝一份mapred-site.xml.template命名为mapred-site.xml

	<configuration>
    	<property>
        	<name>mapreduce.framework.name</name>
        	<value>yarn</value>
    	</property>
	</configuration>

配置etc/hadoop/yarn-site.xml：

	<configuration>
	    <property>
        	<name>yarn.nodemanager.aux-services</name>
        	<value>mapreduce_shuffle</value>
    	</property>
	</configuration>

2、启动ResourceManager和NodeManager后台进程：
	
	$ sbin/start-yarn.sh
	$ jps
	15858 NodeManager
	15301 DataNode
	15542 SecondaryNameNode
	16119 Jps
	15160 NameNode
	15723 ResourceManager
	
3、浏览ResourceMnaager信息：
	
	http://localhost:8088/

4、运行一个MapReduce任务：
	
	$ bin/hdfs dfs -rm -r -f output
	$ bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.1.jar grep input output 'dfs[a-z.]+'
	
5、关闭yarn框架进程
	
	$ sbin/stop-yarn.sh
	
	 
