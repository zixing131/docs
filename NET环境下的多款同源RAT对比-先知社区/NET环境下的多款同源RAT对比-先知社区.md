

# NET环境下的多款同源RAT对比 - 先知社区

NET环境下的多款同源RAT对比

- - -

## 概述

前段时间，笔者发布了一篇《QuasarRAT与AsyncRAT同源对比及分析》文章，文章发布后，还是获得了不少小伙伴的关注，同时还从其他渠道收到了小伙伴的私信：

[![](assets/1709531000-4747fbae6de094e1daa51576c7f3bdec.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301215639-868da6d6-d7d3-1.png)

首先，非常感谢大家的关注及厚爱，其次，针对小伙伴提到的内容，我准备以此为切入口详细研究对比一下。

为了能够更全面的对比分析，笔者准备从如下角度开展研究工作：

-   下载VenomRAT项目、DcRAT项目、在野DcRAT样本及在野VenomRAT样本、xRAT项目进行对比分析；
-   尝试对上述样本配置信息及通信数据进行解密，并对解密算法进行对比；

通过分析，笔者发现上述VenomRAT项目、DcRAT项目、在野DcRAT样本及在野VenomRAT样本使用的配置信息加密算法均相同，以《AsyncRAT加解密技术剖析》文章中“配置信息解密”章节中描述的解密流程作为基础解密算法标准，梳理各样本针对配置信息解密算法的区别如下：

-   VenomRAT：PBKDF2算法运算中，salt字符串值为：VenomByVenom
-   DcRAT：PBKDF2算法运算中，salt字符串值为：DcRatByqwqdanchun
-   在野DcRAT：PBKDF2算法运算中，salt值为函数参数传入；
-   在野VenomRAT样本
    -   配置信息中的Key数据将用于PBKDF2算法的输入，通过迭代计算生成两个加密密钥，一个用于aes运算，密钥长度**16字节**，一个用于HMACSHA256哈希值计算，密钥长度**64字节**；
    -   PBKDF2算法运算中，salt十六进制值为：**BFEB1E56FBCD973BB219022430A57843003D5644D21E62B9D4F180E7E6C33941**
-   xRAT：解密算法不同
    -   将key字符串的**MD5值**作为AES算法key；
    -   待解密字符串的前**16个字节**为AES算法的iv值，后续字节为加密数据；

## VenomRAT

为了能够更好的对VenomRAT进行研究，笔者的第一想法就是看看网络中是否能够下载类似于QuasarRAT项目的具备控制端的VenomRAT程序。通过一系列网络调研，笔者确实是下载了好几个VenomRAT项目，起初还觉得很顺利，直到使用的时候，笔者才发现下载的VenomRAT项目**均携带了木马**。

为了能够继续对VenomRAT进行研究，笔者通过一系列努力，能够比较坎坷的使用VenomRAT某个版本，运行截图如下：

[![](assets/1709531000-d782b400143f46b203913a60649bf8f0.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301215653-8f033b64-d7d3-1.png)

### 生成VenomRAT端木马

在GUI界面中选择【Builder】菜单即可对VenomRAT端木马进行自定义配置，支持自定义配置的内容有：

-   IP/DNS标签页：外联IP、外联端口、Pastebin、Group Name、互斥对象名
-   Startup：自启动信息、启用反虚拟机选项、禁用任务管理器选项、启用蓝屏选项
-   Assembly：属性信息、图标信息

相关截图如下：

[![](assets/1709531000-28f67def331b2420c67b2bd26c55e0f6.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301215707-9763bcc0-d7d3-1.png)

### 木马上线

在受控主机中运行VenomRAT端木马程序，即可成功实现木马上线，上线后即可实现对受控主机的远控管理，相关截图如下：

[![](assets/1709531000-56a4165151aa1891feb50249b4b86549.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301215722-a06db6ea-d7d3-1.png)

### 配置信息解密

通过分析，笔者发现，VenomRAT端木马的反编译代码结构与AsyncRATClient端木马的反编译代码结构相同，均在Client命令空间的Settings类中存在大量的加密配置信息数据，相关截图如下：

[![](assets/1709531000-041fb56f025d2349ee2bf27f520b3665.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301215736-a8afc42e-d7d3-1.png)

尝试基于《AsyncRAT加解密技术剖析》文章中提到的自动化解密脚本，并对解密脚本做简单的修改，可成功对上述配置信息进行解密，解密后截图如下：

[![](assets/1709531000-cb9406a524af7863f2d6cc92ba63d7f0.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301215751-b1b762fc-d7d3-1.png)

### 通信数据解密

对VenomRAT木马上线过程及远程控制过程进行流量抓取分析，发现其通信数据与AsyncRAT木马通信数据一致，均分为两层：

-   第一层加密：调用TLS对通信数据进行加密；
-   第二层加密：调用gzip对通信载荷及传输模块进行加密；

相关截图如下：

[![](assets/1709531000-d3f28d5885a08a67f7322d652a74d824.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301215806-bab7cefa-d7d3-1.png)

为了进一步探究其通信模型，笔者准备从如下角度进行TLS通信解密尝试：

-   尝试基于《适用于不支持指定密钥套件的NET程序的TLS解密方法》文章中提到的修改TLS密钥套件的方法，修改其TLS通信过程中使用的密钥套件；
-   尝试基于《AsyncRAT加解密技术剖析》文章中提到的私钥提取方法及通信解密方法，对VenomRAT木马TLS通信数据包进行解密；

TLS通信解密后截图如下：

[![](assets/1709531000-377302cb3b63be180fb06b7bfc5fa536.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301215821-c3566ac6-d7d3-1.png)

通过分析，发现其TLS解密后的通信数据结构与AsyncRAT木马TLS解密后的通信数据结构极其相似，因此，尝试基于《AsyncRAT通信模型剖析及自动化解密脚本实现》文章中提供的通信解密程序对其进行批量解密，发现可成功对其进行解密，相关截图如下：

[![](assets/1709531000-4574197d528e87a39aeb046a5dd4136d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301215835-cbba6546-d7d3-1.png)

## DcRAT

在对DcRAT项目的查找过程中，笔者发现也不是很顺利，因为笔者发现网络中曝光的DcRAT与github上DcRAT项目中的木马代码结构不一样，但是在进一步分析过程中，笔者发现其均使用了相同的配置信息解密算法，因此，笔者准备分别对其进行研究对比。

通过运行github上的DcRAT项目，可成功打开DcRAT控制端，相关截图如下：

[![](assets/1709531000-8557c4fd23a9b25cc6c80d75426f8ad8.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301215848-d3bcf92a-d7d3-1.png)

### 生成DcRAT端木马

在GUI界面中选择【File】->【Builder】菜单即可对DcRAT端木马进行自定义配置，支持自定义配置的内容有：

-   外联IP、外联端口、Pastebin、Group Name、互斥对象名
-   自启动信息、启用反虚拟机选项、禁用任务管理器选项、启用蓝屏选项
-   属性信息、图标信息

相关截图如下：

[![](assets/1709531000-5a687c288decafab1fc556c49b3cc431.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301215903-dcbaa73e-d7d3-1.png)

### 木马上线

在受控主机中运行DcRAT端木马程序，即可成功实现木马上线，上线后即可实现对受控主机的远控管理，相关截图如下：

[![](assets/1709531000-373fd79fd1e5434cdd7c2506707ccf3f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301215917-e521850a-d7d3-1.png)

### 配置信息解密

通过分析，笔者发现，DcRAT端木马的反编译代码结构与AsyncRATClient端木马的反编译代码结构也相同，均在Client命令空间的Settings类中存在大量的加密配置信息数据，相关截图如下：

[![](assets/1709531000-18d42bdc1eeb1a5fc7af388fb286207c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301215932-edc03dc8-d7d3-1.png)

使用相同的配置信息解密方法，并对解密脚本做简单的修改，可成功对上述配置信息进行解密，解密后截图如下：

[![](assets/1709531000-937f63a9725a99663bdcca8d39ac62d9.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301215947-f6f3cbd0-d7d3-1.png)

### 通信数据解密

对DcRAT木马上线过程及远程控制过程进行流量抓取分析，发现其通信数据也与AsyncRAT木马通信数据一致，相关截图如下：

[![](assets/1709531000-4cc5a84d893dca44eed7c784fe21998a.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301220005-01a01d40-d7d4-1.png)

因此，使用相同的TLS通信数据解密方法，可成功对其TLS通信数据进行解密，解密后截图如下：

[![](assets/1709531000-baba45a3187c1de1a2b25ae585639324.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301220019-0a36f320-d7d4-1.png)

使用相同的通信解密程序对其TLS解密后通信数据进行批量解密，发现可成功对其进行解密，相关截图如下：

[![](assets/1709531000-b8b13ff6f48cba6f61e09a8038275696.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301220035-1360a126-d7d4-1.png)

## 在野DcRAT样本剖析

为了能够全面的对DcRAT进行分析，笔者从网络中下载了一个最新曝光的DcRAT木马，相关曝光信息如下：

[![](assets/1709531000-08f9b552e7a1df7e1c5fb18ab357c674.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301220051-1d398078-d7d4-1.png)

对其进行简单分析，笔者发现此在野DcRAT样本的反编译代码结构与DcRAT项目中的木马反编译代码结构不同，相关截图如下：

[![](assets/1709531000-418fe40a47c93a3185d4556e3e1ac36b.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301220107-2678287e-d7d4-1.png)

### 配置信息解密

进一步对其进行分析，笔者发现此样本调用的配置信息解密函数与AsyncRATClient端木马调用的配置信息解密函数相同，相关截图如下：

[![](assets/1709531000-cf3a82e71c611517eec70ce7c59da298.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301220122-2f355478-d7d4-1.png)

尝试使用相同的配置信息解密方法，并对解密脚本做简单的修改，可成功对样本中的配置信息内容进行解密，相关截图如下：

[![](assets/1709531000-70c0ab8193501f6c86bfadb86e035965.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301220135-3777ddf4-d7d4-1.png)

[![](assets/1709531000-c0e738f7752c4f1a1e070431c5d16f1d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301220152-41241b38-d7d4-1.png)

解密后的数据中还包含base64编码信息，可手动对其进行二次处理，相关操作如下：

```plain
原始字符串：
"twqziPMyOf6TnyOB/OK1jTdK956e34V42RMtGMVty6+ZbZ/0qhyPa51EFIbkOILnUmjGENz8Bsxp9j12/g0Zr3vpvrUnsOzV2cwwuEaLXKjVJIqSveHZfNuYG6F4zyNhcW8shTMg0VI7dKjnY1vGpJbwrXByPVnI4FBFnJoRImSAE1vNJjvzdOzHq5+w2xstewXQhRP4PqEtgiVd3odCEho1geLc70vkATTvkgk2FVbmSJAF1j6SSlWrBFBm8Bl2lLqol1r85lvAIjSpagTFIm8QKQfD05h5sXR17sKazsdKdP9ahYS+ldWkjEMLe8tV5boxfNV1gJDpi15NixjKJWS5myqzYOhSQn1JynlWh9ej0Y2YYlj3YEp/j+xqWYvOvnHPVdOt928Z9+jep98h0SxvkGnrNxLvcDIJI0VSVkC9eIU4XADkRe4hAMmJbvQ5671XQSoLJCsWxQ4IzS596vNXL7n+UKLx2LXD/fkJNE7NMMOKuFGBQ+IgdOffNUw9gOV3731cJ4WFYfMLMLuhZeQI4sDbY9xlAXD9Ha+7hY7Dx9sk3u9ybZZ0DP0nxW2w9zNad/GEX9+MklEXrRjLjGDD5iCQKCAMKaSVEsTvKPZ3RX2BtuRrL2egqdU531tZKbG4yJnXY12vrzJeS2Dg+1/IVQEoFVfNoWF0sPil1Dvmt28pC5+7+9v8/vIxVfn6LP4PbSpTW1qNSZK5LWQDiSAFyFfnO6Vpk7atHaYlb1+t9gBaPBOLJJQCwXLNUVhRwY271kvh8EUUwFo4ld7kPVv5zNIbe5oTVR8UewFIES2f4KGGLo4loJpBM+5dMvomDctqqFNCmASxLHikniRsOs+5ci4I0hig0khqu1JYM8hNxrlaTPI2BboP3f0gFhFN9YmTL1KicLWdh6ftKWRFQX0qPYd6Ww4oQuBAxEkKA76wEHdpoO5iDVCXoQhF+QPpXxRrxvIzbPlpmQsfPiZa+/N4P9zm0wmTv02102/UN+0="
解密后字符串：
["bj0UKX3O1fsx9BYPGXoKHqjvLayVva1jN63FIaBpzhY4ZE1D43om8NOuAFJtihcbnIkDHSHpW8UjRpWHjvb2vPk9sIFCRRHSF7QQdy5lw8PA2odUtBKwGkpYhlU9MEYF","DCR_MUTEX-11Fyfh7gXU61FzPB2sRh","0","VV??","","5","2","WyIxIiwiIiwiNSJd","WyIxIiwiV3lJaUxDSWlMQ0psZVVsM1NXcHZhV1V4VGxwVk1WSkdWRlZTVTFOV1drWm1VemxXWXpKV2VXTjVPR2xNUTBsNFNXcHZhVnB0Um5Oak1sVnBURU5KZVVscWIybGFiVVp6WXpKVmFVeERTWHBKYW05cFpFaEtNVnBUU1hOSmFsRnBUMmxLTUdOdVZteEphWGRwVGxOSk5rbHVVbmxrVjFWcFRFTkpNa2xxYjJsa1NFb3hXbE5KYzBscVkybFBhVXB0V1ZkNGVscFRTWE5KYW1kcFQybEtNR051Vm14SmFYZHBUMU5KTmtsdVVubGtWMVZwVEVOSmVFMURTVFpKYmxKNVpGZFZhVXhEU1hoTlUwazJTVzVTZVdSWFZXbE1RMGw0VFdsSk5rbHVVbmxrVjFWcFRFTkplRTE1U1RaSmJsSjVaRmRWYVV4RFNYaE9RMGsyU1c1U2VXUlhWV2xtVVQwOUlsMD0iXQ=="]
二次处理后结果：
["bj0UKX3O1fsx9BYPGXoKHqjvLayVva1jN63FIaBpzhY4ZE1D43om8NOuAFJtihcbnIkDHSHpW8UjRpWHjvb2vPk9sIFCRRHSF7QQdy5lw8PA2odUtBKwGkpYhlU9MEYF","DCR_MUTEX-11Fyfh7gXU61FzPB2sRh","0","VV??","","5","2","["1","","5"]","["1","["","","{"0":"{SYSTEMDRIVE}/Users/","1":"false","2":"false","3":"true","4":"true","5":"true","6":"true","7":"false","8":"true","9":"true","10":"true","11":"true","12":"true","13":"true","14":"true"}"]"]"]

*****************************************************************

原始字符串：
"6DuJThqLqhXMRndyjcrpSvR+NowgfgPUfadTAPLT7RzQEaQ3bZTS2B69cJ+6b9gMItPpYbJufWtQMjS77Qehab2Q+nE+hYfWDfb+T9kHg8KoSt+NAc00NmL95jbxX5qWdMKBiNsSTppEM/HD93PwYFKZCrLv7VhGHiQP8GV5/h8KKSZ+93DQTyTyXIU9kKzo6EM/bmELphag+kIO5kj28pRQY9kCOtzWU5LxezAmxJdrcp+EGjpZSgMpeynFIZE9"
解密后字符串：
["0","XPkWC3v1QKzwU0J5dAKeTsPBsYp18q5mbMsCqw5G1NTNQgIkoqWSj2GpAinnN33kONVHHGPqEEnGZBvMQFMRTmCiGDCHIS37Ts8DKAchbqOfP9P8xbXIqlQlKxBEEHhv"]

*****************************************************************

原始字符串：
zCEl5MLNt1nWGMDkINJb16lVnQwVhHlbE0ON/jzps092WYVbsn8xXBFE1kAEM8FE6Zu4vZdIFAVDmeASNmk+Cal/saaZFTYrBzpD6gHAmeV/2nzMJLz3TeS9r66FgUt0rP/vImvRIfwAfOjcSrkD1sdkzQFiIyDJZ3lO3QnF4FxspW89vlhx6OBIISdDgT0h

解密后字符串：
[["http://019214cm.nyashland.top/","EternalLineLowgameDefaultsqlbaseasyncuniversal"]]
```

### 外联通信

通过分析，发现此在野DcRAT样本的外联通信方式与DcRAT项目中的木马外联通信方式不同：

-   此在野DcRAT样本使用HTTP请求作为外联通信方式；
-   DcRAT项目中的木马使用TLS作为外联通信方式；

相关截图如下：

[![](assets/1709531000-eb85d9a61ac172fdaa05c8270a56c7ca.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301220210-4c019c56-d7d4-1.png)

[![](assets/1709531000-ba16b7435b547f5ea0bc5f564ea9e05f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301220226-5591ee92-d7d4-1.png)

## 在野VenomRAT样本剖析

为了能够全面的对VenomRAT进行分析，笔者又从网络中下载了一个VenomRAT木马，相关信息如下：

[![](assets/1709531000-eb62a3880a0a4d1f9f5f4464f8c713e6.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301220241-5ed15d80-d7d4-1.png)

[![](assets/1709531000-7f3b67908c5244b2de7df24b15e1d051.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301220258-68e75bf8-d7d4-1.png)

### 样本分析

通过分析，笔者发现此样本并非实际VenomRAT木马实体，进一步分析，笔者发现当此样本运行后，此样本将在内存中释放VenomRAT木马实体，相关截图如下：

[![](assets/1709531000-a515b3bfc54d809aa22c938352b2f75f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301220313-71b6ab26-d7d4-1.png)

使用de4dot对其去混淆，然后使用dnspy查看VenomRAT木马实体的反编译代码，发现VenomRAT木马实体的反编译代码与VenomRAT项目中的木马反编译代码相同，相关截图如下：

[![](assets/1709531000-d06a392200e727cdaa5f6ad869ef049c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301220329-7afd2246-d7d4-1.png)

### 配置信息解密

使用相同的配置信息解密方法，并对解密脚本做简单的修改，可成功对上述配置信息进行解密，解密后截图如下：

[![](assets/1709531000-992c5161aab8987844095a30e55aeb8f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301220343-8358a258-d7d4-1.png)

## xRAT

在之前的《QuasarRAT与AsyncRAT同源对比及分析》文章中，笔者曾对QuasarRAT工具的版本进行过梳理，梳理内容如下：QuasarRAT工具的早期版本名称为xRAT，xRAT于2014年7月8日发布xRAT v2.0.0.0 RELEASE1版本，后续于2015年8月23日发布Quasar v1.0.0.0版本。

通过运行github上的xRAT项目，可成功打开xRAT控制端，相关截图如下：

[![](assets/1709531000-ad3998d210c099da52f47b56ba1dc3ec.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301220357-8c1d871e-d7d4-1.png)

### 生成xRAT端木马

在GUI界面中选择【Builder】菜单即可对xRAT端木马进行自定义配置，支持自定义配置的内容有：

-   Connection：IP/Hostname、Port、Password、Reconnect Delay；
-   Install：Mutex、Install Client选项、Install Name、Install Path、Install Subfolder、Example Path、Registry Key Name；
-   Assembly Information：属性信息；
-   Additional Settings：启用管理员权限选项、修改图标、启用键盘记录选项；

相关截图如下：

[![](assets/1709531000-f8f14b1aa0886558f665e80190aa19a3.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301220413-954efc3c-d7d4-1.png)

### 木马上线

在受控主机中运行xRAT端木马程序，即可成功实现木马上线，上线后即可实现对受控主机的远控管理，相关截图如下：

[![](assets/1709531000-5b20c8499bd08d2c5f1239d57ffb5365.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301220429-9ef0f254-d7d4-1.png)

### 配置信息解密

通过分析，笔者发现，xRAT端木马的反编译代码结构与AsyncRATClient端木马的反编译代码结构也基本相同，相关截图如下：

[![](assets/1709531000-61d5567c207657ecd116057208f785c7.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301220446-a938234a-d7d4-1.png)

通过对其配置信息解密算法解密分析，发现其解密算法整体逻辑相同，但还是存在多处不同点，因此，尝试修改解密算法对其进行解密，发现可成功解密，解密后截图如下：

[![](assets/1709531000-73868969c11e85fd37a3a6d3484c7d42.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301220504-b40d5c36-d7d4-1.png)

修改后的解密脚本代码如下：

[![](assets/1709531000-dbd558cae6a8e64160c2d59f804747c8.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301220520-bd83ff04-d7d4-1.png)

-   main.go

```plain
package main

import (
    "awesomeProject3/common"
    "encoding/hex"
    "fmt"
    "strings"
)

func main() {
    file_in := "C:\\Users\\admin\\Desktop\\11.txt"

    key := ""
    strs := common.FileToSlice(file_in)
    for _, str := range strs {
        if strings.Contains(str, "public static string string_8 = ") {
            key = strings.Split(strings.Split(str, ` = "`)[1], `";`)[0]
        }
    }
    for _, str := range strs {
        if strings.Contains(str, "public static string string_0 = ") {
            Version := strings.Split(strings.Split(str, ` = "`)[1], `";`)[0]
            fmt.Print("Version:")
            decrypt_xRAT_str(key, common.Base64_Decode(Version))
        } else if strings.Contains(str, "public static string string_1 = ") {
            Hosts := strings.Split(strings.Split(str, ` = "`)[1], `";`)[0]
            fmt.Print("Hosts:")
            decrypt_xRAT_str(key, common.Base64_Decode(Hosts))
        } else if strings.Contains(str, "public static string string_2 = ") {
            Password := strings.Split(strings.Split(str, ` = "`)[1], `";`)[0]
            fmt.Print("Password:")
            decrypt_xRAT_str(key, common.Base64_Decode(Password))
        } else if strings.Contains(str, "public static string string_5 = ") {
            Install_Name := strings.Split(strings.Split(str, ` = "`)[1], `";`)[0]
            fmt.Print("Install Name:")
            decrypt_xRAT_str(key, common.Base64_Decode(Install_Name))
        } else if strings.Contains(str, "public static string string_6 = ") {
            Mutex := strings.Split(strings.Split(str, ` = "`)[1], `";`)[0]
            fmt.Print("Mutex:")
            decrypt_xRAT_str(key, common.Base64_Decode(Mutex))
        }
    }
}

func decrypt_xRAT_str(key string, input []byte) {
    aeskey, _ := hex.DecodeString(common.CalculateMD5(key))

    aes_iv := input[:16]
    encode_data := input[16:]

    output, _ := common.Aes_z_Decrypt(encode_data, aeskey, aes_iv)
    fmt.Println(string(output))
    //fmt.Println(hex.EncodeToString(output))
}
```

-   common.go

```plain
package common

import (
    "bufio"
    "crypto/aes"
    "crypto/cipher"
    "crypto/md5"
    "encoding/base64"
    "encoding/hex"
    "fmt"
    "os"
)

func Aes_z_Decrypt(input, key, iv []byte) (output, newiv []byte) {
    output, err := aes_decrypt_cbc(input, key, iv)
    //fmt.Println(hex.EncodeToString(output))
    if err != nil {
        panic(err)
    }
    newiv = append(newiv, input[len(input)-16:]...)
    return
}
func aes_decrypt_cbc(data []byte, key []byte, iv []byte) ([]byte, error) {
    data_new := []byte{}
    data_new = append(data_new, iv...)
    data_new = append(data_new, data...)

    block, err := aes.NewCipher(key)
    if err != nil {
        return nil, err
    }

    if len(data_new) < aes.BlockSize {
        return nil, fmt.Errorf("加密数据太短")
    }

    iv = data_new[:aes.BlockSize]
    data_new = data_new[aes.BlockSize:]

    if len(data_new)%aes.BlockSize != 0 {
        return nil, fmt.Errorf("加密数据长度不是块大小的整数倍")
    }

    mode := cipher.NewCBCDecrypter(block, iv)
    mode.CryptBlocks(data_new, data_new)

    data_new = pkcs5UnPadding(data_new)

    return data_new, nil
}

func pkcs5UnPadding(ciphertext []byte) []byte {
    length := len(ciphertext)
    unpadding := int(ciphertext[length-1])
    return ciphertext[:(length - unpadding)]
}

func Base64_Decode(encodedMessage string) []byte {
    decodedMessage, err := base64.StdEncoding.DecodeString(encodedMessage)
    if err != nil {
        fmt.Println("Base64_Decode Error:", err)
        return nil
    }
    return decodedMessage
}

func CalculateMD5(input string) string {
    hasher := md5.New()
    hasher.Write([]byte(input))
    md5Hash := hex.EncodeToString(hasher.Sum(nil))
    return md5Hash
}

func FileToSlice(file string) []string {
    fil, _ := os.Open(file)
    defer fil.Close()
    var lines []string
    scanner := bufio.NewScanner(fil)
    for scanner.Scan() {
        lines = append(lines, scanner.Text())
    }
    return lines
}
```

### 通信数据解密

对xRAT木马上线过程及远程控制过程进行流量抓取分析，发现其通信数据与其他木马的通信数据不同，xRAT木马通信数据使用socket套接字进行通信，并未使用TLS进行通信，相关截图如下：

[![](assets/1709531000-2e4126984b31ce3405b911c6311fa009.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301220540-c96cdd72-d7d4-1.png)

进一步对其进行研究分析，发现其通信数据解密流程如下：

-   AES解密：将配置信息中的Password作为`decrypt_xRAT_str(key string, input []byte)`函数的key参数，将加密载荷数据作为input 参数；
-   QuickLZ 解压缩：使用QuickLZ 解压算法对AES解密后的数据进行解压缩；

通信数据解密案例如下：

```plain
#通信数据载荷
300000  #载荷数据大小
8954472455a849b42094be7cba1fc9d487725502301762beee04955ded9ad8df5acd0e07dbc128545a7ce086c8552233    #加密载荷数据

#AES解密后数据
4F130000000600000000000080020000000A00

#QuickLZ解压缩后数据
020000000A00
```
