---
title: SSL双向认证实现
comments: false
toc: false
date: 2020-04-18 11:45:50
categories: security
tags:
---

用SSL进行双向身份验证意思就是在客户机连接服务器时，连接双方都要对彼此的数字证书进行验证，保证这是经过授权的才能够连接。

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
只导出证书不导出秘钥`openssl pkcs12 -export -nokeys -cacerts -in root-cert.crt -inkey root.key -out root.p12`


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

## keytool与openssl区别

&emsp;&emsp;JDK里面内置了一个数字证书生产工具:keytool。但是这个工具只能生成自签名的数字证书。所谓自签名就是指证书只能保证自己是完整的，没有经过非法修改的。但是无法保证这个证书是属于谁的（一句话:**keytool没办法签发证书，而openssl能够进行签发和证书链的管理**）。用这种自签名的证书也是可以进行双向验证的但是**这种验证有一个缺点:对于每一个要连接的服务器，都要保存一个证书的验证副本。而且一旦服务器更换证书，所有客户端就需要重新部署这些副本。**对于比较大型的应用来说，这一点是不可接受的。所以就需要证书链进行双向认证。
> 证书链是指对证书的签名有一个预先部署的，众所周知的签名方签名完成，这样每次需要验证证书时只要用这个公用的签名方的公钥进行验证就可以了

&emsp;&emsp;我们使用的浏览器就保存了几个常用的CA_ROOT,每次连接到网站时只要这个网站的证书是经过这些CA_ROOT签名过的,就可以通过验证了。但是这些共用的CA_ROOT的服务不是免费的,而且价格不菲。所以我们可以自己生成一个CA_ROOT的密钥对(OpenSSL)，然后部署应用时，只要把这个CA_ROOT的私钥部署在所有节点就可以完成验证了。

## 实现web项目的ssl双向认证

### openssl生成CA_ROOT

1. 生成根证书私钥
`openssl genrsa -out ca_root.pem 1024`
2. 生成根证书签名请求文件
`openssl req -new -out ca_root.csr -key ca_root.pem`
3. 自签署证书
`openssl x509 -req -in ca_root.csr -out ca_root-cert.pem -signkey ca_root.pem -days 365`
4. 导出ca证书(只导出证书未导出秘钥)
`openssl pkcs12 -export -nokeys -cacerts -in ca_root-cert.pem -inkey ca_root.pem -out ca_root.p12`

### 注册服务端证书

1. 创建服务端密钥库,别名为server，validity有效期为365天，密钥算法为RSA， storepass密钥库密码，keypass别名条目密码。
`keytool -genkey -alias server -validity 365 -keyalg RSA -keypass 123456 -storepass 123456 -keystore server.jks` 
2. 生成服务端证书请求文件
`keytool -certreq -alias server -sigalgMD5withRSA -file server.csr -keypass 123456 -keystore server.jks -storepass 123456`
3. 使用CA_ROOT签证生成服务端证书
`openssl x509 -req -in server.csr -out server.pem -CA ca_root.pem -CAkey ca_root-cert.pem -days 365`
4. 服务端信任CA_ROOT(CA_ROOT证书导入服务端密钥库)
`keytool -import -v -trustcacerts -keypass 123456 -storepass 123456 -alias root -file ca_root-cert.pem -keystore server.jks`
5. 将服务端证书导入服务端密钥库中
`keytool -import -v -trustcacerts -storepass 123456 -alias server -file server.pem -keystore server.jks`
6. 使用CA_ROOT证书生成服务端信任库
`keytool -import -alias servertrust -file ca_root-cert.pem -keystore servertrust.jks`

### 注册客户端证书

1. 创建客户端密钥(指定用户名，下列命令中的user将替换为颁发证书的用户名)
`openssl genrsa -out user-key.pem 1024`
2. 生成客户端证书请求文件
`openssl req -new -out user-req.csr-key user-key.pem`
3. 使用CA_ROOT签证生成对应用户名的客户端证书
`openssl x509 -req -in user-req.csr -out user-cert.pem -signkey user-key.pem -CA ca_root.pem -CAkey ca_root-cert.pem -CAcreateserial -days 365`
4. 将签证之后的证书文件user-cert.pem导出为p12格式文件（p12格式可以被浏览器识别并安装到证书库中）
`openssl pkcs12 -export -clcerts -in user-cert.pem -inkey user-key.pem -out user.p12` 
5. 将签证之后的用户证书导入服务端信任库（**由于没有去CA认证中心购买个人证书，所以只有导入信任库才可进行双向ssl交互**)
`keytool -import -alias usertrustcacerts -file user-cert.pem -keystore servertrust.jks`
6. 查看服务端信任库中用户条目信息
`keytool -list -v -alias user -keystore servertrust.jks -storepass 123456`

### 配置web容器

## JAVA实现

采用JSSE及keytool生成的证书，分别生成SSLServerSocket,SSLSocket  
client采用kclient.keystore中的clientkey私钥进行数据加密，发送给server  
server采用tserver.keystore中的client.crt证书（包含了clientkey的公钥）对数据解密，如果解密成功，证明消息来自client，进行逻辑处理  
server采用kserver.keystore中的serverkey私钥进行数据加密，发送给client  
client采用tclient.keystore中的server.crt证书（包含了serverkey的公钥）对数据解密，如果解密成功，证明消息来自server，进行逻辑处理  
如果过程中，解密失败，那么证明消息来源错误。不进行逻辑处理。这样就完成了双向的身份认证。

[完整样例](https://github.com/yoloz/Samples/tree/master/samples-ssl)

服务端，生成SSLServerSocket代码

``` java
SSLContext ctx = SSLContext.getInstance("SSL");

KeyManagerFactory kmf = KeyManagerFactory.getInstance("SunX509");
TrustManagerFactory tmf = TrustManagerFactory.getInstance("SunX509");

//            KeyStore ks = KeyStore.getInstance("JKS");
//            KeyStore tks = KeyStore.getInstance("JKS");
//            ks.load(new FileInputStream(path.resolve("keytool/kserver.keystore").toFile()), SERVER_KEY_STORE_PASSWORD.toCharArray());
//            tks.load(new FileInputStream(path.resolve("keytool/tserver.keystore").toFile()), SERVER_TRUST_KEY_STORE_PASSWORD.toCharArray());

KeyStore ks = KeyStore.getInstance("pkcs12");
KeyStore tks = KeyStore.getInstance("pkcs12");
ks.load(new FileInputStream(path.resolve("openssl/server.p12").toFile()), SERVER_KEY_STORE_PASSWORD.toCharArray());
tks.load(new FileInputStream(path.resolve("openssl/root.p12").toFile()), SERVER_TRUST_KEY_STORE_PASSWORD.toCharArray());

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

//            KeyStore ks = KeyStore.getInstance("JKS");
//            KeyStore tks = KeyStore.getInstance("JKS");
//            ks.load(new FileInputStream(path.resolve("keytool/kclient.keystore").toFile()), CLIENT_KEY_STORE_PASSWORD.toCharArray());
//            tks.load(new FileInputStream(path.resolve("keytool/tclient.keystore").toFile()), CLIENT_TRUST_KEY_STORE_PASSWORD.toCharArray());

KeyStore ks = KeyStore.getInstance("pkcs12");
KeyStore tks = KeyStore.getInstance("pkcs12");
ks.load(new FileInputStream(path.resolve("openssl/client.p12").toFile()), CLIENT_KEY_STORE_PASSWORD.toCharArray());
tks.load(new FileInputStream(path.resolve("openssl/root.p12").toFile()), CLIENT_TRUST_KEY_STORE_PASSWORD.toCharArray());

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






