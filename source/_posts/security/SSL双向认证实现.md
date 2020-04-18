---
title: SSL双向认证实现
comments: false
toc: false
date: 2020-04-18 11:45:50
categories: security
tags:
---

模拟场景：  
Server端和Client端通信，需要进行授权和身份的验证，即Client只能接受Server的消息，Server只能接受Client的消息。

实现技术：  
JSSE（Java Security Socket Extension）是Sun为了解决在Internet上的安全通讯而推出的解决方案。它实现了SSL和TSL（传输层安全）协议。在JSSE中包含了数据加密，服务器验证，消息完整性和客户端验证等技术。通过使用JSSE，开发人员可以在客户机和服务器之间通过TCP/IP协议安全地传输数据

为了实现消息认证  
Server需要：  
1. KeyStore: 其中保存服务端的私钥
2. Trust KeyStore:其中保存客户端的授权证书

同样，Client需要：  
1. KeyStore：其中保存客户端的私钥
2. Trust KeyStore：其中保存服务端的授权证书

## keytool制作双向认证

### 创建服务端证书

1. 生成服务端私钥，并且导入到服务端KeyStore文件中
`keytool -genkeypair -alias serverkey -keystore kserver.keystore`

2. 根据私钥，导出服务端证书
`keytool -export -alias serverkey -keystore kserver.keystore -file server.crt`
server.crt就是服务端的证书

3. 将服务端证书，导入到客户端的Trust KeyStore中
`keytool -import -alias serverkey -file server.crt -keystore tclient.keystore`
tclient.keystore是给客户端用的，其中保存着受信任的证书

### 创建客户端证书

采用同样的方法，生成客户端的私钥，客户端的证书，并且导入到服务端的Trust KeyStore中<br/>

``` shell
keytool -genkey -alias clientkey -keystore kclient.keystore
keytool -export -alias clientkey -keystore kclient.keystore -file client.crt
keytool -import -alias clientkey -file client.crt -keystore tserver.keystore
```

如此一来，生成的文件分成两组  
服务端保存：kserver.keystore tserver.keystore  
客户端保存：kclient.keystore  tclient.kyestore  

## openssl制作双向认证

### 创建根证书

1. 生成根证书私钥
`openssl genrsa -des3 -out root.key 1024`

2. 生成根证书签名请求文件
`openssl req -new -out root-req.csr -key root.key -keyform PEM`
> 证书签名请求文件CSR(Certificate Signing Request)

3. 自签根证书
`openssl x509 -req -in root-req.csr -out root-cert.crt -signkey root.key -CAcreateserial -days 365`
> csr文件交给CA签名后形成自己的证书(与根证书私钥对应)

4. 导出p12格式证书
`openssl pkcs12 -export -clcerts -in root-cert.crt -inkey root.key -out root.p12`

### 创建服务端证书

1. 生成服务端key
`openssl genrsa -des3 -out server-key.key 1024`

2. 生成服务端请求文件
`openssl req -new -out server-req.csr -key server-key.key`

3. 生成服务端证书（root证书，rootkey，客户端key，客户端请求文件这4个生成客户端证书）
`openssl x509 -req -in server-req.csr -out server-cert.crt -signkey server-key.key -CA root-cert.crt -CAkey root.key -CAcreateserial -days 365`

4. 生成服务端p12格式根证书
`openssl pkcs12 -export -clcerts -in server-cert.crt -inkey server-key.key -out server.p12`

### 创建客户端证书

1. 生成客户端key
`openssl genrsa -des3 -out client-key.key 1024`

2. 生成客户端请求文件
`openssl req -new -out client-req.csr -key client-key.key`

3. 生成客户端证书（root证书，rootkey，客户端key，客户端请求文件这4个生成客户端证书）
`openssl x509 -req -in client-req.csr -out client-cert.crt -signkey client-key.key -CA root-cert.crt -CAkey root.key -CAcreateserial -days 365`

4. 生成客户端p12格式根证书
`openssl pkcs12 -export -clcerts -in client-cert.crt -inkey client-key.key -out client.p12`

如此一来，生成的文件分成两组  
服务端保存：trustStore使用root.p12，keyStore使用server.p12  
客户端保存：trustStore使用root.p12,keyStore使用client.p12  

**根证书作为CA,用来验证对方发送的签名是否正确。如果不对签名进行验证，则不需要根证书放到trustStore中**

## JAVA实现

采用JSSE及keytool生成的证书，分别生成SSLServerSocket,SSLSocket  
client采用kclient.keystore中的clientkey私钥进行数据加密，发送给server  
server采用tserver.keystore中的client.crt证书（包含了clientkey的公钥）对数据解密，如果解密成功，证明消息来自client，进行逻辑处理  
server采用kserver.keystore中的serverkey私钥进行数据加密，发送给client  
client采用tclient.keystore中的server.crt证书（包含了serverkey的公钥）对数据解密，如果解密成功，证明消息来自server，进行逻辑处理  
如果过程中，解密失败，那么证明消息来源错误。不进行逻辑处理。这样就完成了双向的身份认证。

服务端，生成SSLServerSocket代码

``` java
SSLContext ctx = SSLContext.getInstance("SSL");

KeyManagerFactory kmf = KeyManagerFactory.getInstance("SunX509");
TrustManagerFactory tmf = TrustManagerFactory.getInstance("SunX509");

KeyStore ks = KeyStore.getInstance("JKS");
KeyStore tks = KeyStore.getInstance("JKS");

ks.load(new FileInputStream(path.resolve("kserver.keystore").toFile()), SERVER_KEY_STORE_PASSWORD.toCharArray());
tks.load(new FileInputStream(path.resolve("tserver.keystore").toFile()), SERVER_TRUST_KEY_STORE_PASSWORD.toCharArray());

kmf.init(ks, SERVER_KEY_STORE_PASSWORD.toCharArray());
tmf.init(tks);

ctx.init(kmf.getKeyManagers(), tmf.getTrustManagers(), null);

serverSocket = (SSLServerSocket) ctx.getServerSocketFactory().createServerSocket(DEFAULT_PORT);
serverSocket.setUseClientMode(false);
serverSocket.setNeedClientAuth(true);  //需要验证客户端的身份
```

客户端，生成SSLSocket的代码，大同小异

``` java
SSLContext ctx = SSLContext.getInstance("SSL");

KeyManagerFactory kmf = KeyManagerFactory.getInstance("SunX509");
TrustManagerFactory tmf = TrustManagerFactory.getInstance("SunX509");

KeyStore ks = KeyStore.getInstance("JKS");
KeyStore tks = KeyStore.getInstance("JKS");

ks.load(new FileInputStream(path.resolve("kclient.keystore").toFile()), CLIENT_KEY_STORE_PASSWORD.toCharArray());
tks.load(new FileInputStream(path.resolve("tclient.keystore").toFile()), CLIENT_TRUST_KEY_STORE_PASSWORD.toCharArray());

kmf.init(ks, CLIENT_KEY_STORE_PASSWORD.toCharArray());
tmf.init(tks);

ctx.init(kmf.getKeyManagers(), tmf.getTrustManagers(), null);

sslSocket = (SSLSocket) ctx.getSocketFactory().createSocket(DEFAULT_HOST, DEFAULT_PORT);
/*配置套接字在握手时使用客户机（或服务器）模式。
此方法必须在发生任何握手之前调用。一旦握手开始，在此套接字的生存期内将不能再重置模式。
服务器通常会验证本身，客户机则不要求这么做
如果套接字应该以“客户机”模式开始它的握手，此参数为 true
*/
sslSocket.setUseClientMode(true);
```






