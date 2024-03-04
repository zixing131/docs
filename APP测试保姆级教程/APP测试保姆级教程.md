
APP测试保姆级教程

- - -

# **APP测试保姆级教程**

## **0x1 APP相关基础**

​ 首先需要认识APP是怎么构成的。APK本身是一个zip压缩包，因此可以通过压缩包的方式来打开，这里介绍几个常见的结构。

> assets：这是常见的资源文件目录，通常会存放一些图片类似的静态资源，有时候也会存放app的证书（证书是用来干啥的，往后看就明白了），有的APP只是丹丹封装了H5网页，那么其前端代码也会存放在这个目录底下。
> 
> kotlin：这个也是比较常见的文件夹，是APP的一个依赖包，这里一般不需要去分析。
> 
> lib：这里是存放.so文件的地方，.so文件又称为动态加载库，和windows的dll文件差不多，里面也会写一些函数方法，这个文件夹也不需要挨个去看。
> 
> META-INF：不用看
> 
> okhttp3：也是一个依赖包，不需要看。
> 
> res：这里是一些app前端会用到的图形组件库，一般也不用看，有些时候APP的证书会放在里面的raw目录底下。
> 
> AndroidManifest.xml：这里面可以看到APP需要用的一些权、APP的包名以及APP的主函数
> 
> classes\[n\].dex：这是java的二进制文件，也就是APP会加载的代码，这个可以反编译成java代码，然后分析代码
> 
> resources.arsc：这个文件通常不看，不用了解。

[![](assets/1709518965-4e42eb12b27dae5c05f2e5874ed6a4a9.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240303143623-5aa3c1ee-d928-1.png)

​ 对以上APP结构有了初步的认识后，这里再简单了解一下APP的基础信息

> 包名：APP开发的名称，一般是唯一的
> 
> 加固状态：这里我一般喜欢在MT管理器里看加固信息，个人感觉还是比较准的
> 
> 数据目录1：这里是APP的数据目录，经常会用到
> 
> 数据目录2：这里一般是辅助存放信息的，很少用到
> 
> APK路径：这就是APP安装后，会将自身的安装包以及so文件放在这里，运行的时候会取这里的文件。

[![](assets/1709518965-fa140c0b48b4a3e0c30c2f905556b335.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240303143634-61436fd6-d928-1.png)

​ 以上就是简单学习了APP的基础内容，那么接下来便开始讲解如何对APP进行渗透测试了。

## **0x2 APP分析阶段**

### 0x2.1 H5套壳APP

​ 什么是H5套壳APP呢？不懂可以再回头看一遍APP相关基础部分，我们可以在assets目录里发现有www目录或者该目录底下的其他目录里有html文件、js文件、css文件这种。这种的APP就是套壳的H5网页，这种我们可以学习WebViewPP插件动态调试js，去官网学习即可，我也不咋用这个；不动态调试js也可以将这些js文件拖到电脑上使用vscode进行静态分析，拼接API接口参数啥的去进行测试。

### 0x2.2 脱壳

​ 拿到APP，首先使用MT管理器查壳，不会的请看上一节的基础。一般我喜欢把APP放在 /sdcard/Download/ 目录下，这里就分两种情况了，无和有，无壳的情况下，可以用Jadx反编译APP，然后查看代码就完事了。那么这里讲一下有壳的情况下怎么弄呢，有壳那就脱壳呀，好好好，第一步，看看MT管理器分析的是什么壳，若非最新版的企业壳（如360付费壳，梆梆企业壳，娜迦），只说是最新的哈，如果都不是以上最新的，这里就不去踩坑学什么frida-dexdump了，能脱的太少了，那么这里介绍一个：Fundex 可以通杀。使用方法也很简单，LSPosed里找到Fundex，搜索要目标APP名，勾选上，然后打开Fundex，搜索目标APP名，点一下，接着打开APP即可**（这里需要注意的是如果有Root检测，需要用magisk+shamiko屏蔽Root）**。若打开APP闪退，那么，可以试试把Magisk和LSPosed都更新到最新版再试试，如果还不信，那就扔进垃圾箱吧，哈哈哈哈哈。

​ 如果顺利打开APP，那么可以在MT管理器里打开APP的数据目录1，就可以看到脱出来的一些classes\[n\].dex，这里通常看看如果某一个和APP安装包里的classes.dex大小一致，那就不要它，把剩下的拷到/sdcard/Download/目录下，然后发到电脑上等待反编译就行。**（这里需要注意一点，如果电脑上使用Jadx反编译不了脱出来的dex文件，那么需要使用MT管理器的dex修复功能，把对应的dex文件修复了再反编译，这个功能是收费的哦，需要买MT管理器的会员）**

​ 那么这里就介绍完脱壳部分了。

### 0x2.3 抓包

​ 既然我们是脚本小子~. ~ ，那么用完珍惜大佬的Fundex脱壳成功后，最重要的一点当然是抓包了，这里抓包也分好几种情况，且听我一一道来。

#### 0x2.3.1 正常抓包

​ 如果你在测试的时候，运气非常好，碰到一个抓包很顺利的APP，那么恭喜你。

​ 什么情况算得上正常抓包呢？这里同样分两点，WIFI代理和VPN代理。

> 在使用这俩代理之前，需要安装证书，Burp证书、Charles证书、Yakit证书、小黄鸟证书等；前三者已经够了，面具需要刷入Move Certificates模块，然后在系统设置里搜索证书，安装CA证书就行，安装完了一定要重启手机。
> 
> WIFI代理：在WIFI那里能设置，就不叭叭了
> 
> VPN代理：用ProxyDroid就行

​ WIFI代理若是抓不到，就用VPN代理，若还是抓不到，推荐使用Yakit代理流量给Burp（特殊时可以试试勾选国密劫持）

[![](assets/1709518965-4768eac4b2c93f6018daee87a209429a.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240303143649-69e965c8-d928-1.png)

​ 同理可以使用Charles代理给Burp

[![](assets/1709518965-1ee16fb4e92f7644eed5629b76b47de3.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240303143700-70647e56-d928-1.png)

[![](assets/1709518965-77b4cdc479af827804bb24865770a2ce.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240303143709-7616c0c0-d928-1.png)

​ 若还是抓不到，也可以试试Socks5代理，在ProxyDroid里可以设置，同理Charles和Yakit均可接收Socks5流量。

#### 0x2.3.2 SSL单向校验

​ 讲完正常抓包的流程，这里就不得不讲非正常抓包了，首先是SSL单向验证，那么，什么是SSL呢？其实SSL就是在http上加了个证书，也就是所谓的https。如果抓包的时候发现抓不到包，可以试试绕过SSL单向验证，这里推荐两个插件：TrustMeAlready和SSLUnpinning。通常这俩我会用前面那个勾选目标APP，无效的话我会俩都勾选目标APP，若还是无效的话，可以试试frida的sslunpinning脚本**（需要用的时候网上自行搜索，一般那俩插件都能解决，并且frida这个脚本还不一定能解决）**。

#### 0x2.3.3 双向校验

​ 有单向校验，那么就一定会有双向校验了，双向校验的原理其实就是APP和服务器之间用一个证书来绑定，二者只能互相信任，其它任何设备，任何人要跟他说话都是不可信的。怎么识别双向校验呢？首先映入眼帘的应该是抓包的时候返回400状态码，并且能在请求包里看到 No required SSL certificate was sent

[![](assets/1709518965-e50ec5bd6196790ae86e207f48fd10ed.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240303143722-7db472b4-d928-1.png)

#### 0x2.3.3 不走HTTP走Socket

​ 说实话，这么恶心的APP我确实遇到过一次，网上也有人遇到过这样的APP，他运气挺好，只需要HOOK一下是否走HTTP报文为是就完事了。。。。。那么这里讲一下我们遇到这种APP怎么办？Frida RPC了解一下，这里就不多说了哈哈哈哈，网上也有文章：[https://www.adminxe.com/4367.html](https://www.adminxe.com/4367.html)

​ 当然，第二部分APP分析阶段抓包部分若仍然抓不到，那么也可以尝试 Frida RPC，有时候连加解密的地方都可以省略，缺点就是没法爆破。。。。

## 0x3 加解密分析

​ 这是一个比较大的板块，需要会的东西比较多，最重要的就是代码阅读能力以及函数调用顺序的理解程度，是否认识形参和实参之间的联系。

​ 如果在抓包的时候遇到了密文，就是那种base64编码的或者hex编码的一堆看不懂的字符串，那么这里就认为它是进行了加密，我们需要去分析它的加密过程，然后去模拟这个加密过程，接着才能去继续渗透测试。

​ 前面我们已经讲过了脱壳部分以及H5套壳的APP了，第二者可以分析js里的加密流程，前者可以使用Jadx分析脱出来的dex文件反编译得到的Java代码，寻找加密过程，这里讲一下x按键，点击对应函数，按下x按键可以查找谁调用了这个函数，Jadx更多功能可以试试右键某一处你想看的函数名试试（比如生成Frida代码这个功能就非常好用）。

​ 这里有几个方式了解一下：

> 全局搜索encrypt，然后挨个地方分析
> 
> 搜索抓到包的url字段，然后分析周围代码，寻找加密方法
> 
> 搜索特征字符串，比如抓到的包里请求包里是这样的：encStr=xxxxxxx，那么我们就可以搜索encStr，然后分析它是怎么运算得到的即可

​ 加密方法常见的有：

> 对称加密：AES、SM4、DES、RC4、Rabbit，常用的是前三者
> 
> 非对称加密：RSA、SM2
> 
> Hash加密：MD5、SM3、SHA256....

​ 如果有防篡改**（相同数据包只能请求一次，请求第二次无效）**，那么也是同样的思路去分析防篡改的字段是怎么生成的，随后模拟一遍即可。

## 0x4 Frida讲解

​ Frida是一个可以用来动态Hook的工具，通常分为客户端和服务端这两种，服务端装在手机里，通常放在/data/local/tmp/目录下，需要su之后再启动frida服务；客户端装在windows电脑上，用如下两条指令**(需要确保手机和电脑上的frida版本一致，推荐都是最新版即可)**：

```plain
pip install frida
pip install frida-tools
```

​ 这里推荐一下hluda，是frida一个去特征版本，可以过部分壳的检测。

​ frida一个非常简单的代码如下：

```plain
Java.perform(function(){
    console.log("frida注入成功")

    let Response = Java.use("com.xxx.EncryptAESJsonHandler");
    Response["encryptJSon"].implementation = function () {
        console.log(`Response.body is called`);
        let result = this["encryptJSon"]();
        console.log(`Response.body result=${result}`);
        return result;
    };
})
```

​ 仔细看以上代码，这里逐个讲解，其实就是js代码，保存为xxx.js即可。首先必须有Java.perform()，然后这个里面放上一个function，这里面全是js的写法。然后不一样的在于Java.use("com.xxx.EncryptAESJsonHandler")是使用APP里的某个类，然后Response\["encryptJSon"\]里的encryptJSon是前面那个类里的函数，我们使用implementation对其进行重载，然后有几个参数就写几个参数就行，最后记得返回的时候，要执行一下本身这个函数。

​ 以上这个过程其实Jadx里右键生成Frida代码就都有了，只需要替换中间代码部分即可。

[![](assets/1709518965-082357b1de9dc3e9f2ea71dc162b735f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240303143736-85d10548-d928-1.png)

​ 启动方法也很简单，首先USB连接手机和电脑

​ 启动服务端（记得给frida或hluda可执行权限 chmod +x hluda）：

```plain
adb shell
su
/data/local/tmp/hluda
```

​ 客户端：

```plain
frida -U -f com.xxx.xxxx -l xxx.js
```

> \-U 代表USB链接
> 
> \-f 后面跟上需要启动的APP包名
> 
> \-l 后面跟上要执行的脚本

​ 学习完Frida的基础部分，我们可以用其来获取我们反编译的APP在某个函数执行时的形参以及结果，这样我们可以更快获得加解密的key等信息。如果文章内提到的一些插件或者脚本你没有，请在github或者百度里搜索下载，最后，APP渗透讲完了，Good Luck。
