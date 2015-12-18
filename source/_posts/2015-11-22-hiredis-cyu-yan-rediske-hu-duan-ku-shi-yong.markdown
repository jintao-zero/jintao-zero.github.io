---
layout: post
title: "hiredis C语言Redis客户端库使用"
date: 2015-11-22 11:21:43 +0800
comments: true
categories: nosql
tags: Nosql Redis
---
Redis是一款依据BSD开源协议发型的高性能Key-Value存储系统(cache and store)。它通常称为数据结构服务器，因为值(value)可以是字符串(String)，哈希(Hase)，列表(List)，集合(Set)和有序集合(sorted sets)等类型。  

本文对RedisC语言客户端库[hireis](https://github.com/redis/hiredis#using-replies)的安装、使用进行简单介绍。
##Redis安装
参考[Redis](http://redis.io/download)官网中的安装指导进行安装：  
1、下载、解压和编译安装  

	$ wget http://download.redis.io/releases/redis-3.0.5.tar.gz
	$ tar xzf redis-3.0.5.tar.gz
	$ cd redis-3.0.5
	$ make

2、运行服务端
	编译声称的二进制文件在redis目录下的src子目录中，如下运行：  
	
	$ src/redis-server

3、使用命令行工具与redis-server进行交互：  

	$ src/redis-cli
	redis> set foo bar
	OK
	redis> get foo
	"bar"
	
详细的redis使用文档请参考[官网](http://redis.io/documentation)

<!-- more -->

##hiredis安装
上面下载的Redis源代码目录的deps/hiredis目录中包含了hiredis的源代码，也可以从hiredis在github的网址下载：

	$ git clonet https://github.com/redis/hiredis.git
	$ make
	$ make install
	
以上命令会将相关.h头文件拷贝到/usr/local/include/hiredis/目录，  
同时将hiredis相关库文件拷贝到/usr/local/lib/目录。

	-rw-rw-r--  1 jintao jintao 431534 11月  9 21:28 libhiredis.a
	lrwxrwxrwx  1 root   root       15 11月  9 21:50 libhiredis.so -> libhiredis.so.0*
	lrwxrwxrwx  1 root   root       18 11月  9 21:50 libhiredis.so.0 -> libhiredis.so.0.11*
	-rwxrwxr-x  1 jintao jintao 219340 11月  9 21:50 libhiredis.so.0.11*

##hiredis接口
hiredis库接口分为同步和异步两种接口，本文主要使用的是同步接口，主要涉及一下几个:  

	redisContext *redisConnect(const char *ip, int port);
	void *redisCommand(redisContext *c, const char *format, ...);
	void freeReplyObject(void *reply);
	
###建立连接

	redisContext *redisConnect(const char *ip, int port);

redisConnect函数用来创建一个到redis服务器的连接，并返回一个redisContext类型的指针。redisContext对象用来维护连接的状态。  
redisContext对象中err字段非0表示连接处于异常状态。   
redisContext对象errstr字段包含错误信息描述字符串。  
使用redisConnect接口连接Redis服务器后，需要判断是否连接成功：
	
	redisContext *c = redisConnect("127.0.0.1", 6379);
	if (c != NULL && c->err) {
    	printf("Error: %s\n", c->errstr);
    	// handle error
	}
	
###发送命令
有多个方法发送命令到Redis。现在介绍redisCommand函数，redisCommand函数类似printf。最简单的形式如下：
	
	reply = redisCommand(context, "SET foo bar");
	
%s标识符用来将字符串格式到命令中。

	reply = redisCommand(context, "SET foo %s", value);

%b标识符可以用来将二进制数据格式化到命令中，valuelen为数据长度。

	reply = redisCommand(context, "SET foo %b", value, (size_t) valuelen);
	
###处理命令结果

`redisCommand`执行成功后，返回一个`redisReply`对象保存执行结果。  
当命令执行异常，redisCommand返回`NULL`,设置`context`对象`err`字段记录错误原因。  
当执行命令异常，`context`失效，必须重新建立到redis服务器的连接。  
`redisReply`对象中的`type`字段标识返回类型：

* REDIS_REPLY_STRING:  
  命令返回值类型为字符串。reply->str代表返回的字符串值。  
  
* REDIS_REPLY_STATUS:  
	命令返回一个状态。通过reply->str获取状态字符串。状态字符串的长度为reply->len。
	
* REDIS_REPLY_ERROR:
	命令执行失败，返回一个错误。错误原因字符串通过reply->str获取。  
	  
* REDIS_REPLY_INTEGER:  
	命令返回一个整型。整型的值可以通过reply->integer字段获取，整型类型为long long。  
	
* REDIS_REPLY_NIL:  
	命令返回一个nil对象。没有数据可以获取。    
	
* REDIS_REPLY_ARRAY:    
	返回值为一个数组。reply->elements中保存对象个数。每个对象都是一个`redisReply`对象。通过reply->element[..index..]获取这些对象。  

`redisReply`对象需要使用`freeReplyObject()`进行释放，返回对象中的子对象无需使用`freeReplyObject()`进行释放。

###管道批量发送命令  
当redisCommand命令被调用时，Hiredis首先根据Redis协议格式化命令。格式化后的命令被放到context维护的输出缓冲区中。输出缓冲区是动态的，可以容纳很多命令。命令被放置到输出缓冲区以后，调用redisGetReply获取返回值。redisGetReply有如下两条执行路径：    

	1. 输入缓冲区非空：  
		从输入缓冲区中解析一个reply应答并返回。
		如果不能解析出一个reply，则走到第二条路径  
		
	2. 输入缓冲区为空：
		将输出缓冲区写到socket。
		读取socket并解析出一个reply。
使用管道批量发送命令，就是使用如下命令填充输出缓冲区：  

	void redisAppendCommand(redisContext *c, const char *format, ...);
	void redisAppendCommandArgv(redisContext *c, int argc, const char **argv, const size_t *argvlen);

调用数次上面函数发送命令后，调用`redisGetReply`获取随后的返回值。  
下面例子是一个简单的使用管道发送命令的例子（只有一次调用write（2）和一次调用read（2））：

	redisReply *reply;
	redisAppendCommand(context,"SET foo bar");
	redisAppendCommand(context,"GET foo");
	redisGetReply(context,&reply); // reply for SET
	freeReplyObject(reply);
	redisGetReply(context,&reply); // reply for GET
	freeReplyObject(reply);  

###示例：

	#include <hiredis/hiredis.h>

	int main()
	{
    	// create a connection
    	redisContext *context = redisConnect("127.0.0.1", 6379);
    	if ( context != NULL && context->err) {
        	printf("connect to redis fail.%s\n", context->errstr);
        	return -1;
    	}

	    redisReply *reply = redisCommand(context, "select 0");
    	if (NULL != reply) {
        	if (REDIS_REPLY_STATUS == reply->type && strcmp(reply->str,"OK") == 0) {
            	printf("command:select 0 suc.type: %d, str:%s\n", REDIS_REPLY_STATUS, "OK");
            } 
            else {
    	        printf("command:select 0 fail.type: %d, str:%s\n", reply->type, reply->str);
            	freeReplyObject(reply);
            	redisFree(context);
            	return -1;
        	}
    	} else {
        	printf("command:select 0 fail.\n");
        	redisFree(context);
        	return -1;
    	}
    	freeReplyObject(reply);

    	char *c1 = "set key1 eeee";
    	reply = redisCommand(context, c1);
    	if (NULL != reply) {
        	if (reply->type == REDIS_REPLY_ERROR) {
	            printf("command:%s excute fail: type: %d, str:%s\n", c1, reply->type, reply->str);
       	 	}
        	printf("command:%s excute ret: type: %d, str:%s\n", c1, reply->type, reply->str);
    	} else {
        	printf("command:%s fail\n", c1);
        	redisFree(context);
	        return -1;
    	}
    	freeReplyObject(reply);

	    char *c2 = "set key2 222";
    	char *c3 = "get list_key1";
	    redisAppendCommand(context, c2);
    	redisAppendCommand(context, c3);
    
    	char *get_cmd = "get key1";
    	reply = redisCommand(context, get_cmd);
    	if (reply != NULL) {
        	if (reply->type == REDIS_REPLY_ERROR) {
            	printf("%s fail.type:%d value:%s\n", get_cmd, reply->type, reply->str);
        	} else if (reply->type == REDIS_REPLY_STRING) {
            	printf("%s suc.type:%d value:%s\n", get_cmd, reply->type, reply->str);
        	}
        	freeReplyObject(reply);
	    } else {
        	printf("%s execute fail.\n", get_cmd);
        	redisFree(context);
        	return -1;
    	}

    // list
    // hash
    char *hgetall = "hgetall hash_key1";
    reply = redisCommand(context, hgetall);
    if (reply != NULL) {
        if (reply->type == REDIS_REPLY_ERROR) {
            printf("%s ,type:%d result:%s\n", hgetall, REDIS_REPLY_ERROR, reply->str);
        } else if (reply->type == REDIS_REPLY_NIL) {
            printf("%s ,type:%d result:%s\n", hgetall, REDIS_REPLY_NIL, reply->str);
        } else if (reply->type == REDIS_REPLY_ARRAY) {
            printf("%s ,type:%d elements:%lu \n", hgetall, REDIS_REPLY_ARRAY, reply->elements);
            int i;
            for (i = 0; i < reply->elements; i++) {
                redisReply *tmp = reply->element[i];
                if (tmp->type == REDIS_REPLY_INTEGER){
                    printf("elements[%d],type:%d result:%lld\n", i, REDIS_REPLY_INTEGER, tmp->integer);
                } else if (tmp->type == REDIS_REPLY_STRING) {
                    printf("elements[%d] ,type:%d result:%s\n", i, REDIS_REPLY_STRING, tmp->str);
                }
                printf("index:%d ,type:%d \n", i, tmp->type);
            }
        }
        freeReplyObject(reply);
    } else {
        printf("%s fail\n", hgetall);
    }
    redisFree(context);
    return 0;
    }
    
    
    
