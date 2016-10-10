---
layout: post
title: "hadoop环境搭建"
date: 2015-09-19 15:41:51 +0800
comments: true
categories: bigdata 
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
	
##集群环境
搭建四台机器（node1、node2、node3、node4）的Hadoop集群环境：  
1、node1部署NameNode和ResourceManager  
2、node2、node3、node4作为slave节点部署DataNode和NodeManager  

典型集群环境下，应该将单独一台机器运行NameNode，另外一台机器运行ResourceManager，这些是主控节点。其他服务（比如Web App Proxy Server和MapReduce Job History server）通常运行在一台指定机器或者共享设备上，依赖于系统负载。集群中其他机器都同时部署DataNode和NodeManager，这些都是slaves节点。


###准备工作
1、每台机器都安装JAVA环境  
   参考上面单机环境搭建，进行安装、配置环境变量
  
2、编辑hosts文件
	编辑每台机器的hosts文件，将hostname和ip的对应关系添加到/etc/hosts中
	
	192.168.88.120 node0
	192.168.88.121 node1
	192.168.88.122 node2
	192.168.88.123 node3
	
在各台机器上通过主机名ping各台机器，查看是否能够ping通

3、配置ssh无密码登录
每台机上运行，生存密钥
	
	$ ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa
	$ cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys
	
ssh集群启动时，主控节点会通过ssh远程登录到slaves节点执行脚本启动slaves节点，因此我们需要配置主控节点到slaves节点的无密码登录：  
将主控节点的pub密钥拷贝到各个slaves节点~/.ssh/目录下

	scp id_rsa.pub jintao@node1:~/.ssh/node0.pub

将node0.pub文件的内容追加到各个节点的authorized_keys文件中：
	
	cat node0.pub >> authorized_keys
	
从node0节点ssh到slaves节点上，查看是否不需要密码  

4、创建hadoop工作目录

	mkdir /home/jintao/hadoop_dir
如果将hadoop工作目录配置在其他目录下，则需要使hadoop的运行账户对该目录具有读写权限
	
5、下载Hadoop安装程序  
从官网下载安装程序，解压到~/toolkit/目录下，toolkit后面陆续会安装其他zookeeper、hbase等项目

###配置Hadoop
管理员应该使用etc/haddop-env.sh和etc/hadoop/mapred-env.sh和etc/hadoop/yarn-env.sh三个相关脚本来配置Hadoop进程环境。  

1、打开安装目录下etc/hadoop/hadoop-env.sh

	添加一句 export JAVA_HOME=/usr/local/lib/jdk1.8.0_45
	
2、vim core-site.xml
	
	<configuration>
        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://node0:9000</value>
        </property>
        <property>
                <name>hadoop.tmp.dir</name>
                <value>/home/jintao/hadoop_dir</value>
                <description>A base for other temporary directories.</description>
        </property>
	</configuration>

设置fs.defaultFS和hadoop.tmp.dir属性

3、vim hdfs-site.xml

	<property>
    	<name>dfs.replication</name>
        <value>2</value>
    </property>

4、vim mapred-site.xml

	<property>
    	<name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>

5、vim yarn-site.xml

	<property>
    	<name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
    	<name>yarn.resourcemanager.hostname</name>
        <value>node0</value>
    </property>
6、vim slaves
	
	ode1
	node2
	node3

7、将配置好的hadoop程序同步到slaves节点上
	
	scp -r toolkit/ node1:~
	scp -r toolkit/ node2:~
	scp -r toolkit/ node3:~
	
8、启动namenode节点  

	1、cd ~/toolkit/hadoop-2.7.1/
	2、bin/hdfs namenode -format
	3、sbin/start-all.sh
	
	
9、node0上查看jps
	
	9904 Jps
	9426 SecondaryNameNode
	9190 NameNode
	9607 ResourceManager

10、node1、node2、node3上查看jps
	
	9507 Jps
	9323 NodeManager
	9213 DataNode
	
11、系统启动日志记录在hadoop-2.7.1/logs目录下
	
	
