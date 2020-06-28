> 关于HTTPS的知识，查找了很多博客，才大致弄明白。我把大致流程，和几个重点相关知识都写上了。

涉及以下四个知识点

- 数字签名

- 数字证书

- 对称加密

- 非对称加密

![整体流程](https://tva1.sinaimg.cn/large/00831rSTly1gd7jqndkblj318p0lcjvr.jpg)

### 1. HTTPS介绍

> HTTPS，又称 HTTP over TLS。
>
> TLS的前身是SSL，TLS 1.0 通常被标示为 SSL3.1，TLS 1.1 为 SSL 3.2，TLS 1.2 为 SSL 3.3



HTTPS和HTTP协议相比，提供了

1. 数据完整性：内容传输经过完整性校验

2. 数据隐私性：内容经过对称加密，每个链接生成一个唯一的加密密钥

3. 身份认证：第三方违法伪造服务端（客户端）身份



其中，数据完整性和隐私性 由 TLS Record Protocol 保证，身份认证由 TLS Handshaking Protocols实现。



- TLS Handshake Protocol

> 客户端与服务端在握手消息中，交换了`client_random`和`server_random`，使用RSA公钥加密传输`premaster secret` ，最后通过算法，双方计算出 `master secret`。

除了使用RSA算法在公共信道交换密钥，也可以使用 Diffie-Hellman算法

- TLS Record Protocol

> 即 3.1节 的工作流程

[猫尾博客 HTTPS工作原理](https://cattail.me/tech/2015/11/30/how-https-works.html)



------



### 2. 工作流程

1. 客户端(浏览器) 发送随机数 `client_random` 和支持的加密方式列表
2. 服务器(网站) 返回随机数 `server_random` ，选择的加密方式和服务器数字证书
3. 客户端验证 **服务器数字证书** ，使用证书中的 **服务器公钥 非对称加密**  `premaster secret` 发送给服务端
4. 服务端使用对应的 **私钥** 解密 `premaster secret`
5. 两端分别通过 `client_random ` , `server_random` 和  `premaster secret`  生成  `master secret ` ，用于 **对称加密** 后续通信内容



------



### 3. 数字证书

- 网站的公钥
- 数字证书的签发机构信息
- 证书持有者网站自身信息
- 证书有效期
- 指纹 & 指纹/hash/摘要 算法
- 数字签名
- 。。。。



#### 3.1 server端数字签名的计算

server找CA机构，对数字证书的内容，用指纹算法计算得到一个hash值，然后使用CA根私钥，将hash值加密得到数字签名



#### 3.2 数字证书的验证

客户端浏览器，验证证书时，首先通过 **浏览器内置的CA机构根公钥**，将数字签名解密为hash1值。

然后用证书中的指纹hash算法，将证书内容计算成hash2值。

对比hash1和hash2，就能知道内容是否被篡改过。



------



### 4. 发送数据

#### 4.1 工作流程

- 发送端，将数据 (Record) 分段，压缩，增加MAC（Message Authentication Code 消息认证码）并**对称加密**
- 接收端，将数据 (Record) 解密，验证MAC ，解压并重组



#### 4.2 消息散列码 MAC

消息认证码(Message Authentication Code) 是一种 **确认完整性** 并进行 **认证** 的技术，简称 MAC 。或者说，**是一种与密钥相关联的单项散列函数**。

使用消息认证码，可以确认自己收到的消息，是否是发送者的本意，也就是可以判断消息是否被篡改(完整性)，以及是否有人伪装成发送者发送了这条消息(认证)。

> 消息认证码所需的密钥，存在密钥配送问题。为了密钥在配送时不被窃取。需要用上述工作流程中讲的方式，或**者Diffie-Hellman密钥交换** 等安全方式 配送密钥

[消息认证码是怎么一回事？](https://halfrost.com/message_authentication_code/)

