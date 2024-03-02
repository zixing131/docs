

# APK 逆向分析入门 - 以某 app 脱壳为例 - 先知社区

APK 逆向分析入门 - 以某 app 脱壳为例

- - -

## 分析

### 源码分析-Jadx

-   拖入 jadx，发现是有加固的，看目录应该是腾讯御安全加固。

[![](assets/1705982064-e5df6e182175aad034f14f3cca0967a3.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240121114654-b817eaba-b80f-1.png)

-   使用 MT 管理器验证后就是腾讯御安全加固，这里要考虑如何脱壳。

### 安装

-   可以正常安装，而且第一步要先登录，这里就先进行脱壳，再考虑接下来的事。

[![](assets/1705982064-9a26e1ff6154733e779ed0b7f73a3c7c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240121114701-bc6575ba-b80f-1.png)

## 脱壳

### 加固基本原理

-   apk 加壳是为了提高安全性，但是也会影响真实体验，所以有的 apk 选择加固有的考虑到大量用户的体验就不进行加固，而且考强大的“法律团队”。
-   加壳目前主要分为 4 个过程的进化，这里就简要介绍，详细了解大家可以去网上找一些大佬的讲解。
    -   一：代码混淆
    -   二：动态加载 dex
    -   三：动态加载 dex 并进行函数抽象化
    -   四：VMP 虚拟化保护
-   一般遇到的就是 dex 加固，再复杂的脱壳也比较麻烦，大家可以考虑一下投入与回报。
    -   原理简单来说就是把真实原本的 apk 加载到内存后 dump 出来。而脱壳要做的就是找到一个合适的时机 dump 出真实的 dex。这里推荐一篇博文，详细介绍加固脱壳的原理。[https://mp.weixin.qq.com/s?\_\_biz=MjM5NTc2MDYxMw==&mid=2458457914&idx=1&sn=1d505597b47089cbd8bdc1e6e0a2b068&chksm=b18e27b086f9aea63d4404f118b0dc4c0b87c5986f20e556a5a85c967834df0d7f3e8e6d98d6&scene=27](https://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458457914&idx=1&sn=1d505597b47089cbd8bdc1e6e0a2b068&chksm=b18e27b086f9aea63d4404f118b0dc4c0b87c5986f20e556a5a85c967834df0d7f3e8e6d98d6&scene=27)

### 脱壳

这里不用 hook 等方式手动脱，能工具的还是优先工具。这里使用脱壳神器 BlackDex[https://github.com/CodingGay/BlackDex](https://github.com/CodingGay/BlackDex)

[![](assets/1705982064-9f03bd4d5d5b8fc34220f21634dc8505.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240121114711-c2539af6-b80f-1.png)

-   这里直接用 BlackDex 显示脱壳成功，但是显示有多个 dex 文件。

[![](assets/1705982064-cf794412b0d3b888af6b4e2c2d8586d8.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240121114718-c635548e-b80f-1.png)

-   这里要先做的是删除多余没用的 dex 并替换原有 apk 下的 dex。同时比较原有 dex 文件，把脱壳后的和原大小相同的也删掉。

[![](assets/1705982064-f4c0c2a6036db3984a72663d564688a6.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240121114724-ca0f3d72-b80f-1.png)

-   对 dex 进行修复，修复后产生的 bak 备份文件删除就行。

[![](assets/1705982064-880af284ba405c4fb4116073212c1c88.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240121114731-cddb6ac0-b80f-1.png)

-   去原有 dex 下的加固代理文件中找到真实的入口名，并修改 amf.xml 文件中的入口为真实的入口名。下图一是加固代理入口中的真实入口，图二是要替换掉的 amf.xml 入口。

[![](assets/1705982064-c966b4e9999ce7d8f5de0a8aa701a37c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240121114737-d1d929e6-b80f-1.png)

[![](assets/1705982064-f8e8191e222d9b12972191e79cd259ec.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240121114743-d4ee1114-b80f-1.png)

-   接下来对剩余的脱壳后的 dex 进行重命名。然后将 dex 添加到 apk 项目中。注意保存的时候一定**不要自动签名！**

[![](assets/1705982064-c8375ba161ad0a2f13fc42f50204d1cf.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240121114756-dceba1ba-b80f-1.png)

[![](assets/1705982064-c478601c0ed333c11eb75e631a113d11.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240121114802-e0a3113a-b80f-1.png)

-   删除加固残留。删除根目录，lib，assets 加固特征码文件。so 文件中带 shell 的删除掉，assert 中带 0O1 和 00OO 这种的。

[![](assets/1705982064-ea21dac9d333267c207dd5ec71994c51.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240121115038-3da6f676-b810-1.png)

-   此时，再用 MT 管理器查看显示未加固，但是有校验未通过。直接用 NP 管理器去除签名校验。

[![](assets/1705982064-b25289bd86282788a7a7d45a31ad68a2.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240121114809-e4a24756-b80f-1.png)

[![](assets/1705982064-2cd54a148f96be0df5adf820b75f146e.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240121114815-e83e9932-b80f-1.png)

-   安装，发现可以正常安装并使用，到此去除腾讯御安全加固就可以了。

[![](assets/1705982064-3b1e1187ba60a61c2e17a5a1aa82b095.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240121114921-0f9a5fb6-b810-1.png)

[![](assets/1705982064-2f1aaa391270e0f0eddb2c1328760f47.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240121114925-120bdf68-b810-1.png)

-   之前的内容对加固的 app 进行简单的分析和脱壳。这里继续分析此 app，进行关闭广告、跳过强制更新。  
    \## 应用分析
    
-   登录
    
    -   登录的时候可以使用密码和验证码登录，这里就抓包分析使用密码登录的逻辑。登录的逻辑在`com.dalongtech.cloud.app.quicklogin` 代码中。
    -   通过抓包的分析，登录请求中对手机号没有做任何处理，对密码进行了简单的编码。响应也是没经过任何处理的。

[![](assets/1705982064-d715d268b6f041f4ad2c400ee28b60d1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240121203954-2d85ae00-b85a-1.png)

### 去除广告页

-   登录后遇到的第一个问题是广告界面。
    -   `com.dalongtech.cloud.app.appstarter.``AppStarterActivity` 界面是广告 activity。
    -   在上面 activity 找了半天，没找到广告计时器定义在哪。只能通过 amf.xml 文件看看是否可以跳过这个广告 activity。这里将下面图中的 intent 过滤器复制到广告后的下一个 activity：`HomePageActivityNew`

[![](assets/1705982064-d86b53620294f59bd2f092aed3c3d1cb.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240121204025-3fcce2ea-b85a-1.png)

-   进行了上面的更改后可以不加载广告界面直接进入主界面。

[![](assets/1705982064-093e0dc15bab00f0a730af397776f12c.gif)](https://xzfile.aliyuncs.com/media/upload/picture/20240121204110-5a9e7660-b85a-1.gif)

### 饶过强制更新

-   遇到的第二个问题：强制更新，无法取消弹窗。而且无法点击以后再说

[![](assets/1705982064-c2e19b00779ce2e95c784bc511c4d8ee.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240121204209-7e3e3182-b85a-1.png)

-   根据当前 activity 提示，发现更新弹窗的逻辑代码在`com.dalongtech.cloud.wiget.dialog.``UpdateAppDialog`
-   经过分析对弹窗 activity 的分析，发现了一个可以控制按钮状态的函数。其中**\`this**.f12837i`对应的是`(Button) a(R.id.update\_dialog\_cancelBtn)\` ，即取消更新的 button。如果没猜错的话，z 应该是 False，这里使用 frida 验证一下我们的猜想。

[![](assets/1705982064-9d6720ddf9e0dc77d930665fdcd22b6f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240121204227-88c97ea4-b85a-1.png)

-   frida hook 函数
    
    ```plain
    function carhook3(){
           let UpdateAppDialog = Java.use("com.dalongtech.cloud.wiget.dialog.UpdateAppDialog");
           UpdateAppDialog["c"].overload('boolean').implementation = function (z) {
           console.log('c is called' + ', ' + 'z: ' + z);
           let ret = this.c(z);
           console.log('c ret value is ' + ret);
           return ret;
       };
       };
    ```
    
-   z 的返回结果，可以看出是这里让取消不能点击。现在有两个方法。一个是 hook 函数的时候修改 z 的值为 true。另一个是在 smali 代码中对 z 取反，为了让修改的持久，这里选择直接修复 smali 代码。
    

[![](assets/1705982064-6f1bfaee5a6fbb03cc207f2aa986683a.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240121204243-92855ea4-b85a-1.png)

-   针对使用 hook 修改 c 函数的入参的方式，经过尝试也是可行的

[![](assets/1705982064-6ac05708a0e61472822b34fa798d0c71.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240121204257-9a54f432-b85a-1.png)

-   查找 c 函数的调用，发现只有一处，**\`this**.f12120c.c(!"1".equals(data.getIs\_force()));\` 在 smali 中将取反操作去掉就可以了。我们可以将 0x1 变为 0x0 或者直接删除这一句。
    
    ```plain
    xor-int/lit8 v0, v0, 0x1
       指令将 v0 寄存器的值与 1 进行按位异或操作，相当于对 v0 寄存器的最低位（最后一位）进行取反操作。
    ```
    
-   修改后可以看出，以后点击可以点击了，而且点击后可以暂时不更新。
    

[![](assets/1705982064-9ae28eca2a4829addd21370552607527.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240121204311-a2c5bb2e-b85a-1.png)
