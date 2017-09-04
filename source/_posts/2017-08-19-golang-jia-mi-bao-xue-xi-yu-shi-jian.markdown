---
layout: post
title: "golang 加密包学习与实践"
date: 2017-08-19 10:57:59 +0800
comments: true
categories: dev
tags: crypt golang
---
Go标准库[crypto](https://golang.org/pkg/crypto/)提供了多种加解密算法、摘要算法。本文主要是对加密包中各个算法接口进行学习实践的总结。  

* [aes](#aes)

* [rsa](#rsa)

<!-- more -->

## <span id="aes">aes</span>

## <span id="rsa">rsa</span>
[rsa](https://golang.org/pkg/crypto/rsa/)包实现了[RSA](https://en.wikipedia.org/wiki/RSA_(cryptosystem))加密算法，关于`RSA`的标准为[PKCS #1](https://en.wikipedia.org/wiki/PKCS_1)。  
`RSA`既可以用与加解密也可以用与签名。  
使用RSA进行加密和签名的最初标准是`PKCS#1`，`RSA encryption`和`RSA signatures`默认参照`PKCS#1`1.5版本。但是这个版本的标注存在缺陷，新的的设计应该参照第二版本，以`OAEP`和`PSS`称呼。  

`rsa`包中的相关操作不是常量时间复杂度。   
对使用包中进行加解密操作和签名操作进行说明：  
> 生成key
	
	func GenerateKey(random io.Reader, bits int) (*PrivateKey, error)

`random`为随机数源，比如`crypto/rand.Reader`   
`bits`为密钥位数，目前建议是2048位  

> 加密  

	func EncryptOAEP(hash hash.Hash, random io.Reader, pub *PublicKey, msg []byte, label []byte) ([]byte, error)
`EncryptOAEP`函数对消息体进行加密。  
`hash`参数是一个哈希函数。对同一消息进行加解密必须使用同一种哈希函数，`sha256.New()`是一个合理选择。  
`random`参数为随机数源，用来保证对同一个消息体进行两次加密时的输出不同。  
`label`参数可以包含任意数据，这些数据不会被加密，标签参数给消息增加了重要的上下文信息。加解密时传入的`label`标签需要一致。如果不需要，可以为空。
`msg`待加密消息长度必须小于等于公钥长度减去2倍哈希长度。  

> 解密
	
	func DecryptOAEP(hash hash.Hash, random io.Reader, priv *PrivateKey, ciphertext []byte, label []byte) ([]byte, error)

`hash`加解密使用的哈希函数必须一致，`sha256.New()`是一个合理选择。  
`random`随机数源。  
`label`参数值必须与加密时传入的值一致。  

> 加解密实践

```
func TestCrypt() {
    //create private key
    privateKey, err := rsa.GenerateKey(rand.Reader,2048)
    if err != nil {
        log.Fatal("generateKey err ",err)
        return
    }
    // encrypt
    ciphertext, err := rsa.EncryptOAEP(sha256.New(), rand.Reader, &privateKey.PublicKey, []byte("hahaha"), []byte("label"))
    if err != nil {
        log.Fatal("encrypt err ", err)
        return
    }
    log.Printf("Ciphertext: %x\n", ciphertext)
    log.Println(hex.EncodeToString(ciphertext))
    plaintext, err := rsa.DecryptOAEP(sha256.New(), rand.Reader, privateKey, ciphertext, []byte("label"))
    if err != nil || bytes.Compare([]byte("hahaha"), plaintext) != 0 {
        log.Fatal("decrypt err: ", err)
        return
    }
    log.Println("plaintext :", string(plaintext))
}
```

> 签名  

	func SignPSS(rand io.Reader, priv *PrivateKey, hash crypto.Hash, hashed []byte, opts *PSSOptions) ([]byte, error)

`rand`为随机数源。  
`priv`为进行签名用到的私钥。  
`SignPSS`函数依据`RSASSA-PSS`标准计算摘要数据的签名。注意：摘要数据必须是使用指定哈希函数对输入数据计算得出的摘要值。`opts`参数可以为空，此时使用默认值。  
> 验证签名  

	func VerifyPSS(pub *PublicKey, hash crypto.Hash, hashed []byte, sig []byte, opts *PSSOptions) error

VerifyPSS验证PSS签名。`hashed`是使用指定函数对输入数据计算得出的摘要值。sig为签名数据。opts可以为空。  

> 签名实践

```
func TestSign() {
    //create private key
    privateKey, err := rsa.GenerateKey(rand.Reader,2048)
    if err != nil {
        log.Fatal("generateKey err ",err)
        return
    }
    // sign
    hashed := sha256.Sum256([]byte("hahaha"))
    /*
    opts := rsa.PSSOptions{
       SaltLength:10,
        Hash:crypto.SHA256,
    }
    */
    sig, err := rsa.SignPSS(rand.Reader,privateKey, crypto.SHA256,  hashed[:], nil)
    if err != nil {
        log.Fatal("SignPSS err ", err)
        return
    }
    log.Println(hex.EncodeToString(sig))
    err = rsa.VerifyPSS(&privateKey.PublicKey, crypto.SHA256, hashed[:], sig, nil)
    if err != nil {
        log.Fatal("VerifyPSS err", err)
        return
    }
    log.Println("VerifyPSS suc")
}
```
 