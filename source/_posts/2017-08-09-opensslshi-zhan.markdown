---
layout: post
title: "openssl实战"
date: 2017-08-09 17:50:42 +0800
comments: true
categories: ops
tags: openssl tls
---
本篇文章主要是对openssl的使用进行实践总结。  

* [生成密钥](#key)
* [创建证书](#crt)
* [实践](#practice)

<!-- more -->

## <span id="key">生成密钥</span>
### 生成RSA密钥  
命令：  
```
openssl genrsa [-out filename] [-passout arg] [-aes128] [-aes128] [-aes192] [-aes256] [-camellia128] [-camellia192] [-camellia256] [-aes192]
       [-aes256] [-camellia128] [-camellia192] [-camellia256] [-des] [-des3] [-idea] [-f4] [-3] [-rand file(s)] [-engine id] [numbits]
```
参数：  
`-out filename` 输出到结果文件。如果不指定，则输出到标准输出。  
`-aes128|-aes192|-aes256|-camellia128|-camellia192|-camellia256|-des|-des3|-idea`   
指定加密算法加密密钥。  
`numbits`指定密钥位数。默认为512位    

生成一个2048位rsa非对称密钥，保存到rsa-fd.key文件中：  
	
	openssl genrsa -out rsa-fd.key 2048

`rsa-fd.key`文件为`PEM`格式：  

	$ file rsa-fd.key
	rsa-fd.key: PEM RSA private key
	
使用`rsa`命令解析出私钥结构：  

	openssl rsa -text -in rsa-fd.key
	
查看私钥中的公开部分：  
	
	openssl rsa -in rsa-fd.key -pubout -out rsa-pub.key

###生成DSA密钥
DSA的密钥生成分成两个部分:先生成DSA的参数，然后再生成密钥。
命令：

	openssl dsaparam -genkey 2048 | openssl dsa -out dsa.key
	
###生成ECDSA密钥
创建ECDSA密钥的过程是类似的，但是不能创建任意长度的密钥。对于每个密钥，你需要选择一个命名曲线(named curve)，它可以控制密钥长度，同时也限定了椭圆曲线的参数。下面的例子使用secp256r1这个命名曲线创建一个256位长度的ECDSA密钥  

	openssl ecparam -genkey -name secp256r1 | openssl ec -out ec.key

##<span id="crt">创建证书</span>
###创建证书申请
> 使用openssl req命令交互式生成创建证书申请：  

```
$ openssl req -new -key rsa-fd.key -out rsa-fd.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:ShangHai
Locality Name (eg, city) []:ShangHai
Organization Name (eg, company) [Internet Widgits Pty Ltd]:limo
Organizational Unit Name (eg, section) []:limo
Common Name (e.g. server FQDN or YOUR name) []:localhost
Email Address []:.

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:.
An optional company name []:.
$
```
其中`Common Name`指定证书保护的域名。  
> 使用openssl req查看证书申请内容：  

```
Data:
        Version: 0 (0x0)
        Subject: C=CN, ST=ShangHai, L=ShangHai, O=limo, OU=limo, CN=localhost
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
            RSA Public Key: (2048 bit)
                Modulus (2048 bit):
                    00:c0:2b:11:a0:53:7d:fa:83:7f:29:2b:b5:dc:bc:
                Exponent: 65537 (0x10001)
        Attributes:
            a0:00
Signature Algorithm: sha1WithRSAEncryption
        7c:3d:47:49:d5:f7:c9:89:65:81:62:3d:bb:84:b8:ed:70:c4:
        e8:ba:ae:03:2e:af:3a:cb:31:07:e3:ef:57:a1:c7:a8:a6:c8:
```

> 使用配置文件生成证书：  

配置文件内容如下：

```
[req]
prompt = no
distinguished_name = dn
req_extensions     = ext

[dn]
CN      = localhost
O       = limo
L       = ShangHai
C       = CN

[ext]
subjectAltName = DNS:www.localhost.com,DNS:localhost
```
执行命令如下：

	$ openssl req -new -config rsa-fd.conf -key rsa-fd.key -out t.csr

###自签名证书
如果已经有了csr，则执行以下命令创建证书：  

	$ openssl x509 -req -days 365 -in rsa-fd.csr -signkey rsa-fd.key -out rsa-fd.crt
	Signature ok
	subject=/C=CN/ST=ShangHai/L=ShangHai/O=limo/OU=limo/CN=localhost
	Getting Private key

##<span id="practice"> 实践 </span>

> 创建根CA证书  

1. 创建rsa密钥

	```
	$ openssl genrsa -out ca.key 2048  
	Generating RSA private key, 2048 bit long modulus  
	........+++  
	..................................................+++  
	e is 65537 (0x10001)  
	```
2. 创建证书申请
	
		$ openssl genrsa -out ca.key 2048 -out ca.csr
		$ 

3. 创建自签名证书
		
		$ openssl x509 -req -in ca.csr -signkey ca.key 	-out ca.crt
		Signature ok
		subject=/C=CN/ST=ShangHai/L=ShangHai/O=li/OU=li/CN=RootCA
		Getting Private key

> 创建client密钥和证书

1. 创建rsa密钥  
	
	```
	$ openssl genrsa -out client.key 2048
Generating RSA private key, 2048 bit long modulus
..................................+++
..........+++
e is 65537 (0x10001)
	```

2. 创建证书申请
	
		openssl req -new -key client.key -out client.csr
		
3. 使用root CA签名client证书
	
		$ openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt
		Signature ok
		subject=/C=CN/ST=ShangHai/L=SH/O=li/OU=li/CN=localhost	
		Getting CA Private Key
需要增加`-CAcreateserial`参数否则报错

> 创建server密钥和证书

方法同生成client密钥证书类似 

> 使用Go创建https server，并对客户端进行双向验证：  

```
func HelloServer(w http.ResponseWriter, req *http.Request) {
	w.Header().Set("Content-Type", "text/plain")
	w.Write([]byte("This is an example server.\n"))
}

func main() {
	pool := x509.NewCertPool()
	caPath := "ca.crt"
	caCrt, err := ioutil.ReadFile(caPath)
	if err != nil {
		fmt.Println("ReadFile err ", err)
		return
	}
	ok := pool.AppendCertsFromPEM(caCrt);
	if !ok {
		fmt.Println("append err")
		return
	}

	fmt.Println(ok)
	tlsConfig := &tls.Config{
		ClientCAs:pool,
		ClientAuth: tls.RequireAndVerifyClientCert,
	}
	server := http.Server{
		Addr:      ":443",
		TLSConfig: tlsConfig,
	}
	http.HandleFunc("/hello", HelloServer)
	err = server.ListenAndServeTLS("server.crt", "server.key")
	log.Fatal(err)
}
```
> 使用Go创建https client

```
func main() {
	pool := x509.NewCertPool()
	caPath := "ca.crt"
	caCrt, err := ioutil.ReadFile(caPath)
	if err != nil {
		fmt.Println("ReadFile err:", err)
		return
	}
	pool.AppendCertsFromPEM(caCrt)
	cliCrt, err := tls.LoadX509KeyPair("client.crt", "client.key")
	if err != nil {
		fmt.Println("Loadx509keypair err:", err)
		return
	}
	tlsConfig := &tls.Config{
		RootCAs:      pool,
		Certificates: []tls.Certificate{cliCrt},
		InsecureSkipVerify:true,
	}
	tr := http.Transport{TLSClientConfig: tlsConfig}
	client := http.Client{Transport: &tr}
	url := "https://localhost/hello"
	resp, err := client.Get(url)
	if err != nil {
		panic(err)
	}
	data, err := ioutil.ReadAll(resp.Body)
	fmt.Println(string(data))
}
```
[code](https://github.com/jintao-zero/go-practice/tree/master/https)
## 参考
1. [https权威指南]()
2. [Go和HTTPS](http://tonybai.com/2015/04/30/go-and-https/)
