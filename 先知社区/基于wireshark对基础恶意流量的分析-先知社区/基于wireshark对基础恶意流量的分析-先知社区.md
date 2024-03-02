

# 基于 wireshark 对基础恶意流量的分析 - 先知社区

基于 wireshark 对基础恶意流量的分析

- - -

# ***恶意流量分析***

## 1\. 网站流量特征分析

主要分析请求 url，user-agent，请求参数等

### （1）dirsearch

-   介绍：dirsearch 是一个基于 Python 的命令行工具，用于对 web 服务器中的目录和文件进行暴力破解。它通过多线程的方式进行暴力扫描，支持保持连接、多种后缀、生成报告（纯文本、JSON）、启发式检测无效的网页、递归的暴力扫描、支持 HTTP 代理、用户代理随机化、批量处理、请求延迟等功能特点。
-   主要作用：dirsearch 可以用来发现 web 服务器中的目录和文件，并尝试通过猜测文件名或目录名来获取敏感信息。它可以帮助网络安全专业人员或黑客攻击者进行安全漏洞扫描和攻击。
-   主要特征：通过显示过滤器，过滤 http.request 数据流，可以在数据流中看到大量 http GET 请求再对路径以及文件进行爆破，可以发现在 info 中的路径有很明显的顺序，例如 dir 将 a 开头的路径跑完后就开始跑 b 开头的路径如此反复，所以在遇到有类似特征的流量，可以初步判断为 dirsearch 攻击。  
    [![](assets/1699492279-da3adb962e9d5c9f85872cfb0abf9bac.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231107203255-c712c1ba-7d69-1.png)

### (2) sqlmap

-   介绍：SQLMap 是一款开源的自动化 SQL 注入工具，由 Python 写成。它主要功能是扫描、发现并利用给定的 URL 和 SQL 注入漏洞。其广泛的功能和选项包括数据库指纹、枚举、数据库提权、访问目标文件系统，并在获取操作权限时执行任意命令。SQLMap 完全支持 MySQL、Oracle、PostgreSQL、MSSQL、Access、IBM DB2、SQLite、Firebird、Sybase、SAP MaxDB、HSQLDB 和 Informix 等多种数据库管理系统。
-   主要特征：依然是通过 wireshark 的显示过滤器过滤 sqlmap 攻击流量包 过滤 http.request 后，可以发现 info 中有大量的 sql 语句。  
    [![](assets/1699492279-6b5a34373bc3bcef06973e49606b3c00.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231107205626-0fe0c2e0-7d6d-1.png)  
    如果攻击者没有设置 sqlmap 工具，在传输层的请求头中的 user-agent 可以看到 sqlmap 的网站，这是一个强特征，只要有这个特征就可以断定是 sqlmap  
    [![](assets/1699492279-a90c0a4879c0b890afdf9121f065d515.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231108111858-8e4fe156-7de5-1.png)  
    如果需要过滤 sqlmap 的数据流 使用 http.user\_agent contains sqlmap 可以将包含 sql 的数据流过滤出来，进行下一步分析。  
    \## 2.漏洞扫描器流量特征分析  
    \### （1）AWVS
-   介绍：一个用于测试和管理 Web 应用程序安全性的平台。它能够自动扫描互联网或者本地局域网中是否存在漏洞，并报告漏洞。AWVS 可以扫描任何通过 Web 浏览器访问和遵循 HTTP/HTTPS 规则的 Web 站点，适用于任何中小型和大型企业的内联网、外延网和面向客户、雇员、厂商和其它人员的 Web 网站。AWVS 可以通过检查 SQL 注入攻击漏洞、XSS 跨站脚本攻击漏洞等漏洞来审核 Web 应用程序的安全性。
-   特征：参数、ua、content\_type 中含有 test、testing、wvs、acunetix\_wvs\_security\_test、acunetix，acunetix\_wvs，其中主要是 test、testing 比较常见。  
    使用显示过滤器，过滤 awvs 攻击流量包
    
    ```plain
    http.content_type contains "test（替换上述特征）"
    ```
    
    可以发现在 http 传输层协议中，POST 请求下属的 Content\_type 包含 testing  
    [![](assets/1699492279-5302b4c2e4d1aaef7807c77b82d47697.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231107211739-068665f8-7d70-1.png)  
    或者使用
    
    ```plain
    http.request.uri.query.parameter contains "test"
    ```
    
    过滤后，可以在 post info 请求中看到 name=testingXXXX 的特征信息

### （2）namp

-   介绍：Nmap 是一款针对大型网络的端口扫描工具，它最初设计目的是为了快速扫描大型网络。Nmap 支持多种协议的扫描，如 UDP、TCP、ICMP 等，同时支持性能和可靠性统计，例如动态延时计算、数据包超时和转发、并行端口扫描、通过并行 ping 侦测下层主机等功能。
-   主要特征：使用显示过滤器过滤 namp 流量包
    
    ```plain
    tcp && ip.dst == [namp 扫描的 ip]
    ```
    
    可以发现有大量的 tcp 包 在对目标 ip 进行端口访问  
    [![](assets/1699492279-9f0433a5c5176e79c5d22bf2dbf6aaf2.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231108114008-838aefec-7de8-1.png)

## 3.webshell 管理工具流量特征分析

### （1）蚁剑

-   介绍：蚁剑（AntSword）是一款开源的跨平台 WebShell 管理工具，主要面向合法授权的渗透测试安全人员以及进行常规操作的网站管理员。它推崇模块化的开发思想，遵循开源，致力于为不同层次的人群提供最简单易懂、方便直接的代码展示及其修改说明。
-   主要特征：蚁剑是明文传输（即使使用了加密与编码，也会有包传输密码协商过程，该过程也会有明文存在）即：在 post 包中的 html 可以看到传入的参数 观察多个 POST 包中可以发现 html 下属的 form item 都有一个 display\_errors  
    [![](assets/1699492279-75c269f6bf8bf502a86e07cb23b35859.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231108121202-f7f8f3e8-7dec-1.png)  
    右键，选择作为过滤器应用中选中  
    [![](assets/1699492279-fc5037846e789f1adf982738fb2222dc.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231108121845-e8867308-7ded-1.png)  
    将显示过滤器中多余的信息删去，只留下 urlencoded-form.value contains "display\_errors"  
    [![](assets/1699492279-041d07f9012512662a8e601a548f1063.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231108123344-0043569e-7df0-1.png)  
    然后在追踪 http 通过 url 解密，就可以实现分析攻击者的操作，溯源等操作  
    [![](assets/1699492279-1bae78a650893774bd6bcb2505557e41.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231108123446-250aa9fa-7df0-1.png)  
    \### （2）冰蝎
-   介绍：“冰蝎”是一个动态二进制加密的网站管理客户端，基于 JAVA 开发，能够跨平台使用。最新版本为 v2.0.1，兼容性较之前的版本有较大提升。它的主要功能包括基本信息、命令执行、虚拟终端、文件管理、Socks 代理、反弹 shell、数据库管理、自定义代码等，非常强大。
-   主要特征：冰蝎 2.0，3.0，4.0 版本之间各有差异  
    例如 4.0 的特征：  
    在 4.0 更新的源代码中，定义了 14 个 user-agent
    
    ```plain
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 11_2_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/89.0.4389.114 Safari/537.36",
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:87.0) Gecko/20100101 Firefox/87.0",
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/96.0.4664.110 Safari/537.36",
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36 Edg/99.0.1150.55",
    "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/96.0.4664.110 Safari/537.36",
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:98.0) Gecko/20100101 Firefox/98.0",
    "Mozilla/5.0 (Windows NT 10.0) AppleWebKit/537.36 (KHTML, like Gecko)
    Chrome/84.0.4147.125 Safari/537.36",
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko)
    Chrome/84.0.4147.125 Safari/537.36",
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:79.0) Gecko/20100101 Firefox/79.0",
    "Mozilla/5.0 (Windows NT 6.3; Trident/7.0; rv:11.0) like Gecko"
    ```
    
    大多数情况为第一个  
    1.观察冰蝎攻击流量包的 POST 请求包，可以在 user-agent 中看到上诉特征  
    [![](assets/1699492279-72b9bb02fcae8df8a0283764e3707ea8.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231108125048-62b36a06-7df2-1.png)  
    如果需要拦截这个攻击，可以在防火墙中拦截 Chrome/89.0.4389.114 这个版本的流量 因为现在 chrome 更新到 100.xx 的版本了 只要攻击者或者极少用户使用，影响不大  
    [![](assets/1699492279-52870782a88a5518df28e1332b54f846.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231108130115-d82ba9be-7df3-1.png)  
    2.请求体头部字节与响应头部字节不会变化  
    [![](assets/1699492279-9d59a4fae1f547bd7cc8aabcd35f281b.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231108130234-0785b448-7df4-1.png)  
    [![](assets/1699492279-330133e1a3b719cca3c299b6fbcf61e2.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231108130245-0df53a24-7df4-1.png)  
    可以发现他们请求体中的 key 值开头一直不变 这是在实际情况是不可能的 所以在看到此特征的流量可以直接判断为冰蝎。  
    \### （3）哥斯拉
-   介绍：哥斯拉是一款由 Java 语言开发的 webshell 权限管理工具，功能强大，能绕过市面大部分的静态查杀和流量加密，不受大部分的流量 Waf 限制，自带的插件冰蝎和蚁剑无法比拟。它不仅可以实现传统的命令执行、文件管理、数据库管理等功能，还可以进行 MSF 联动、绕过 OpenBasedir、ZIP 压缩和解压、代码执行、绕过 DisableFunctions 等操作。
-   主要特征：哥斯拉客户端与 shell 建连初期的三个固定行为特征，且顺序出现在同一个 TCP 连接中。可以总结为：  
    特征：发送一段固定代码（payload），http 响应为空  
    特征：发送一段固定代码（test），执行结果为固定内容  
    特征：发送一段固定代码（getBacisInfo）  
    先发送一段 pass 的加密密文  
    [![](assets/1699492279-b401749dd405f65691fa3e3dd7d192f3.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231108132550-474582f4-7df7-1.png)  
    然后目标服务器会返回一串空回复  
    [![](assets/1699492279-227491a88bfdb359d0ac86f2daaa58ca.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231108132622-5a7b4cbe-7df7-1.png)  
    在通过交互 2 次 返回固定值 密文  
    [![](assets/1699492279-01262b2b26b391f177bebe6ededa8ba8.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231108132648-6a48e46c-7df7-1.png)  
    哥斯拉的响应体中有一个特征是前 16 位和后 16 为会组成一个 32 位 md5 正则匹配类似于 (?i:\[0-9A-F\]{16})\[\\w+/\]{4,}=?=?(?i:\[0-9A-F\]{16})  
    可以在返回包中的密文发现 前 16 和后 16 为一个 32 位的 MD5 值  
    [![](assets/1699492279-fdd3971439f625da1b0274a955f96356.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231108132833-a888bdd8-7df7-1.png)

## 4.常见 C2 远控服务器流量特征分析

### (1) metasploit

-   介绍：Metasploit 是一款开源的网络安全评估工具，它被广泛用于发现、验证和利用网络漏洞。这个工具是由 Rapid7 开发的，包括 Metasploit Pro、Metasploit Express 和 Metasploit Community Edition 等版本。
-   主要特征：meterpreter 命令执行的过程已经进行了加密处理，继续进行活动，防御者也不会发现有价值的数据，但可以观察到每个数据都包含 MZ 标头和 DOS 模式异常。  
    metasploit 是使用 tcp 协议，使用显示过滤器，过滤 metasploit 流量包  
    [![](assets/1699492279-6f662182e3ed645819e6c6bd7d059a5b.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231108133407-6ff9a526-7df8-1.png)  
    可以发现有 2 个端口一直在交互 9999 和 57591 可以初步判断这个为远控后门，追踪下 tcp 的数据流，发现有明显特征  
    [![](assets/1699492279-9dba96c5b8cf166dfc33db970244a78c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231108133824-08bc7324-7df9-1.png)  
    连续追踪多个 tcp 包都有这个特征，所以当我们遇到该特征的流量，可以判断为 metasploit 流量。

### （2）cobaltstrike

-   介绍：Cobalt Strike 提供了一套功能强大的工具和框架，用于模拟网络攻击、横向移动、特权升级、持久化访问以及命令和控制等任务。它旨在帮助安全专业人员评估和改善组织的网络防御能力，同时提供了完整的红队操作平台。Cobalt Strike 的核心功能包括木马部署与远控、模拟攻击、内网渗透和横向移动、网络侦察、命令和控制以及报告和日志记录。
-   主要特征：  
    (1）cs 使用的 http 协议 cs 流量包的第一个 GET 请求 有一个随机编码 IPYK 转换为 ascii 之和与 256 取余计算值等于 92，下载 stage payload 的过程 uri 符合 checksum8 规则，即：路径的 ascii 之和与 256 取余计算值等于 92  
    [![](assets/1699492279-99f3e276bba03de55e3199c34a96a4bd.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231108135055-c8616f9e-7dfa-1.png)  
    (2) c2 服务器如果没有任务下发，会有规律的请求响应间隔，用于维持连接，比如这里是每间隔 3s，发一次包（间隔时间可以在 cs 内设置）有些响应在 4s，是因为网络有延迟不影响  
    [![](assets/1699492279-dbe0778c4193d3d09781cee3d6a9cf10.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231108135541-72d5f40e-7dfb-1.png)  
    如果 cs 服务器有任务下发，则会加密放在 http 心跳包的 cookie 里面，下发后，靶机完成指令，也同样会返回一个 post 包都加密并隐藏在 cookie 中。

# 总结

以上为根据课程等总结，大多都是在 POST 和 GET 请求中，还有很多流量特征需要大家发现，大家可以讨论讨论，如有错误或者更好的方法，请指出。
