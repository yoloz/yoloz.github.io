---
title: X.509数字证书中的扩展项
comments: false
toc: false
date: 2020-04-17 17:46:50
categories: security
tags:
---

&emsp;&emsp;在 RFC 5280《Internet X.509 Public Key Infrastructure Certificate and Certificate Revocation List (CRL) Profile》中定义了 X.509 公钥数字证书和证书撤销列表的内容和格式。<br/>
&emsp;&emsp;在 X.509 编码格式的数字证书中，使用 Issuer 子项标明证书的颁发者，Issuer 子项必须包含一个非空的 DN (Distinguished Name) 名，证书中还可以使用扩展项 issuerAltName (即 Issuer Alternative Names 的缩写) 来表示证书颁发者的其他名称，issuerAltName 是一个非关键的（non-critical）扩展项。<br/>
&emsp;&emsp;数字证书使用 Subject 子项标明证书的持有者，并且使用 subjectAltName 扩展项对持有者的身份进行更多的标记和界定，在 subjectAltName 中可以包含有关持有者身份的多条信息。**如果证书持有者是一个CA，则 Subject 项不能为空，必须包含一个 DN 名，该 DN 名必须与该 CA 颁发的所有证书中 Issuer 项的 DN 名相同。**<br/>
&emsp;&emsp;在 RFC 5280 4.2.1.6.小节 “Subject Alternative Name” 中，定义了**subjectAltName（即Subject Alternative Name，还可以缩写为：SAN），它可以包含以下内容：电子邮件地址、域名、IP 地址、URI（Uniform Resource Identifier），而且以上类型的值可以存在一个或多个。如果证书中的 Subject 项的内容为空，则 CA 在颁发证书时，必须在证书中放入扩展项 subjectAltName，并且应将该扩展项标记为 critical。当证书中的 Subject 项包含不为空的 DN 名时，CA 在颁发证书时必须将 subjectAltName 标记为 non-critical。**<br/>
* 当扩展项 subjectAltName 包含一个电子邮件地址时，邮件地址的格式必须为 Local-part@Domain 形式，例如 TestUser@163.com，注意不能写成 \<Local-part@Domain\>，即不要使用尖括号 < 和 > 将邮件地址包起来。类似于 subscriber@example.com 形式的电子邮件地址是合法的，但是像 subscriber.example.com 这种形式的电子邮件地址不允许在SubAltName 中出现。
* 当扩展项 subjectAltName 包含一个 IP 地址时，可以使用 IPv4 或 IPv6 形式的地址。
* 当扩展项 subjectAltName 包含一个 dNSName 名时，此时证书持有者的名称是一个域名，这个域名以 IA5String 格式表示，被存储在 subjectAltName 的 dNSName 子项中。尽管空字符串也是一个合法的域名，但是在 subjectAltName 中不允许使用值为空串的 dNSName 项。

在 RFC 5280 中对 subjectAltName 的 ASN.1 编码定义如下：

``` shell
id-ce-subjectAltName  OBJECT IDENTIFIER ::= { id-ce 17 }

SubjectAltName  ::=  GeneralNames

GeneralNames  ::=  SEQUENCE SIZE (1..MAX) OF GeneralName

GeneralName  ::=  CHOICE {
    otherName [0]  OtherName,
    rfc822Name [1]  IA5String,
    dNSName [2]  IA5String,
    x400Address [3]  ORAddress,
    directoryName [4]  Name,
    ediPartyName [5]  EDIPartyName,
    uniformResourceIdentifier [6]  IA5String,
    iPAddress [7]  OCTET STRING,
    registeredID [8]  OBJECT IDENTIFIER }

OtherName  ::=  SEQUENCE {
    type-id  OBJECT IDENTIFIER,
    value [0]  EXPLICIT ANY DEFINED BY type-id }

EDIPartyName  ::=  SEQUENCE {
    nameAssigner [0]  DirectoryString OPTIONAL,
    partyName [1]  DirectoryString }
```

&emsp;&emsp;根据 RFC 6125《Representation and Verification of Domain-Based Application Service Identity within Internet Public Key Infrastructure Using X.509 (PKIX) Certificates in the Context of Transport Layer Security (TLS)》中的规定，**当一个网站使用 TLS 证书标记自己的身份时，如果证书中包含 subjectAltName，在识别证书持有者时会忽略 Subject 子项，将以 subjectAltName 中包含的内容来识别证书持有者。在早期颁发的证书中可能未包含 subjectAltName，但是在新颁发的证书中必须包含 subjectAltName 项。**最新版本的浏览器如 Chrome、Firefox 等在通过 HTTPS 访问 web 网站时，会对网站证书中的 subjectAltName 项进行检查，如果未包含该项或该项有问题，Chrome 浏览器会报告 NET::ERR_CERT_COMMON_NAME_INVALID 错误，并提示“安全证书没有指定主题备用名称”。

&emsp;&emsp;对于一张 TLS 证书，如果这张证书将被用于多个网址，则应在 Subject 子项中CN(CommonName)部分放入一个最重要的URI，并且只能包含一个 URI。可以将所有的 URI 放置到 subjectAltName（即：SAN）中，根据 CA/Browser Forums（即：CA/浏览器论坛国际组织）的规定，**在 Subject 子项 CommonName 部分出现的 URI 必须在 SAN 中再次出现，而且通常出现在“SAN DNS Name=...”列表中第一个元素的位置。浏览器会检查当前访问的网址是否与证书中 SAN 扩展项里包含的 DNS Name 列表中的某一个网址相符，如果不符就会报错。**

```shell
$ keytool -genkeypair \
        -alias test \
        -keyalg RSA \
        -keypass 123456 \
        -sigalg SHA256withRSA \
        -dname "cn=github.com,ou=github.com,Inc.,o=Github, Inc.,l=San Francisco,st=California,c=US" \
        -validity 3650 \
        -keystore ddssingsong.p12 \
        -storetype PKCS12 \
        -storepass 123456
        -ext SAN=dns:github.com,dns:www.github.com,ip:127.0.0.1 ##subject子项中CN和subjectAltName(SAN)扩展项里的DNS第一个相符
```