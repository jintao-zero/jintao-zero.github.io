---
layout: post
title: "[翻译]SOCKS5 协议"
date: 2017-04-02 16:38:00 +0800
comments: true
categories: network
tags: SOCKS  
---
##介绍
网络防火墙用来隔离组织内部网络和外部网络，比如`INTERNET`。这些防火墙系统扮演着内外部网络之间的应用层网关，用来控制`TELNET`、`FTP`和`SMTP`等协议的进入。随着更多复杂应用层协议被设计用来加速全局信息发现，需要为这些协议提供一个框架以便透明和安全的通过防火墙。 也需要提供为这种通过提供可靠授权。   
`SOCKS5`协议被设计用来为基于TCP或者UDP协议的client-server模式应用程序方便且安全的使用网络防火墙服务。这个协议概念上认为是在应用层和传输层之间，因此它不提供网络层网关服务，比如转发`ICMP`消息。  

已经存在了一个`SOCKS4`协议，这个版本的协议在4版本的基础上支持UDP协议，包含通用授权范型和支持域名和V6版本IP地址。  

<!-- more -->

## TCP客户端处理步骤
当一个基于TCP协议的客户端希望与位于防火墙后面的目标之间建立一个连接时，它必须打开一个到`SOCKS`服务端的连接。`SOCKS`服务按照惯例位于1080TCP端口。 如果连接建立成功后，客户端发送鉴权请求。服务端评估请求，要么建立相应连接或者拒绝这个请求。    
客户端连接socks服务端，发送一个版本标识符或者方法选择消息：  
	
	+----+----------+----------+
	|VER | NMETHODS | METHODS  |
    +----+----------+----------+
    | 1  |    1     | 1 to 255 |
    +----+----------+----------+
`VER`字段设置为`X'05'`是这个协议的版本。`NMETHODS`字段包含出现在`METHODS`字段中的值的字节数。  
socks服务端从`METHODS`中选择一个方法，发送一个`METHOD`选择消息：  
	
	+----+--------+
    |VER | METHOD |
    +----+--------+
    | 1  |   1    |
    +----+--------+

如果选择的`METHOD`为`X'FF'`，那么客户端列出的所有方法都不被接受，这时候客户端必须关闭连接。  
目前定义的`METHOD`值有：  

	o  X'00' NO AUTHENTICATION REQUIRED
    o  X'01' GSSAPI
    o  X'02' USERNAME/PASSWORD
    o  X'03' to X'7F' IANA ASSIGNED
    o  X'80' to X'FE' RESERVED FOR PRIVATE METHODS
    o  X'FF' NO ACCEPTABLE METHODS  
    
方法选择之后，客户端和服务端之间进入与方法相关的子协商。  

## 请求  
子协商完成后，客户端发送请求详情。如果协商结果是需要对发送的消息进行封装加密，那么请求消息必须进行封装加密。  
SOCKS协议请求格式为：  
	
	+----+-----+-------+------+----------+----------+
    |VER | CMD |  RSV  | ATYP | DST.ADDR | DST.PORT |
    +----+-----+-------+------+----------+----------+
    | 1  |  1  | X'00' |  1   | Variable |    2     |
    +----+-----+-------+------+----------+----------+
字段说明如下：  
	
	o  VER    protocol version: X'05'
    o  CMD
    o  CONNECT X'01'
    o  BIND X'02'
    o  UDP ASSOCIATE X'03'
    o  RSV    RESERVED
    o  ATYP   address type of following address
    o  IP V4 address: X'01'
    o  DOMAINNAME: X'03'
    o  IP V6 address: X'04'
    o  DST.ADDR       desired destination address
    o  DST.PORT desired destination port in network octet order
SOCKS服务端将会根据源和目的地址评估请求，并且根据请求类型返回一个或者多个回复消息。  

##地址
在地址字段（`DST.ADDR BND.ADDR`），`ATYP`字段指示了地址的类型：    

* X'01'
指示地址为IPv4地址，长度为4个字节。  
* X'03'
指示地址为域名。地址的第一个字节为域名的长度，域名不是以`NUL`结束的。  
* X'04'
地址为IPv6地址，长度为16个字节。  

##回复  
对应上面的请求，服务端回复以下应答：  

	+----+-----+-------+------+----------+----------+
    |VER | REP |  RSV  | ATYP | BND.ADDR | BND.PORT |
    +----+-----+-------+------+----------+----------+
    | 1  |  1  | X'00' |  1   | Variable |    2     |
    +----+-----+-------+------+----------+----------+ 
 字段说明如下：  
 
 	 o  VER    protocol version: X'05'
     o  REP    Reply field:
        o  X'00' succeeded
        o  X'01' general SOCKS server failure
        o  X'02' connection not allowed by ruleset
        o  X'03' Network unreachable
        o  X'04' Host unreachable
        o  X'05' Connection refused
        o  X'06' TTL expired
        o  X'07' Command not supported
        o  X'08' Address type not supported
        o  X'09' to X'FF' unassigned
        
     o  RSV    RESERVED
     o  ATYP   address type of following address
	     o  IP V4 address: X'01'
         o  DOMAINNAME: X'03'
         o  IP V6 address: X'04'
     o  BND.ADDR       server bound address
     o  BND.PORT       server bound port in network octet order

标记为`RESERVED`的字段必须设置为X'00'
### CONNECT
对于`CONNECT`请求消息的应答中，`BND.PORT`字段为服务端绑定的用来连接到目标主机的端口号，`BND.ADDR`字段为绑定的地址。绑定的地址通常与客户端连接服务端的地址不同。

### BIND
BIND请求被用来要求客户端接受来自服务端的请求。FTP是一个常见的例子，它使用客户端到服务端的连接发送命令和状态，使用服务端到客户端的连接传输数据。    
客户端使用`CONNECT`建立主连接，使用`BIND`请求只是用来建立第二个连接。  
对于`BIND`请求，`SOCKS`服务端将使用`DST.ADDR`和`DST.PORT`。  
对于`BIND`操作，SOCKS服务端将会发送两个回复到客户端。服务端创建和绑定一个新的socket套接字后发送第一个回复。回复中`BND.PORT`字段值为SOCKS服务端本地绑定用来监听和接受连接的端口号。`BND.ADDR`字段为SOCKS服务端绑定的IP。客户端通过主控制链路发送这些信息给应用程序服务端。当新的连接建立成功或者失败后，SOCKS服务端会发送第二个应答。 在第二个应答中，`BND.PORT`和`BND.ADDR`字段包含连接主机的ip和端口信息。  

### UDP
`UDP ASSOCIATE`请求用来与UDP中继进程建立一个联系来处理UDP数据报。客户端希望发送UDP数据报到`DST.ADDR`和`DST.PORT`字段标示的IP和端口号。在建立UDP ASSOCIATE时，如果客户端不拥有这些信息，客户端必须使用全零的端口号和ip地址。    
当发送UDP ASSOCIATE请求的TCP连接中断时，这个UDP 连接也要中断。  
在对UDP ASSOCIATE请求的应答中，`BND.PORT`和`BND.ADDR`字段表示客户端必须将待转发的UDP消息发送到这个地址。  

### 应答处理  
如果应答消息中结果字段为失败，SOCKS服务端必须在发送应答后立即关闭TCP连接。关闭连接必须在检测到失败后10秒内终止连接。    

如果应答消息中结果字段为成功，请求为`BIND`或者`CONNECT`的话，客户端就可以开始传输数据。 如果协商了封装加密方法的话，在客户端和SOCKS服务端通信过程中需要进行封装加密。  

## UDP客户端
udp客户端必须想UDP ASSOCIATE应答中`BND.PORT`标示的字段发送UDP数据报。如果协商了封装方法，那么通讯时需要进行封装。  
每个UDP数据需要携带一个UDP请求头：  

	+----+------+------+----------+----------+----------+
    |RSV | FRAG | ATYP | DST.ADDR | DST.PORT |   DATA   |
    +----+------+------+----------+----------+----------+
    | 2  |  1   |  1   | Variable |    2     | Variable |
    +----+------+------+----------+----------+----------+
   
 字段说明如下：  
 
	 o  RSV  Reserved X'0000'
     o  FRAG    Current fragment number
     o  ATYP    address type of following addresses:
     		o  IP V4 address: X'01'
          o  DOMAINNAME: X'03'
          o  IP V6 address: X'04'
     o  DST.ADDR       desired destination address
     o  DST.PORT       desired destination port
     o  DATA     user data

UDP中继服务端必须从SOCKS服务端获取申请建立UDP ASSOCIATE的客户端ip地址。中继服务端对于其他源ip发送过来的数据包将会丢弃。  


##参考 

[rfc1928](https://www.ietf.org/rfc/rfc1928.txt)  
[rfc1929](https://tools.ietf.org/html/rfc1929)

