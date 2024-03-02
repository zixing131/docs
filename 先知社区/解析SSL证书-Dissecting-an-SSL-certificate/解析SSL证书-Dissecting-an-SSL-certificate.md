

# 解析 SSL 证书 --- Dissecting an SSL certificate

# Dissecting an SSL certificate  
解析 SSL 证书

Hello! In my networking zine (which everyone will be able to see soon), there is a page about TLS/SSL (basically [this tweet](https://twitter.com/b0rk/status/809594614147645440)). But as happens when you write 200 words about a thing on a page, there is a lot more interesting stuff to say. So in this post we will dissect an SSL certificates and try to understand it!  
你好啊！在我的网络杂志（每个人很快就能看到）中，有一个关于 TLS/SSL 的页面（基本上就是这个 tweet）。但是当你在一页纸上写 200 字关于一件事的时候，有很多更有趣的东西要说。因此，在这篇文章中，我们将剖析 SSL 证书，并试图了解它！

I am not a security person and I am not going to give you security advice for your website (want to know what TLS ciphers you should use? I have no idea!!!). But! I think it’s interesting to know what it means to “issue a SSL certificate” and I can talk about that a little.  
我不是一个安全的人，我不会给你给予安全建议，为您的网站（想知道什么 TLS 密码，你应该使用？我不知道！）。但是，我...我想知道“颁发 SSL 证书”是什么意思是很有趣的，我可以谈一点。

### TLS: newer version of SSL  
TLS：SSL 的新版本

I was confused about what this “TLS” thing was for a long time. Basically newer versions of SSL are called TLS (the version after SSL 3.0 is TLS 1.0). I’m going to just call it “SSL” throughout because that is less confusing to me.  
我很长一段时间都不知道“TLS”是什么。基本上，SSL 的新版本被称为 TLS（SSL 3.0 之后的版本是 TLS 1.0）。我将始终称之为“SSL”，因为这对我来说不那么令人困惑。

### What’s a certificate? 什么是证书？

Suppose I’m checking my email at [https://mail.google.com](https://mail.google.com/)  
假设我 https://mail.google.com

`mail.google.com` is running a HTTPS server on port 443. I want to make sure that I’m **actually** talking to mail.google.com and not some other random server on the internet owned by EVIL PEOPLE.  
`mail.google.com` 正在端口 443 上运行 HTTPS 服务器。我想确保我 mail.google.com

This “certificate” business was kind of mysterious to me for a very long time. One day my cool coworker Ray told me that I could connect to a server on the command line and download its certificate!  
这种“证书”业务对我来说很长一段时间都很神秘。有一天，我的酷同事 Ray 告诉我，我可以通过命令行连接到服务器并下载它的证书！

(If you want to just look at an SSL certificate, you can click on the green lock in your browser and reliably get all the information you need. But this is more fun.)  
(If 如果您只想查看 SSL 证书，您可以点击浏览器中的绿色锁，并可靠地获取所需的所有信息。但这更有趣）。

So, let’s start by looking at mail.google.com’s certificate and deconstruct it a bit.  
那么，让我们先来看看 mail.google.com 的证书，并对其进行一点解构。

First, we run `openssl s_client -connect mail.google.com:443` 首先，我们运行 `openssl s_client -connect mail.google.com:443`

This is going to print a bunch of stuff, but we’ll just focus on the certificate. Here, it’s this thing:  
这将打印一堆东西，但我们只关注证书。在这里，它是这样的：

```plain
$ openssl s_client -connect mail.google.com:443
...
-----BEGIN CERTIFICATE-----
MIIElDCCA3ygAwIBAgIIMmzfdZnO9pMwDQYJKoZIhvcNAQELBQAwSTELMAkGA1UE
BhMCVVMxEzARBgNVBAoTCkdvb2dsZSBJbmMxJTAjBgNVBAMTHEdvb2dsZSBJbnRl
cm5ldCBBdXRob3JpdHkgRzIwHhcNMTcwMTE4MTg1MjExWhcNMTcwNDEyMTg1MDAw
WjBpMQswCQYDVQQGEwJVUzETMBEGA1UECAwKQ2FsaWZvcm5pYTEWMBQGA1UEBwwN
TW91bnRhaW4gVmlldzETMBEGA1UECgwKR29vZ2xlIEluYzEYMBYGA1UEAwwPbWFp
bC5nb29nbGUuY29tMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAiYcr
C9Rn7g9xjsg7khqfRPxUnvpgGyCHqJMXxZGtdf+G02d07cPlMEeaGG12vHyVfRZD
tc/F1ZfwenH6gf0uMobtgw7n2NQa7T7qxuqSUDhZsO1sI1LL/Yqy8QHoooOZQWMz
ytuRA18zti4vQV1dCijADh0+NWI1GDUAKidbaH/fBRrStqBev5Bhq3ZaGj3fDjAO
7CG0Wk3n4Ov2yg44XOdgkLMzjdnbV8l6cZDC7lCK1VsEU1mEd0O0Dw4OcnHLuBPw
IkioZayhPOXDXUS+bhpmtEiCkt8kbHG6jNMC4m8t62Jaf/Si3XNcHhDa4wPCTvid
X//PuuNlRZVg3NjK/wIDAQABo4IBXjCCAVowHQYDVR0lBBYwFAYIKwYBBQUHAwEG
CCsGAQUFBwMCMCwGA1UdEQQlMCOCD21haWwuZ29vZ2xlLmNvbYIQaW5ib3guZ29v
Z2xlLmNvbTBoBggrBgEFBQcBAQRcMFowKwYIKwYBBQUHMAKGH2h0dHA6Ly9wa2ku
Z29vZ2xlLmNvbS9HSUFHMi5jcnQwKwYIKwYBBQUHMAGGH2h0dHA6Ly9jbGllbnRz
MS5nb29nbGUuY29tL29jc3AwHQYDVR0OBBYEFI69aYCEtb2swbJJR3cMOTdcfvZ4
MAwGA1UdEwEB/wQCMAAwHwYDVR0jBBgwFoAUSt0GFhu89mi1dvWBtrtiGrpagS8w
IQYDVR0gBBowGDAMBgorBgEEAdZ5AgUBMAgGBmeBDAECAjAwBgNVHR8EKTAnMCWg
I6Ahhh9odHRwOi8vcGtpLmdvb2dsZS5jb20vR0lBRzIuY3JsMA0GCSqGSIb3DQEB
CwUAA4IBAQAhiqQIwkGp1NmlLq89gjoAfpwaapHuRixxl2S54fyu/4WOHJJafqVA
Tya9J7GTUCyQ6nszCdVizVP26h9TKOs9LJw5jWV9SOnPU2UZKvrNnOUi2FUkCcuD
lsADdKSXNzye3jB88TENrWC/y3ysPdAgPO/sXzyRvNw8SVKl2+RqMDpSRpBptF9e
Lp+WLAM3xKS5SPwCNdCiA332o7qiKRKQm/6bbIWnm7hp/ZnLxbyKaIVytRdiwRNp
O/TTpRv2C708GA3PH6i1pYE86xm3w7lGhN9OiCZpKOJD6ZUH3W20idgPKYPBCO/N
Op2AF3I4iUGeQjXFVLgS6mjUvdLndL9G
-----END CERTIFICATE-----
```

So far, this is unintelligible nonsense. “MIIElDcca… WHAT?!”  
到目前为止，这是无法理解的废话。“什么？

It turns out that this nonsense is a format called “X509”, and the `openssl` command knows how to decode it.  
事实证明，这种无意义的格式称为“X509”，并且 `openssl` 命令知道如何解码它。

So I saved this blob of text to a file called `cert.pem`. You can save it to your computer [from this gist](https://gist.githubusercontent.com/jvns/2c249b8059c0b183c02bb3a71e12e418/raw/b4afd1877471b72f8900b2396c30bc670a5701c9/mail_google_cert.pem) if you want to follow along.  
所以我把这个文本块保存到一个名为 `cert.pem` 的文件中。你可以保存到您的计算机从这个要点，如果你想遵循沿着。

Our next mission is to **parse this certificate**. We do that like this:  
我们的下一个使命是解析这个证书。我们这样做：

```plain
$ openssl x509 -in cert.pem -text

Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 3633524695565792915 (0x326cdf7599cef693)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=US, O=Google Inc, CN=Google Internet Authority G2
        Validity
            Not Before: Jan 18 18:52:11 2017 GMT
            Not After : Apr 12 18:50:00 2017 GMT
        Subject: C=US, ST=California, L=Mountain View, O=Google Inc, CN=mail.google.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:89:87:2b:0b:d4:67:ee:0f:71:8e:c8:3b:92:1a:
                    9f:44:fc:54:9e:fa:60:1b:20:87:a8:93:17:c5:91:
                    .... blah blah blah ............
                    c2:4e:f8:9d:5f:ff:cf:ba:e3:65:45:95:60:dc:d8:
                    ca:ff
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Alternative Name: 
                DNS:mail.google.com, DNS:inbox.google.com

            X509v3 Subject Key Identifier: 
                8E:BD:69:80:84:B5:BD:AC:C1:B2:49:47:77:0C:39:37:5C:7E:F6:78
    Signature Algorithm: sha256WithRSAEncryption
         21:8a:a4:08:c2:41:a9:d4:d9:a5:2e:af:3d:82:3a:00:7e:9c:
         1a:6a:91:ee:46:2c:71:97:64:b9:e1:fc:ae:ff:85:8e:1c:92:
         ......... blah blah blah more goes here ...........
```

This is a lot of stuff. Here are the parts of this that I understand  
这东西太多了以下是我理解的部分

-   `CN=mail.google.com` is the “common name”. Counterintuitively you should ignore this field and look at the “subject alternative name” field instead  
    `CN=mail.google.com` 是“通用名”。与直觉相反，您应该忽略此字段，转而查看“subject alternative name”字段
-   an **expiry date**: Apr 12 18:50:00 2017 GMT  
    失效日期：2017 年 4 月 12 日 18:50:00 GMT
-   The `X509v3 Subject Alternative Name:` section has the list of domains that this certificate works for. This is mail.google.com and inbox.google.com, which makes sense – they’re both email domains.  
    `X509v3 Subject Alternative Name:` 部分包含此证书适用的域列表。这是 mail.google.com 和 inbox.google.com，这是有意义的 - 它们都是电子邮件域。
-   The `Public Key Info` section tells us the **public key** that we’re going to use to communicate with mail.google.com. We do not have time to explain public key cryptography right now, but this is basically the encryption key we’re going to use to talk secretly.  
    `Public Key Info` 部分告诉我们将用于与 mail.google.com 通信的公钥。我们现在没有时间来解释公钥密码学，但这基本上是我们要用来秘密交谈的加密密钥。
-   Lastly, the **signature** is a really important thing. Basically anyone could make a certificate for mail.google.com. I could make one right now! But if I gave you that certificate, you would have no reason to believe that it is a real certificate  
    最后，签名是一件非常重要的事情。基本上任何人都可以为 mail.google.com 制作证书。我现在就可以做一个！但如果我给你那张证书，你就没有理由相信它是一张真实的证书

So let’s talk about certificate signing.  
让我们来谈谈证书签名。

### certificate signing 证书签名

Every certificate on the internet is basically two parts put together  
互联网上的每个证书基本上都是两个部分放在一起

1.  A certificate (the domain name it’s valid for and public key and other stuff)  
    证书（它的有效域名和公钥和其他东西）
2.  A **signature** by someone else. This basically says, “hey, this is okay, Visa says so”  
    别人的签名。这基本上是说，“嘿，这是好的，签证说，”

I have a bunch of certificates on my computer in /etc/ssl/certs. Those are the certificates my computer trusts to sign other certificates. For example, I have `/etc/ssl/certs/Staat_der_Nederlanden_EV_Root_CA.pem` on my laptop. Some certificate from the Netherlands! Who knows! If they signed a `mail.google.com` certificate, my computer would be like “yep, looks great, sounds awesome”.  
我的计算机上有一堆证书，在/etc/ssl/certs 中。这些证书是我的计算机信任的证书，可用于签署其他证书。例如，我的笔记本电脑上有 `/etc/ssl/certs/Staat_der_Nederlanden_EV_Root_CA.pem` 。来自荷兰的证书！谁知道呢！如果他们签署了一个 `mail.google.com` 证书，我的电脑会像“是的，看起来很棒，听起来很棒”。

If some random person across the street signed a certificate, my computer would be like “I have no idea who you are”, and reject the certificate.  
如果街对面的某个随机的人签署了一份证书，我的计算机就会像“我不知道你是谁”一样，拒绝证书。

The mail.google certificate is  
mail.google 证书是

-   s:/C=US/ST=California/L=Mountain View/O=Google Inc/CN=mail.google.com  
    s：/C=US/ST=加州/L=山景/O=谷歌公司/CN= mail.google.com
-   which is signed by a “Google Internet Authority G2” certificate  
    由“Google Internet Authority G2”证书签名
-   which is signed by a “GeoTrust Global CA” certificate  
    由“GeoTrust Global CA”证书签名
-   which is signed by an “Equifax Secure Certificate Authority” certificate  
    由 Equifax Secure Certificate Authority 认证

I have an /etc/ssl/certs/GeoTrust\_Global\_CA.pem file on my computer, which I think is why I trust this mail.google.com certificate. (Geotrust signed Google’s certificate, and Google signed mail.google.com)  
我的计算机上有一个/etc/ssl/certs/GeoTrust\_Global\_CA.pem 文件，我想这就是我信任此 mail.google.com 证书的原因。（Geotrust 签署了谷歌 mail.google.com

### what does getting a certificate issued look like?  
获得证书是什么样的？

So when you get a certificate issued, basically how it works is:  
所以当你得到一个证书颁发，基本上它是如何工作的：

1.  You generate the first half of the certificate (“jvns.ca! expires on X date! This is my public key!”). This part is public.  
    生成证书的前半部分（“jvns.ca！在 X 日期过期！这是我的公钥！”）。这部分是公开的。
2.  At the same time, you generate a **private key** for you certificate. You keep this very secret and safe and do not show it to anybody. You’ll use this key every time you establish an SSL connection.  
    与此同时，您为证书生成私钥。你要保守秘密，不要给任何人看。每次建立 SSL 连接时都将使用此密钥。
3.  You pay a **certificate authority** (CA) that other computers trust to sign your certificate for you. Certificate authorities are supposed to have integrity, so they are supposed to actually make sure that when they sign certificates, the person they give the cert to actually owns the domain.  
    您向其他计算机信任的证书颁发机构（CA）付款，让其为您签署证书。证书颁发机构应该具有完整性，因此他们应该确保在签署证书时，他们给予证书的人实际上拥有该域。
4.  You configure your website with your signed certificate and use it to prove that you are really you! Success!  
    您可以使用您的签名证书配置您的网站，并使用它来证明您真的是您！成功啦！  
    

This “certificate authorities are supposed to have integrity thing” I think is why people get so mad when there are

[↓↓↓](https://www.symantec.com/page.jsp?id=test-certs-update)  
  
issues like this with Symantec  
  
[↑↑↑](https://www.symantec.com/page.jsp?id=test-certs-update)

where they generated test certificates “to unregistered domains and domains for which Symantec did not have authorization from the domain owner”  
这种“证书颁发机构应该有完整性的东西”，我认为这就是为什么人们得到如此疯狂时，有这样的问题与赛门铁克在那里他们生成测试证书“未注册的域和域，赛门铁克没有从域所有者的授权”

### certificate transparency  
证书透明度

The last thing we are going to talk about is [certificate transparency](https://www.certificate-transparency.org/what-is-ct). This is a super interesting thing and has a good website and I am almost certainly going to mangle it.  
最后我们要讨论的是证书透明性。这是一个超级有趣的事情，有一个很好的网站，我几乎肯定会毁了它。

I will try anyway!  
无论如何我会试试！

So, we said that certificate authorities are “supposed to have integrity”. But there are SO MANY certificate authorities that my computer trusts! And at any time one of them could sign a rogue certificate for mail.google.com. That’s no good.  
因此，我们说证书颁发机构“应该具有完整性”。但是我的计算机信任的证书颁发机构太多了！在任何时候，他们中的一个都可以为 mail.google.com 签署一个流氓证书。这可不好

This isn’t a hypothetical issue – the [certificate transparency](https://www.certificate-transparency.org/what-is-ct) website talks about more than one instance where a CA has been compromised or otherwise has made a mistake.  
这不是一个假设性的问题 - 证书透明度网站讨论了不止一个 CA 被泄露或犯了错误的情况。

So, here’s the deal. At any given time, Google **knows** all the valid certificates that are supposed to exist for `mail.google.com` (there is probably only one or something). So certificate transparency is basically a way to make sure that if there is a certificate in circulation for mail.google.com that they DON’T know about, that they can find out.  
所以，是这样的。在任何时候，Google 都知道 `mail.google.com` 应该存在的所有有效证书（可能只有一个或其他东西）。因此，证书透明度基本上是一种确保如果有一个证书在流通的 mail.google.com，他们不知道，他们可以找到。

Here are the steps, as I understand them  
以下是我理解的步骤

1.  Every time any CA signs a certificate, they are supposed to put into a global public “certificate log”  
    每次任何 CA 签署证书时，都应该将其放入全局公共“证书日志”中
2.  Also the Googlebot puts every certificate it finds on the internet into the certificate log  
    Googlebot 也会将它在互联网上找到的每一个证书放入证书日志中
3.  If a certificate **isn’t** in the log, then my browser will not accept it (or will stop accepting it in the future or something)  
    如果证书不在日志中，那么我的浏览器将不会接受它（或者将来会停止接受它）
4.  Anyone can look at the log at any time to find out if there are rogue certificates in there  
    任何人都可以在任何时候查看日志，以发现其中是否存在流氓证书

So if that CA in the Netherlands signs an evil mail.google.com certificate, they either have to put it in the public log (and Google will find out about their evil ways) or leave it out of the public log (and browsers will reject it).  
因此，如果荷兰的 CA 签署了一个邪恶的 mail.google.com 证书，他们要么把它放在公共日志中（谷歌会发现他们的邪恶方式），要么把它从公共日志中删除（浏览器会拒绝它）。

### setting up SSL stuff is hard  
设置 SSL 很难

Okay! We have downloaded a SSL certificate and dissected it and learned a few things about it. Hopefully some of you have learned something!  
好的，我会的我们已经下载了一个 SSL 证书，并对其进行了剖析，了解了一些有关它的事情。希望你们中的一些人已经学到了一些东西！

Picking the right settings for your SSL certificates and SSL configuration on your webserver is confusing. As far as I understand it there are about 3 billion settings. [Here is an example of an SSL Labs result for mail.google.com](https://www.ssllabs.com/ssltest/analyze.html?d=mail.google.com&s=216.58.194.165&hideResults=on). There is all this stuff like `OLD_TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256` on that page (for real, that is a real thing.). I’m happy there are tools like SSL Labs that help mortals make sense of all of it.  
在 Web 服务器上为 SSL 证书和 SSL 配置选择正确的设置是令人困惑的。据我所知，大约有 30 亿个设置。下面是 mail.google.com 的 SSL 实验室结果示例。在那一页上有所有像#0 #这样的东西（对于真实的来说，那是一个真实的东西。）。我很高兴有像 SSL 实验室这样的工具可以帮助凡人理解这一切。

Someone told me [https://cipherli.st/](https://cipherli.st/) is a way to pick secure SSL configuration if you’re not sure what to do. I don’t know if it’s good or not.  
有人告诉我，如果你不确定该怎么做，https://cipherli.st/是一种选择安全 SSL 配置的方法。我不知道它好不好。

### let’s encrypt is amazing  
让我们加密是惊人

Also [let’s encrypt](https://letsencrypt.org/) is really cool! They let you have a certificate for your site and make it secure, and you don’t even need to understand all this stuff about how certificates work on the inside! And it’s FREE.  
也让我们加密是真的很酷！他们让你有一个证书为您的网站，使其安全，你甚至不需要了解所有这些东西如何证书的工作在内部！而且是免费的。

Want a weekly digest of this blog?  Subscribe

[New zine: "Networking! ACK!"](https://jvns.ca/blog/networking-zine-launch/ "Previous Post: New zine: "Networking! ACK!"") [A magical machine learning art tool](https://jvns.ca/blog/2017/02/02/a-magical-machine-learning-tool/ "Next Post: A magical machine learning art tool")
