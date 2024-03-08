

# 记一次 JA3 指纹识别的经历

最近在写一个小工具，但是遇到一个问题而引发的研究。

在开发 Python 工具的过程中遇到这样一个问题：开发的程序放在 docker 中运行，使用`requests`模块请求一个`https`服务，但是总是能触发目标服务的验证机制，会要求我输入图片验证码进行验证。

可以看出，目标服务器能检测出请求客户端是一个 bot，而我并不想让目标服务检测出来，每次请求都要输入验证码，在我看来是一件很麻烦的事。

接着，我就开始研究是哪个环节被服务器检测出来有问题。

环境如下：

-   A: 平常使用的 Mbp 笔记本
    
-   B: 装了 docker 的 Linux 主机
    
-   C: 运行 python 工具的 docker 环境
    

对于上述的三个环境，确保它们处在同一个内网中，这样外网的服务器没法通过 IP 地址来进行区分。

然后进行以下测试：

1.  在 A 环境中使用浏览器访问目标，并不会触发验证码。说明目标服务器并没有把测试环境中的外网 IP 地址放入黑名单中。
2.  在 A 环境中启动 Burp Suite，在 C 环境中加上环境变量`HTTP_PROXY/HTTPS_PROXY`确保 C 环境中运行程序产生的流量能被代理到 Burp 中。在 Burp 中查看流量发现，不会触发验证码机制。
3.  在 C 环境中运行脚本时加上环境变量`SSLKEYLOGFILE`，然后在网关使用 tcpdump 抓包，会触发验证码机制。随后在本地使用 wireshark 对解密出的 HTTPS 流量进行分析。
4.  把 A 和 C 环境中的程序放到 B 环境中运行，不会触发验证码机制。

针对 2，3，4 步骤的结果进行分析：

-   只有在 docker 环境中运行程序才会触发验证码机制。
-   目标服务器并没有对 HTTP 数据包进行检测，并不是 HTTP 数据包中多了或者少了某段数据导致被目标服务器检测出来的。
-   因为 A, B, C 环境中的 python 版本不一样，所以有可能是 docker 环境中的 python 或者其相关库 (比如 requests) 在流量中多了某些数据导致被检测出来。

5.  把 C 环境中的 requests/urllib3 库的版本和 B 环境中的版本一致，在 C 环境中运行程序仍然会触发验证码。
    
6.  在 C 环境中把程序中使用 requests 发送的流量提取出来，使用 curl 想目标服务发送数据包，不会触发验证码。
    

测试到这，发现只有在 C 环境中使用 python 的 requests 发送流量才会被目标服务器检测出来，这个时候因为知识的缺失，只能猜测可能在 SSL 握手环节进行了检测，但是却不知是如何检测的。

随后在协会的大哥帮助下知道了 JA3 指纹[\[1\]](#jump1)。

简单的说就是对 SSL 握手的 Client Hello 流量包的一些不随机的字段进行 hash 签名，然后可以做一个 JA3 指纹库，这样在一定程度上可以通过 JA3 指纹得知客户端的 UA。

一般情况下，Web 服务器想要知道客户端的什么设备，都是通过 HTTP 头的`User-Agent`字段，但是众所周知，对该字段进行伪造是轻而易举的事。所以出现了 JA3 指纹，默认情况下，如果不对 SSL 的参数进行特殊设置，那么 JA3 指纹就是固定的。

比如使用 Python 的 requests 请求一个 HTTPS 服务，正常情况下开发人员只会对 HTTP 请求进行操作，并不会修改 ssl 的参数，最多设置一个`verify=False`。这样`requests`发送的`Client Hello`包中的非随机字段都是固定的，从而计算出的 JA3 指纹都是固定的。

可以使用下面的代码获取到 requests 的 JA3 指纹：

[](javascript: "Copy")

```body
import requests
url2 = "https://tools.scrapfly.io/api/fp/ja3"
res = requests.get(url2, headers={"origin": "https://scrapfly.io"})
print(res.json()["ja3_digest"])
```

可以使用下面的代码获取到 curl 的 JA3 指纹：

[](javascript: "Copy")

```body
$ curl 'https://tools.scrapfly.io/api/fp/ja3' -H 'origin: https://scrapfly.io'
$ curl --ciphers ECDHE-RSA-AES128-GCM-SHA256  'https://tools.scrapfly.io/api/fp/ja3' -H 'origin: https://scrapfly.io'
# 通过上面两个请求的结果可以发现，我们能很容易的让JA3指纹的结果产生变化。
```

在浏览器上可以访问`https://scrapfly.io/web-scraping-tools/ja3-fingerprint`获取到浏览器的`JA3`指纹。

经过研究发现，Chrome 的`JA3`指纹是随机变化的，但是使用 Chrome 访问目标服务器却不会触发验证码。从这可以猜测，目标服务器采用的是 JA3 黑名单。

而黑名单是最好绕过的，想把客户端的 JA3 指纹设置成指定值是有难度的，但是让客户端的 JA3 指纹不等于某个值是非常容易的。只要对 ssl 的 ciphers 参数进行任何变动，都会导致 JA3 指纹发现变化。

有以下两个简单的绕过方案：

1.  docker 换一个 python 版本，比如原本使用的是`FROM python:3.10-slim-buster`，现在换成`FROM python:3.11-slim-buster`。
2.  修改`requests.packages.urllib3.util.ssl_.DEFAULT_CIPHERS`的值。

不过经过一天的测试发现，第一个方案并不靠谱，目标服务器还有其他的检测方案，过了一天`3.11`的指纹也进入到黑名单中。所以考虑写一个函数，让`DEFAULT_CIPHERS`的值每次请求都发现变化，就像 Chrome 一样，每次 JA3 值都不一样。

# [](#%E6%80%BB%E7%BB%93 "总结")总结

通过以上研究可以发现，黑名单机制并不靠谱。

# [](#%E5%8F%82%E8%80%83%E9%93%BE%E6%8E%A5 "参考链接")参考链接

1.  [https://github.com/yolossn/JA3-Fingerprint-Introduction](https://github.com/yolossn/JA3-Fingerprint-Introduction)
