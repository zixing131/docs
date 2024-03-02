

# 奇安信攻防社区-JWT 渗透姿势

### JWT 渗透姿势

JWT（JSON Web Token）是一种无状态认证机制，通过将用户身份和权限信息存储在令牌中，实现安全地在网络应用间传递信息。它具有跨域支持、扩展性和灵活性、安全性以及可扩展的验证方式等特点，成为现代应用开发中重要的认证和授权解决方案。

## 0X01 什么是 jwt？

> JWT 全称 JSON Web Token，是一种标准化格式，用于在系统之间发送加密签名的 JSON 数据。
> 
> 原始的 Token 只是一个 uuid，没有任何意义。

JWT 的结构由三部分组成，分别是 Header、Payload 和 Signature，下面是每一部分的详细介绍和示例：

### Header 部分

在 JWT 中 Header 部分存储的是 Token 类型和加密算法，通常使用 JSON 对象表示并使用 Base64 编码，其中包含两个字段：alg 和 typ

-   alg(algorithm)：指定了使用的加密算法，常见的有 HMAC、RSA 和 ECDSA 等算法
-   typ(type)：指定了 JWT 的类型，通常为 JWT

下面是一个示例 Header：

{  
"alg": "HS256",  
"typ": "JWT"  
}

### Payload 部分

Payload 包含了 JWT 的主要信息，通常使用 JSON 对象表示并使用 Base64 编码，Payload 中包含三个类型的字段：注册声明、公共声明和私有声明

-   公共声明：是自定义的字段，用于传递非敏感信息，例如：用户 ID、角色等
-   私有声明：是自定义的字段，用于传递敏感信息，例如密码、信用卡号等
-   注册声明：预定义的标准字段，包含了一些 JWT 的元数据信息，例如：发行者、过期时间等

下面是一个示例 Payload：

{  
"sub": "1234567890",  
"name": "John Doe",  
"iat": 1516239022  
}

其中 sub 表示主题，name 表示名称，iat 表示 JWT 的签发时间

### Signature 部分

Signature 是使用指定算法对 Header 和 Payload 进行签名生成的，用于验证 JWT 的完整性和真实性

-   Signature 的生成方式通常是将 Header 和 Payload 连接起来然后使用指定算法对其进行签名，最终将签名结果与 Header 和 Payload 一起组成 JWT
-   Signature 的生成和验证需要使用相同的密钥

下面是一个示例 Signature

HMACSHA256(base64UrlEncode(header) + "." +base64UrlEncode(payload),secret)

其中 HMACSHA256 是使用 HMAC SHA256 算法进行签名，header 和 payload 是经过 Base64 编码的 Header 和 Payload，secret 是用于签名和验证的密钥，最终将 Header、Payload 和 Signature 连接起来用句点 (.) 分隔就形成了一个完整的 JWT

### 完整的 JWT

第一部分是 Header，第二部分是 Payload，第三部分是 Signature，它们之间由三个 `.` 分隔，注意 JWT 中的每一部分都是经过 Base64 编码的，但并不是加密的，因此 JWT 中的信息是可以被解密的

下面是一个示例 JWT

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV\_adQssw5c

## 0X02 解密平台

下面是一个 JWT 在线构造和解构的平台：

[https://jwt.io/](https://jwt.io/)

![image-20231222210239843](assets/1706247644-f2907cb5250042c3ff8410c1017fa7e7.png)

## 0X03 工作原理

JWT 的工作流程如下：

-   用户在客户端登录并将登录信息发送给服务器
-   服务器使用私钥对用户信息进行加密生成 JWT 并将其发送给客户端
-   客户端将 JWT 存储在本地，每次向服务器发送请求时携带 JWT 进行认证
-   服务器使用公钥对 JWT 进行解密和验证，根据 JWT 中的信息进行身份验证和授权
-   服务器处理请求并返回响应，客户端根据响应进行相应的操作

## 0X04 jWT 名词

1.  JWS（Signed JWT）：JWS 是指已签名的 JWT。它由 JWT 的 Header、Payload 和 Signature 组成，其中 Signature 是使用密钥对 Header 和 Payload 进行数字签名得到的。通过验证签名，可以确保 JWT 的完整性和真实性。
2.  JWK（JSON Web Key）：JWK 是指用于 JWT 的密钥。它可以是对称加密密钥（例如密码），也可以是非对称加密密钥（例如公钥/私钥对）。JWK 用于生成和验证 JWT 的签名，确保只有拥有正确密钥的一方能够对 JWT 进行操作。
3.  JWE（Encrypted JWT）：JWE 是指经过加密的 JWT。它是在 JWS 基础上进行了进一步的加密，将 JWT 的 Payload 部分加密后得到的结果。JWE 可用于保护敏感信息，确保只有授权的接收方能够解密和读取 JWT 的内容。
4.  JKU（JSON Web Key Set URL）：JKU 是 JWT Header 中的一个字段，该字段包含一个 URI，用于指定用于验证令牌密钥的服务器。当需要获取公钥或密钥集合时，可以使用 JKU 字段指定的 URI 来获取相关的 JWK 信息。
5.  X5U：X5U 是 JWT Header 中的一个字段，它是一个 URL，指向一组 X.509 公钥证书。类似于 JKU，X5U 字段用于指定可用于验证 JWT 的公钥证书的位置。
6.  X.509 标准：X.509 是一种密码学标准，定义了公共密钥基础设施（PKI）中的数字证书格式。这些证书包含有关实体（例如个人、组织或设备）的信息，以及相关的公钥和数字签名。X.509 证书在许多互联网协议中广泛使用，如 TLS/SSL 等。

## 0x05 JWT 基础安全问题

### 1、未对签名进行验证

JWT 库会通常提供一种验证令牌的方法和一种解码令牌的方法，比如:Node.js 库 jsonwebtoken 有 verify() 和 decode()，有时开发人员会混淆这两种方法，只将传入的令牌传递给 decode() 方法，这意味着应用程序根本不验证签名

下边我们通过 portswigger 靶场来演示一下这个漏洞案例：

靶场地址：[https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-unverified-signature](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-unverified-signature)

![image-20231222221236099](assets/1706247644-6b35d557cf3fbec73f7a39f7be254e56.png)

（1）首先看看通关要求：修改您的会话令牌以访问管理面板`/admin`，然后删除用户`carlos`

（2）前文我们说到，JWT 需要开发者提供一个 Signature（签名），如果我们不对签名进行验证，极有可能产生如下的越权情况。

（3）打开靶场，登录，访问`/admin`

![image-20231222221534305](assets/1706247644-cced0bf0a159d8cfd5c83c82316b0bb5.png)

（4）因为我们使用的 jwt，所以权限相关的设置肯定在 jwt 中。我们抓个包拿到 jwt 解密看看

![image-20231222221823154](assets/1706247644-45f43c490304d5c11a03e2e1d1ff0165.png)

![image-20231222221925066](assets/1706247644-cf98d3329a75ce53632b5a353c8c3bd3.jpeg)

（5）把`wiener`修改成`administrator`

把第二部分 payload 拿出来 base64 解密，然后修改，修改万再拼接会 jwt 中

![image-20231222222217488](assets/1706247644-27b7209973a875e607f5b42660896b45.jpeg)

![image-20231222222319473](assets/1706247644-6eda7707f18c094213a5c24fafa01c15.png)

![image-20231222222404181](assets/1706247644-bd329e587f2cf153cfa95a1bae4a0a31.png)

（6）把修改后的 jwt 替换原本的 jwt

![image-20231222222504138](assets/1706247644-789156634ca028f3e08bd6e9ce423187.png)

（7）访问`/admin`，删除`carlos`用户即可通关

![image-20231222222553594](assets/1706247644-064b21c11f1d949193f69982afdf17ae.png)

![image-20231222222634903](assets/1706247644-36af5f243d0e60057d9d561196b71700.png)

### 2、未对加密算法进行强验证

在 JWT 的 Header 中 alg 的值用于告诉服务器使用哪种算法对令牌进行签名，从而告诉服务器在验证签名时需要使用哪种算法，目前可以选择 HS256，即 HMAC 和 SHA256，JWT 同时也支持将算法设定为"None"，如果"alg"字段设为"None"，则标识不签名，这样一来任何 token 都是有效的，设定该功能的最初目的是为了方便调试，但是若不在生产环境中关闭该功能，攻击者可以通过将 alg 字段设置为"None"来伪造他们想要的任何 token，接着便可以使用伪造的 token 冒充任意用户登陆网站

下边我们通过 portswigger 靶场来演示一下这个漏洞案例：

靶场地址：[https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-flawed-signature-verification](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-flawed-signature-verification)

![image-20231222223126751](assets/1706247644-822b400f5d5982e83bd897f8354e1fe3.png)

这关与上边的漏洞原理不同，但最终的效果都是可以`伪造token`，攻击手法与上一关卡相同，唯一不同的是这次是需要`把header中的alg参数的值改为none`即可！

不在演示！

### 3、弱密钥

在实现 JWT 应用程序时，开发人员有时会犯一些错误，比如：忘记更改默认密码或占位符密码，他们甚至可能复制并粘贴他们在网上找到的代码片段然后忘记更改作为示例提供的硬编码秘密，在这种情况下攻击者使用众所周知的秘钥来暴力破解服务器的秘钥是很容易的

下边我们通过 portswigger 靶场来演示一下这个漏洞案例：

靶场地址：[https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-weak-signing-key](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-weak-signing-key)

![image-20231222224611748](assets/1706247644-13282e6dabd6f4de18645045cd868ccb.png)

（1）打开靶场 --> 登录 --> 抓包 --> 拿到 JWT

![image-20231222225235639](assets/1706247644-6f3ad4058d7d4741ff95e9beb94d8bef.png)

eyJraWQiOiIwZWMxNmY3ZS0yOGU0LTQxMjUtYjUxMS0yZDc1ZmRjZjRiM2QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsInN1YiI6IndpZW5lciIsImV4cCI6MTcwMzI2MDMwOH0.t5QY8-kd-bzIqp0PyXg2EUUxg1jPl6-NYKBI4BTw8P0

（2）JWT 字典：[https://github.com/wallarm/jwt-secrets](https://github.com/wallarm/jwt-secrets)

（3）我们使用`jwt_tool`来爆破 JWT：[https://github.com/ticarpi/jwt\_tool](https://github.com/ticarpi/jwt_tool)

使用教程：[https://www.cnblogs.com/xiaozi/p/12005929.html](https://www.cnblogs.com/xiaozi/p/12005929.html)

![image-20231222225046915](assets/1706247644-d53967e6a24770078c762f0e711d7a17.png)

![image-20231222225746340](assets/1706247644-08863480691f1826ca418f573e44b3bf.png)

（4）拿到爆破出来的秘钥`secret1`，我们篡改 jwt

![image-20231222230033259](assets/1706247644-903e3ef19fe786b35b427f303fb1d971.png)

（5）修改本地或者抓包修改 jwt 都可以，然后访问/admin，删除`carlos`用户

![image-20231222230440373](assets/1706247644-799d01b9103c2eb9ed34fa9a31ec5b52.png)

## 0X06 JWT 标头注入

### 1、通过 jwk 参数注入自签名的 JWT

（1）再来回顾一下 jwk 是什么吧！

JWK（JSON Web Key）：JWK 是指用于 JWT 的密钥。它可以是对称加密密钥（例如密码），也可以是非对称加密密钥（例如公钥/私钥对）

（2）漏洞原理

在理想情况下，服务器应该是只使用公钥白名单来验证 JWT 签名的，但对于一些相关配置错误的服务器会用 JWK 参数中嵌入的任何密钥进行验证，攻击者就可以利用这一行为，用自己的 RSA 私钥对修改过的 JWT 进行签名，然后在 JWK 头部中嵌入对应的公钥进行越权操作

（3）RSA 加密算法

[https://www.cnblogs.com/pcheng/p/9629621.html](https://www.cnblogs.com/pcheng/p/9629621.html)

这位师傅的文章总结的非常好，**公钥加密、私钥解密、私钥签名、公钥验签。**

下边我们通过 portswigger 靶场来演示一下这个漏洞案例：

靶场地址：[https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-jwk-header-injection](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-jwk-header-injection)

![image-20231223025921262](assets/1706247644-187788eb3b947b7ab3e7da2994137699.png)

（1）这个案例我们使用`burpsuite`，首先先去安装一个插件`jwt enditor`

![image-20231223030051079](assets/1706247644-015c71d344e8c5ca721ab2f2602b189f.png)

（2）安装好了后我们使用它生成一个新的 RSA 密钥

![image-20231223030327718](assets/1706247644-adbe9d732396625606bbeb171e334ec3.png)

（3）打开靶场 --> 登录 --> bp 抓包 --> 发送到`Repeat`模块

![image-20231223030946972](assets/1706247644-378c9ab9fa6713c5842d585748ae8ee3.png)

（4）将 sub 内容修改为 administrator

![image-20231223031024846](assets/1706247644-9fc8472911b7ecb113fa8c6c3e89bfd8.png)

（5）点击"Attack"，然后选择"Embedded JWK"，

![image-20231223031129929](assets/1706247644-494732cb885c0334a64ed112bc4684a2.png)

（6）出现提示时选择您新生成的 RSA 密钥

![image-20231223031203432](assets/1706247644-c08d91784f25c8a9f742124c146b5ec8.png)

（7）复制新生成的 jwt

![image-20231223031245153](assets/1706247644-aef8d6d493cd67a02fd823dd3308a83f.png)

（8）替换本地的 jwt

![image-20231223031352592](assets/1706247644-d6de5ddf87289a25d62c12589d3052ce.png)

（9）然后访问/admin，删除`carlos`用户

![image-20231223031452606](assets/1706247644-231ec6ed03173707710409fae46b7dc8.png)

### 2、通过 jku 参数注入自签名的 JWT

（1）先回顾什么是 jku？

JKU（JSON Web Key Set URL）：JKU 是 JWT Header 中的一个字段，该字段包含一个 URI，用于指定用于验证令牌密钥的服务器。当需要获取公钥或密钥集合时，可以使用 JKU 字段指定的 URI 来获取相关的 JWK 信息。

（2）看看他长什么样？

![image-20231223034620634](assets/1706247644-11341c71da918f6f8d7d9961b27cc8bd.png)

下边我们通过 portswigger 靶场来演示一下这个漏洞案例：

靶场地址：[https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-jku-header-injection](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-jku-header-injection)

（1）分析一下

漏电点都是仅通过公钥判断数据是否被篡改，但公钥在 header 头中，用户可控！

唯一的区别就是这关中的公钥使用服务器获取的。

（2）那还是之前的流程

-   `jwt enditor`生成一个新的 RSA 密钥，使用之前的也可以
-   打开靶场 --> 登录 --> bp 抓包 --> 发送到`Repeat`模块

（3）复制公钥，拿到漏洞利用服务器（在题目中选择"Go eo exploit server"，然后加上 key 头并保存到 exploit 的 body 中）

![image-20231223033528304](assets/1706247644-30a48a2641fa439dcd3b229db2771233.png)

切记加`key`头

{  
"keys": \[  
​  
\]  
}

![image-20231223041017444](assets/1706247644-d669ead1e6905bb495543c7977e4153a.png)

（4）在 bp 中需要改动三处

![image-20231223041438999](assets/1706247644-42d6c03b619745ead01af0b1f34f8ca6.png)

-   kid：改成我们插件生成的公钥中的 kid
-   jku：添加 jdu 字段（漏洞利用服务器地址）![image-20231223034301655](assets/1706247644-d07e9317b4855c9140ef84f6b69d9780.png)
-   sub：讲 wiener 改为 administrator

（5）点击下面的 sign，选择 Don’t modify header 模式

![image-20231223034459509](assets/1706247644-aea3ba4ed5afece62ef3c1bfbe87e185.png)

（6）之后的流程

替换本地 jwt --> 删除`carlos`用户

![image-20231223041239306](assets/1706247644-0213c631a2d025e96238b4e8940332f1.png)

### 3、通过 kid 注入 JWT，与目录遍历攻击相结合

JWS 规范没有针对 kid 进行严格设置，比如必须是 uuid 的格式或者是其他的，它只是开发人员选择的任意字符串。

那么我们可以通过将 kid 指向数据库中的特定条目，或者是指向文件名，作为验证密钥。

例如：

{  
"kid": "../../path/to/file",  
"typ": "JWT",  
"alg": "HS256",  
"k": "asGsADas3421-dfh9DGN-AFDFDbasfd8-anfjkvc"  
}

当 JWT 使用的是对称加密算法的时候，极有可能存在目录遍历的漏洞，我们能够强制服务器使用其文件系统中的任意文件作为验证密钥。

我们可以先尝试读取`dev/null`这一文件，`dev/null`这一文件默认为空文件，返回为 null，我们可以在 Symmetric Key 中，将 k 值修改为`AA==`也就是 null，进行攻击。

下边我们通过 portswigger 靶场来演示一下这个漏洞案例：

靶场地址：[https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-kid-header-path-traversal](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-kid-header-path-traversal)

![image-20231223163130615](assets/1706247644-e094d335d9b85d566c06f51fe41862cf.png)

（1）使用 bp 插件`jwt enditor`生成一个 **Symmetric Key**，也就是对称密钥，并将 k 的值修改为`AA==`

![image-20231223162359656](assets/1706247644-0b8861cd097499b967452c317abc350f.png)

（2）打开靶场 --> 登录 --> bp 抓包 --> 发送到`Repeat`模块

（3）接着，我们在抓到的包中修改 kid 值，尝试用目录遍历读取`dev/null`此文件。并将 sub 修改为 administrator

![image-20231223162825481](assets/1706247644-00cef819ed3b6168f850b23b73c9b40b.png)

（4）点击下面的 Sign，使用 OCT8 的密钥攻击。

![image-20231223162857397](assets/1706247644-14c9b4d6e72ec9a7c163c2b819103cf9.png)

（5）接着就是：替换本地 jwt --> 删除`carlos`用户

因为是目录遍历，所以多尝试。

![image-20231223162953206](assets/1706247644-0a37b841b19f4d9daf1471da2bbf0937.png)

## 0X07 JWT 算法混淆

### 1、对称加密与非对称加密。

可以使用一系列不同的算法对 JWT 进行签名。其中一些，如 HS256（HMAC + SHA-256）使用“对称”密钥。这意味着服务器使用单个密钥对 Token 进行签名和验证。显然，这需要保密，就像密码一样。

![](assets/1706247644-cde31d021c81df2d39f8d5f9d893b9a5.png)

其他算法，例如 RS256 (RSA + SHA-256) 使用“非对称”密钥对。它由服务器用来签署令牌的私钥和可用于验证签名的数学相关的公钥组成。

![](assets/1706247644-bd927821295e8bba7fefcea9b9029390.png)

顾名思义，私钥必须保密，但公钥通常是共享的，以便任何人都可以验证服务器颁发的令牌的签名。

### 2、算法混淆漏洞产生的原因？

算法混乱漏洞通常是由于 `JWT 库的实现有缺陷而引起`的。尽管实际的验证过程因所使用的算法而异，但许多库提供了一种与算法无关的单一方法来验证签名。这些方法依赖于`alg`令牌标头中的参数来确定它们应执行的验证类型。

以下伪代码显示了此泛型`verify()`方法的声明在 JWT 库中的简化示例：

function verify(token, secretOrPublicKey){  
algorithm \\= token.getAlgHeader();  
if(algorithm \\== "RS256")  
{  
// Use the provided key as an RSA public key  
}else if (algorithm \\== "HS256")  
{  
// Use the provided key as an HMAC secret key  
}  
}

当随后使用此方法的网站开发人员假设它将专门处理使用 RS256 等非对称算法签名的 JWT 时，就会出现问题。由于这个有缺陷的假设，他们可能总是将固定的公钥传递给该方法，如下所示：

publicKey = <public-key-of-server>;  
token = request.getCookie("session");  
verify(token, publicKey);

在这种情况下，如果服务器收到使用 HS256 等对称算法签名的令牌，则库的通用`verify()`方法会将公钥视为 HMAC 密钥。这意味着攻击者可以使用 HS256 和公钥对令牌进行签名，并且服务器将使用相同的公钥来验证签名。

上边是抄官方的话，下边我们用大白话来解释一下：

-   假设开发使用的是`RS256`这非对称加密算法生成的 jwt。
-   由于信息泄露等原因攻击者可以拿到这个`公钥`，因为上边说过公钥通常是共享的
-   攻击者使用`HS256`算法伪造一个 jwt，用这个`公钥`作为签名的密钥。

程序会使用`verify()`这个方法来验证 jwt 有没有被篡改。但是这个库设计的有问题（问题：他是通过你 jwt 头中`alg`来判断是使用那种算法来进行签名的。所以我们可以篡改他的算法），这块就会使用`RS256`生成的公钥作为`HS256`的秘钥来验证攻击者伪造的 jwt。这个公钥攻击者可控，所以伪造的 jwt 就会通过验证。

### 3、执行算法混淆攻击的步骤

算法混淆攻击通常涉及以下高级步骤：

-   [获取服务器的公钥](https://portswigger.net/web-security/jwt/algorithm-confusion#step-1-obtain-the-server-s-public-key)
-   [将公钥转换为合适的格式](https://portswigger.net/web-security/jwt/algorithm-confusion#step-2-convert-the-public-key-to-a-suitable-format)
-   [创建一个恶意 JWT，](https://portswigger.net/web-security/jwt/algorithm-confusion#step-3-modify-your-jwt)其负载经过修改，`alg`标头设置为`HS256`.
-   使用公钥作为秘密，使用 [HS256 对令牌进行签名。](https://portswigger.net/web-security/jwt/algorithm-confusion#step-4-sign-the-jwt-using-the-public-key)

### 4、通过算法混淆绕过 JWT 身份验证

下边我们通过 portswigger 靶场来演示一下这个漏洞案例：

靶场地址：[https://portswigger.net/web-security/jwt/algorithm-confusion/lab-jwt-authentication-bypass-via-algorithm-confusion](https://portswigger.net/web-security/jwt/algorithm-confusion/lab-jwt-authentication-bypass-via-algorithm-confusion)

![image-20231223172704453](assets/1706247644-b4b1eed142a49f4d370619f35412918f.png)

（1）获取服务器的公钥

服务器有时通过映射到/jwks.json 或/.well-known/jwks.json 的端点将它们的公钥公开为 JSON Web Key(JWK) 对象，比如大家熟知的/jwks.json，这些可能被存储在一个称为密钥的 jwk 数组中，这就是众所周知的 JWK 集合

![image-20231223171250162](assets/1706247644-d8d066fc53d0fbfb093db2134960224b.png)

（2）将公钥转换为合适的格式

在 Burpsuite 的`JWT Editor` 中点击"New RSA Key"，用上边获取到泄露的 JWK 而生成一个新的 RSA Key

![image-20231223171605944](assets/1706247644-d3e3e480059c6bac209959a18e3a5826.png)

选中"Copy Public Key as PEM"，同时将其进行 base64 编码操作，保存一下得到的字符串 (备注：上下的一串-----END PUBLIC KEY-----不要删掉)

\-----BEGIN PUBLIC KEY-----  
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAj+eOXtjDkD6BYr0ftlLo  
rnU+xsXB2btxi4REHYghwP4YCiZjX7UsPvEYRWyt8FJzyQap+zUoueiFTWBt/Ngt  
qOCQWPUDMv9BQ3Kjpos6yC/PM8TEmJLsg0F2b1OcIoDuPgo9v0JWmSmpS+THqUwH  
xgizbwFBbZxS+aGPV9vv0KyULDV2CLjWjbyYh+2sJZFW7DFq1EHWedtqmTcY3/Gt  
Sv3CBNdv9Hn/J5d5P9gorrbuKrPnc2qD967poetwrmI/9TxQdCVSEjqLdBqEIzBg  
IbLRST2J0DNHX54ESyjcutmfRG833wEm1c8S98bhG3eGx+HpqX5/hkPPlwdZTPTy  
9QIDAQAB  
\-----END PUBLIC KEY-----

LS0tLS1CRUdJTiBQVUJMSUMgS0VZLS0tLS0KTUlJQklqQU5CZ2txaGtpRzl3MEJBUUVGQUFPQ0FROEFNSUlCQ2dLQ0FRRUFqK2VPWHRqRGtENkJZcjBmdGxMbwpyblUreHNYQjJidHhpNFJFSFlnaHdQNFlDaVpqWDdVc1B2RVlSV3l0OEZKenlRYXArelVvdWVpRlRXQnQvTmd0CnFPQ1FXUFVETXY5QlEzS2pwb3M2eUMvUE04VEVtSkxzZzBGMmIxT2NJb0R1UGdvOXYwSldtU21wUytUSHFVd0gKeGdpemJ3RkJiWnhTK2FHUFY5dnYwS3lVTERWMkNMaldqYnlZaCsyc0paRlc3REZxMUVIV2VkdHFtVGNZMy9HdApTdjNDQk5kdjlIbi9KNWQ1UDlnb3JyYnVLclBuYzJxRDk2N3BvZXR3cm1JLzlUeFFkQ1ZTRWpxTGRCcUVJekJnCkliTFJTVDJKMEROSFg1NEVTeWpjdXRtZlJHODMzd0VtMWM4Uzk4YmhHM2VHeCtIcHFYNS9oa1BQbHdkWlRQVHkKOVFJREFRQUIKLS0tLS1FTkQgUFVCTElDIEtFWS0tLS0tCg==

（3）生成一个对称加密的 key，把 k 替换成我们刚修改完格式的公钥

![image-20231223172157829](assets/1706247644-21fb66bb78ac55d4d8850cb601296f45.png)

（4）篡改 jwt。

![image-20231223172427569](assets/1706247644-e1e679aa264417418e8a13b1b64365cb.png)

（5）使用这个公钥签名

![image-20231223172523042](assets/1706247644-5bc3b5a73918e2438a7098e9d02777b9.png)

（6）替换本地 jwt --> 删除`carlos`用户

![image-20231223172605282](assets/1706247644-a379aafd2997aaf478e56dc35475c17b.png)

### 5、通过算法混淆（不暴露密钥）绕过 JWT 身份验证

在公钥不可用的情况下您仍然可以通过使用 jwt \_ forgery.py 之类的工具从一对现有的 JWT 中获取密钥来测试算法混淆，您可以在 rsa\_sign2n GitHub 存储库中找到几个有用的脚本

[https://github.com/silentsignal/rsa\_sign2n](https://github.com/silentsignal/rsa_sign2n)

下边我们通过 portswigger 靶场来演示一下这个漏洞案例：

靶场地址：[https://portswigger.net/web-security/jwt/algorithm-confusion/lab-jwt-authentication-bypass-via-algorithm-confusion-with-no-exposed-key](https://portswigger.net/web-security/jwt/algorithm-confusion/lab-jwt-authentication-bypass-via-algorithm-confusion-with-no-exposed-key)

![image-20231223182733413](assets/1706247644-7e15ae75df8a5dca3074e90c5a828993.png)

（1）打开靶场，首先我们先正常登录、退出，拿到两个 jwt

eyJraWQiOiI2YTFkYjRlYS1hNGJmLTQ0NzQtYjQxMC0zYzk5NTc3YzJhMWMiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsInN1YiI6IndpZW5lciIsImV4cCI6MTcwMzMyOTc4OX0.BCzT\_VhEaeqfxPoRBYwduIju0AXpmPqJ8HzM-7iqYquyNx2NgPRiJNVvYemspXIPQ8\_-sGr67Qn6lVSHgj51xoNd\_jTJfYO8AlQWF4oAiz2Hfjfng6DN7VoiuJ7vQCMh9VSWnzLzG30leaEzyRjzHGnbFE9EUZ5Hbu7tXOFJU5IwHE35TuU5Xcnv2DXpRDxTsJpHvk5gKQWPx4XLNOY--8LJncBRUoDXD7jXCW0hdY19DPkIDI\_xNKYi27sGkShv8\_zf3G5oSVdChCVgSfdGyCivTtuQ3pAeTl1AwmJll9wy7v4MunWSMgXD\_-SyiCakBNgPaMm\_gWfTlATrrPPT\_g

eyJraWQiOiI2YTFkYjRlYS1hNGJmLTQ0NzQtYjQxMC0zYzk5NTc3YzJhMWMiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsInN1YiI6IndpZW5lciIsImV4cCI6MTcwMzMyOTgxM30.IeXISe59u7Ju0k-NU2RUXORZlY4uKOOpDQC01TSVq35asYsRSUnh7yKS4bwooI6L5wSiZduaYbFNQeg2Na0RDvJx6-wIYR8LlEVQ9U2n7\_8Z7pPkF9QVQDJGF6mUhQAEOS35gBFUOwvYsR2WaayKaOH\_nSOJbFiQjOzF8EykR0LEz5vk2NhMYYMDbbO1LGJ5i2QBtIB5SfTwlZPiy7lK9d\_Une2a0FwmeaoNA\_4dIsiVo4hD3Av-DT\_voCN9pSN-AuoofKqJYwolxHatfUGP4ONuVRwcVScJmvaH2UAh4YI1deRCk62nChBhmBt6TTclGn9xzJX7TeGfqsn6wmWQhA

（2）随后将其放到靶场提供的 docker 工具里面运行，运行的命令如下：

docker run --rm -it portswigger/sig2n <token1> <token2>

![image-20231223181749611](assets/1706247644-35d3e699519675d0f35f9cb6a4f965b4.png)

jwt \_ forgery.py 脚本会输出一系列 token 的存在情况值

![image-20231223181803810](assets/1706247644-f333a87a076f401026f39a67eef0cb97.png)

这里我们尝试每一个 Tempered JWT，不过靶场这里给了提示说是 X.509 形式的，所以我们只需要将 X.509 形式的 JWT 进行验证即可

（3）剩下的步骤结合上一关的一样了，创建恶意的 jwt --> 签名 --> 替换本地 jwt --> 删除`carlos`用户

![image-20231223182245303](assets/1706247644-3b9eb899533523ddb183169565f7c7fb.png)

## 0X08 一些其他的 jwt 安全问题

就拿我之前碰到过的两个漏洞来说明！严格意义上不能说是 jwt 的问题，应该是程序设计的问题，但是跟 jwt 沾了一点边！

### 1、未授权

访问管理后台，/adplanet/PlanetUser 页面下有用户管理功能！

![image-20231223183847582](assets/1706247644-5cbd6c102ba0c9ad0da7e7d716f8156d.png)

我们访问该页面并抓包，发现 api:/square/GetAllSquareUser，用于获取用户信息，包括用户名、密码、邮箱地址等敏感信息！

![img](assets/1706247644-0ded1e4ea12d21296c25c7f9aa95b427.png)

我们把这个数据包放入文本模糊模块中，删除他的 JWT token，然后发送数据包，我们可以发现我们仍然可以获取到用户信息！

![img](assets/1706247644-168c4ff84802e7362486275a3e5d7cd7.png)

### 2、垂直越权

和上边类似，只是另一个不同的功能带点。

还是抓包后删 jwt，但是发现返回 401

![image-20231223184051843](assets/1706247644-927ab8dc90c5d176dc506f92c64e8aea.png)

此时，我们去前台注册一个普通用户获取他的 jwt。携带这个 jwt 发包！

成功垂直越权！

## 0X09 JWT 漏洞的防护

-   使用最新的 JWT 库，虽然最新版本的稳定性有待商榷，但是安全性都是较高的。
-   对 jku 标头进行严格的白名单设置。
-   确保 kid 标头不容易受到通过 header 参数进行目录遍历或 SQL 注入的攻击。

## 0X0A 参考文章

[https://paper.seebug.org/3057/#\_1](https://paper.seebug.org/3057/#_1)

[https://www.freebuf.com/articles/web/337347.html](https://www.freebuf.com/articles/web/337347.html)

[https://www.cnblogs.com/pcheng/p/9629621.html](https://www.cnblogs.com/pcheng/p/9629621.html)

[https://www.cnblogs.com/xiaozi/p/12005929.html](https://www.cnblogs.com/xiaozi/p/12005929.html)

[https://portswigger.net/web-security/jwt/algorithm-confusion](https://portswigger.net/web-security/jwt/algorithm-confusion)

## 0X0B 原文地址

转载自公众号：安服仔 Yu9  
原文地址：[https://mp.weixin.qq.com/s/3rraOTO3Z-n9GzxFco5FOA](https://mp.weixin.qq.com/s/3rraOTO3Z-n9GzxFco5FOA)
