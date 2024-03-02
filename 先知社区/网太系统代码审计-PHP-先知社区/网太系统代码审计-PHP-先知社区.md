

# 网太系统代码审计-PHP - 先知社区

网太系统代码审计-PHP

- - -

### 0x00 前言

网太 CMS PHP 版 基于 PHP+mysql 的技术架构，php 环境版本支持 php 5.3 到 php 7.3，建议用 php 7.1~7.3。

**源码下载：**  
[http://d.otcms.com/php/OTCMS\_PHP\_V7.16.zip](http://d.otcms.com/php/OTCMS_PHP_V7.16.zip)

**软件架构**  
非 MVC 架构  
页面：smarty  
数据库：Mysql

### 0x01 反射型 XSS1

在`inc\classJS.php`，发现该文件定义了一些用于 js 弹窗的方法。一些方法的可控字符串被`AlertFilter()`方法包裹了。

[![](assets/1706958950-9fe9cc48db2f5ba69afa520001b52e75.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240126093633-564a1e22-bbeb-1.png)

`AlertFilter()`方法将英文双引号替换成中文双引号，无法使用英文双引号就没法闭合 alert()，那么使用了`AlertFilter()`是没有 XSS 的。

[![](assets/1706958950-baf92c0d636bbbf05c09a7202b797535.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240126093748-82d4e350-bbeb-1.png)

这里有一些弹窗方法没有使用`AlertFilter()`，例如`HrefEnd()`，找找使用的位置。

[![](assets/1706958950-e18ba1653a75a2ee0ea077f4fd2ef7f0.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240126095702-330ccba0-bbee-1.png)

找到使用`HrefEnd()`的位置。

[![](assets/1706958950-838c3010b8c1554d9e1d7cb8b2c057e4.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240126095842-6e9751a4-bbee-1.png)

在`users_deal.php`里 backURL 可控且没有做过滤。

[![](assets/1706958950-062b592c668556d25a56a06f50a7e8b9.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240126095824-63e4ba62-bbee-1.png)

构造 Payload，为了防止`document.location.href`跳转，将它的值设置为`#`号即可。

[![](assets/1706958950-8c510827d8b3f3226b523602cdca15e9.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240126151443-94271dce-bc1a-1.png)

测试触发 XSS。  
Payload：`http://localhost/users_deal.php?backURL=%23%22%3balert(1);//#`

[![](assets/1706958950-1a544515029f25a3d2d12b5d4feca3a9.gif)](https://xzfile.aliyuncs.com/media/upload/picture/20240126151647-de1081a0-bc1a-1.gif)

### 0x02 反射型 XSS2

在`inc\classAreaApp.php`，发现了一处简单的 XSS。

[![](assets/1706958950-5a45f754f43e4a5cd1c7966b75bb6328.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240125163307-5d749f9a-bb5c-1.png)

构造 Payload 测试触发反射型 XSS。  
Payload：`http://localhost/wap/users/p.php?m=sendPhoneForm&dataID=7866&theme=1%22&_=1706170896361&type=%22%3E%3Cscript%3Ealert(1);%3C/script%3E`

[![](assets/1706958950-4ad5207a8d75f5c1b792815826798e73.gif)](https://xzfile.aliyuncs.com/media/upload/picture/20240125163615-cd5ea134-bb5c-1.gif)

### 0x03 反射型 XSS3

在`wap\users\read.php`，有一处获取城市信息的功能。

[![](assets/1706958950-6bcc9664d94b69192f526238d706161b.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240125165947-16f827d6-bb60-1.png)

跟进到`inc\classProvCity.php#GetDeal()`，可以看到获取了两个参数 idName、prov。

[![](assets/1706958950-fe354cefff35116e33d6629766eba703.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240125170119-4e05e54c-bb60-1.png)

继续跟进到`inc\classProvCity.php#GetCityOptionJs()`，idName 被拼接之后返回给 echo 进行输出了。

[![](assets/1706958950-507004aa34f034ee5f08562b55891c4c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240125170240-7e85af86-bb60-1.png)

构造 Payload 测试触发 XSS。  
Payload：`http://localhost/wap/users/read.php?m=getCityData&idName=%3Cscript%3Ealert(1);%3C/script%3E`

[![](assets/1706958950-ba1d33f1450d01141c1dc2a4b31027a2.gif)](https://xzfile.aliyuncs.com/media/upload/picture/20240125170801-3d95972e-bb61-1.gif)

### 0x04 反射型 XSS4

在`apiRun.php#AutoRun()`看到可疑操作。

[![](assets/1706958950-5205caafad2d59644ff1f5da68740258.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240126145014-28a24054-bc17-1.png)

找到`$mode`变量是从`$_GET`获取且无过滤。

[![](assets/1706958950-26ee166e331bf4524fd327b47ab5ae95.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240126145054-4013e17a-bc17-1.png)

构造 Payload，可以通过`";`前面的内容，然后使用`//`注释后续内容，中间则是 XSS 的内容。

[![](assets/1706958950-890d32336f4a7c803adda2bd523e13dd.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240126145347-a7a6a94e-bc17-1.png)

构造 Payload 测试触发 XSS。  
Payload：`http://localhost/apiRun.php?mode=%22;alert(1);//&mudi=autoRun`

[![](assets/1706958950-cc5102f4d47f2e864de89ef0361adbb7.gif)](https://xzfile.aliyuncs.com/media/upload/picture/20240126145726-2a43f0fa-bc18-1.gif)

在`apiRun.php#AutoRunBig()`有同样的问题。

[![](assets/1706958950-00670d45e36e825db08d85fee89932ca.gif)](https://xzfile.aliyuncs.com/media/upload/picture/20240126150425-23c76026-bc19-1.gif)

### 0x05 SSRF1

用 Xcheck 找到一处 SSRF，在`inc\QrReader\QrReader.php#__construct()`里。如果`file_get_contents`获取的内容不是图片，它的返回值在交给`imagecreatefromstring`处理时会引发报错，所以这里只能 SSRF。

[![](assets/1706958950-e5b5a90408b928d7d3cb234913ed41fe.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240126162900-f48a18ba-bc24-1.png)

根据 Xcheck 路径跟踪表，`QrReader`对象在`admin\readDeal.php#ReadQrCode()`被创建，传入了可控参数`img`。

[![](assets/1706958950-416aae894e96ce956ac15dc25b7a7f7e.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240126161659-46db6346-bc23-1.png)

可以利用回显的延迟来对内网进行探测。

[![](assets/1706958950-cde007573d1db63139b8e35341eff018.gif)](https://xzfile.aliyuncs.com/media/upload/picture/20240126162237-10295186-bc24-1.gif)

### 0x06 SSRF2

用 Xcheck 找到第二处 SSRF，在`inc\classReqUrl.php#UseCurl()`。

[![](assets/1706958950-b765813122fe9b33cc12ea7c3f22479a.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240126164848-b8d8c76e-bc27-1.png)

根据 Xcheck 路径跟踪表，`UseCurl()`在`inc\classReqUrl.php#UseAuto()`被使用。

[![](assets/1706958950-ce655533d72993fd72326618405eddb0.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240126165144-21ade256-bc28-1.png)

接着按路径跟踪表，在`admin\read.php#GetSignal()`里调用了`UseAuto()`，可以看到获取了参数`signalUrl`。

[![](assets/1706958950-624525880db52cde71266376d7f0851c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240126165602-bb2fdaa6-bc28-1.png)

这里同样只能通过回显的延迟来对内网进行探测。

[![](assets/1706958950-c36a3599487b85227027149b43750f91.gif)](https://xzfile.aliyuncs.com/media/upload/picture/20240126170039-60a6c95e-bc29-1.gif)

### 0x07 SSTI

在后台有模板管理功能，编辑首页`index.html`试一下。

[![](assets/1706958950-0fc5f9e7da3ccc7b6e3afbcb3a239530.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240125130446-42928330-bb3f-1.png)

[![](assets/1706958950-7e34c360f52101c9dd9a9336883f2930.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240125130655-8f4a44ba-bb3f-1.png)

[![](assets/1706958950-75762c10acab12a5c273baee044b5cfa.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240125130801-b6b798ea-bb3f-1.png)

`{7*7}`直接以字符串的方式显示。

[![](assets/1706958950-cba034de323ed33aaba88c55b2dacf8d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240125130907-de04e97a-bb3f-1.png)

重新看一下，看到`{}`内的开头内容要为`otcms:`，按照这个样式进行修改。

[![](assets/1706958950-c50df1bafe0cf306eb0cdb97078532e8.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240125131056-1ef5b6c6-bb40-1.png)

重新访问，可以看到表达式被执行了。

[![](assets/1706958950-72c74118d33a0cb3b0fd9f4c1a42d01a.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240125131250-63135016-bb40-1.png)

它使用的模板引擎是 Smarty，可以使用`{otcms:$smarty.version}`查看一下 Smarty 的版本。

[![](assets/1706958950-f96e8e7e848aab178e6a90cbb3215c87.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240125131642-ed43fd58-bb40-1.png)

通过上述内容已经可以确认这里存在 SSTI 漏洞了，关于 Smarty 的 SSTI 漏洞，本人之前在先知发布过一篇[Smarty 最新 SSTI 总结](https://xz.aliyun.com/t/11108?time__1311=mqmx0DyDcDuGqq0vo4%2BxOD9WuKqDvN%2BQex&alichlgref=https%3A%2F%2Fxz.aliyun.com%2Fu%2F39303#toc-6 "Smarty 最新 SSTI 总结")，针对这个版本我们可以使用`CVE-2021-26119`来 RCE。  
将构造好的 Payload 写入 index.html。

[![](assets/1706958950-c2ed6bf9f02ff31124a4e116377119a4.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240125132759-806d9c0a-bb42-1.png)

测试触发 RCE。

[![](assets/1706958950-57285c263eea5e040d10ac1b567cd381.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240125132834-95ba5620-bb42-1.png)

分析一下代码，通过抓包找到功能代码。

[![](assets/1706958950-ffc34ea7119ce19092169f82672f1e75.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240125133517-85e582b4-bb43-1.png)

在`admin\template_deal.php#AddOrRevFile()`很简单可以看到`fileContent`没做任何防范 SSTI 的操作，针对文件名倒是做了许多处理。

[![](assets/1706958950-62dbf9ce87abdd51bb4d5834fd801bef.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240125133927-1ad9f332-bb44-1.png)  
[![](assets/1706958950-9ee9b64f5bc2a10c360eab8f492957dc.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240125134051-4ca6ed34-bb44-1.png)
