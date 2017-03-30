---
layout: post
title: "golang 实现HTTP代理"
date: 2017-03-29 16:51:16 +0800
comments: true
categories: network
tags: Golang HTTP 
---
使用Golang实现一个HTTP代理程序，程序部署在海外VPS节点，可以实现翻墙功能  
HTTP Proxy的基本原理是将客户端HTTP请求发送到目标主机，将目标主机返回的HTTP应答转发到客户端,对于HTTPS协议略有不同，HTTPS协议需要Proxy先建立一条到目标主机的链路，之后再进行协议本身的通讯，下面简单介绍一个HTTP Proxy的实现过程：    

###分析第一条请求
1. 启动一个简版HTTP Proxy程序，代码如下：
	
		func handleConn(conn net.Conn) {
			defer conn.Close()
			log.Println("client conn form ", conn.RemoteAddr())
			var buffer [1024]byte
			_, err := conn.Read(buffer[:])
			if err != nil {
				return
			}
			log.Println(string(buffer[:]))
		}

		func main() {
			// listen on tcp port
			l, err := net.Listen("tcp", ":8081")
			if err != nil {
				log.Println(err)
				return
			}
			for {
				conn, err := l.Accept()
				if err != nil {
					log.Println(err)
					return
				}
				go handleConn(conn)
			}
		}  
		go run httpproxy.go
	这个程序主要是用来接收客户端发送的第一条HTTP请求

<!-- more -->

2. Safari设置HTTP/HTTPS代理  
	设置代理为`localhost:8081`  
	![proxy](/images/http-proxy-1.png)
		
3. Safari输入框访问`http://localhost:8080`，我们的proxy程序输出HTTP请求：  
	
		GET http://www.localhost.com:8080/ HTTP/1.1
		Host: www.localhost.com:8080
		Proxy-Connection: keep-alive
		Upgrade-Insecure-Requests: 1
		Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
		User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_3) AppleWebKit/602.4.8 (KHTML, like Gecko) 	Version/10.0.3 Safari/602.4.8
		Accept-Language: zh-cn
		Accept-Encoding: gzip, deflate
		Connection: keep-alive
其中跟代理有关的是前两行：  
	
		GET http://www.localhost.com:8080/ HTTP/1.1
		Host: www.localhost.com:8080
当访问`http://localhost`时，输出的前两行为：  
		
		GET http://www.localhost.com/ HTTP/1.1
		Host: www.localhost.com
		
4. Safari输入框访问`https://localhost:8080`时，proxy程序输出HTTP请求为：  
	
		CONNECT www.localhost.com:8080 HTTP/1.1
		Host: www.localhost.com:8080
		User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_3) AppleWebKit/602.4.8 (KHTML, like Gecko) Version/10.0.3 Safari/602.4.8
		Connection: keep-alive
		Proxy-Connection: keep-alive
		
   访问网址为`https://localhost`时，输出为：  
	
		CONNECT www.localhost.com:443 HTTP/1.1
		Host: www.localhost.com
		User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_3) AppleWebKit/602.4.8 (KHTML, like Gecko) Version/10.0.3 Safari/602.4.8
		Connection: keep-alive
		Proxy-Connection: keep-alive

设置代理后，客户端发送HTTP请求之前需要先与Proxy通讯建立到目标主机的通讯隧道，命令消息格式为：  
	
	method absoluteURI HTTP/1.1

使用`HTTP`协议时，使用`GET`命令发送请求，使用`HTTPS`协议时，使用`CONNECT`命令发送请求,后面跟着[absoluteURI](http://greenbytes.de/tech/webdav/rfc2616.html#general.syntax),再就是HTTP版本信息
第二行为`Host`头标示了需要访问的目标地址，根据[`HTTP/1.1`]([Request-URI](http://greenbytes.de/tech/webdav/rfc2616.html#request-uri))版本规定，客户端和服务的必须支持`Host`头，但是默认协议情况下，host头只包含域名，不包含默认端口号(`HTTP(80)``HTTPS(443)`)

我们的proxy程序需要根据`absoluteURI`获取目标域名和端口，从而建立到目标主机的连接，示例代码如下：  
	
	var buffer [1024]byte
	n, err := conn.Read(buffer[:])
	if err != nil {
		return
	}
	log.Println(string(buffer[:]))
	_, line, err := bufio.ScanLines(buffer[:], true)
	if err != nil {
		log.Println(err)
		return
	}
	var method string
	var absUri string
	_, err = fmt.Sscanf(string(line), "%s%s", &method, &absUri)
	if err != nil {
		log.Println(err)
		return
	}
	absUrl, err := url.Parse(absUri)
	if err != nil {
		log.Println(err)
		return
	}
	log.Printf("%+v", *absUrl)
	var hostPort string
	if absUrl.Scheme == "http" {
		if strings.Index(absUrl.Host, ":") == -1 {
			hostPort = absUrl.Host + ":80"
		} else {
			hostPort = absUrl.Host
		}
	} else {
		hostPort = absUrl.Scheme + ":" + absUrl.Opaque
	}
	log.Println("hostPort", hostPort)

### 建立连接
1. 根据获取的目标地址+端口建立到目标主机的tcp连接
2. 根据第一条请求的协议类型(HTTP/HTTPS),做不同应答处理：  
	2.1 HTTP消息，直接转发给目标主机，不对客户端有应答  
	2.2 HTTPS CONNECT消息，对客户端应答`200`消息：
		
		HTTP/1.1 200 Connection established\r\n\r\n
3. 不间断进行消息转发，某一方连接异常后，关闭双方连接
	
		go io.Copy(conn, serverConn)
		io.Copy(serverConn, conn)

### 部署
1. [完整代码](https://github.com/jintao-zero/golang-http-proxy)
2. 编译生成二进制可执行文件，部署到海外VPS服务器，浏览器设置对应ip和端口后，即可翻墙 

### 遗留问题
1、转发功能基本实现，但是依然无法访问google、facebook等被GFW墙的网站
	我猜想是明文HTTP请求中`google、facebook`类的网站名被匹配到，出口连接被关闭，使用了ssh tunnel功能，将数据加密传输解决了这个问题

###参考
1. [HTTP1.0 VS HTTP1.1](http://greenbytes.de/tech/webdav/rfc2616.html#changes.from.1.0)
2. [how does http proxy work?](http://stackoverflow.com/questions/7155529/how-does-http-proxy-work)
3. [Proxy server](https://en.wikipedia.org/wiki/Proxy_server#Web_proxy_servers)