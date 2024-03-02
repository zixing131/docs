

# Android 第一代落地 DEX 加固总结 - 先知社区

Android 第一代落地 DEX 加固总结

- - -

第一代 dex 落地壳加固是 Android 加固的基础，近日学习加固相关知识，记录一下自己踩过的坑和总结自己对落地壳的理解。  
下面的代码或多或少参考了大佬们的代码，主要是加上自己的理解和注释，经过折腾最终在最新版雷电模拟器 9（Android9）实现了加固和自解密运行。  
参考项目和文章会放在末尾。

## 1、理论知识

关于 DEX 文件的组成，可以自行搜索详细学习  
在 Android 的 DEX 加固中，主要和 DEX 头部三个字段有关，分别是：

-   checksum  
    文件校验码，使用 alder32 算法校验文件除去 maigc，checksum 外余下的所有文件区域
-   signature  
    使用 SHA-1 算法 hash 除去 magic、checksum 和 signature 外余下的所有文件区域
-   file\_size  
    dex 文件的大小  
    DEX 加固主要就是一个拼接  
    成品 = 待加固 app（会经过加密） + 解密 dex  
    经过拼接之后，会影响上面说的 3 个字段，所以需要我们手动去修改更正。

[![](assets/1706771621-6277b37f84666cd1393d6aaef817324f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240129164440-a434f668-be82-1.png)

这里借用 Jack\_Jia 大佬的图

我们一般把加密后的 apk 放在解密 dex 的末尾，并且最后 4 个字节填充为源 apk 的大小，方便我们解密然后运行该 app。

主要的过程总结如下：

### 加密源 app：

1、加密源 app  
2、把加密后的源 app 放在壳 (classes.dex) 后面（最后面 4 个字节填充源 app 大小）  
3、修改壳的 checksum、signature 和 file\_size

### 解壳 apk

1、主动读取 classes.dex 末端的加密数据  
2、调用本 apk 的解密方法进行解密  
3、动态加载解密之后的 apk

主要的过程很好理解，但是有一些细节需要我们去注意。

### 如何加载且正常运行解密之后的 apk？

我们使用 dexclassloader 来加载我们的 apk，为了让加载的 apk 有自己的启动流程和生命周期，将我们的 dexclassloader 替换 LoadedApk 中的 mClassLoader 这个字段  
mClassLoader 用于加载 APK，动态运行我们解密之后的 apk

[![](assets/1706771621-dcef948e8e51d8e31e8df29562cfc1ca.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240129164507-b477f7e6-be82-1.png)

```plain
RefInvoke.setFieldOjbect("android.app.LoadedApk","mClassLoader",weakReference.get(),DexClassLoader);
```

## 2、加密和组装

使用 java 语言来完成加密源 app 这一目的，粘太多代码不太好  
这里先解释一下 System.arraycopy 方法

System.arraycopy 方法的原型如下：

```plain
public static void arraycopy(Object src, int srcPos, Object dest, int destPos, int length)
```

这个方法用于将一个数组中的元素复制到另一个数组中，可以用于复制基本数据类型数组和对象数组。参数的含义如下：

src: 源数组，从这个数组中复制元素。  
srcPos: 源数组中开始复制的位置（索引）。  
dest: 目标数组，将元素复制到这个数组中。  
destPos: 目标数组中开始粘贴的位置（索引）。  
length: 要复制的元素数量。  
首先我们先获取原 apk 和壳 dex，然后进行修改 3 个字段，最后保存即可。

```plain
public static void main(String[] args) {

    try {
    //目录自定义
        File payloadSrcFile = new File("F:\AndroidUnidbg\untitled\src\orgin.apk");//原apk目录
        File unShellDexFile = new File("F:\AndroidUnidbg\untitled\src\shell.dex");//具有加固功能的dex文件
        byte[] payloadArray = encrpt(readFileBytes(payloadSrcFile),0x66);//对apk进行加密，这里可以自定义，我采用了异或加密

        byte[] unShellDexArray = readFileBytes(unShellDexFile);//readFileBytes函数：将文件内容转换为bytes类型

        int payloadLen = payloadArray.length;
        int unShellDexLen = unShellDexArray.length;
        int totalLen = payloadLen + unShellDexLen +4;//可以看到最终的长度是增加了4，最后4个字节用于填充源app文件的大小
        byte[] newdex = new byte[totalLen];

        //添加解壳代码
        System.arraycopy(unShellDexArray, 0, newdex, 0, unShellDexLen);
        //添加加密后的解壳数据
        System.arraycopy(payloadArray, 0, newdex, unShellDexLen, payloadLen);
        //添加解壳数据长度
        System.arraycopy(intToByte(payloadLen), 0, newdex, totalLen-4, 4);
        //修改DEX file size文件头
        fixFileSizeHeader(newdex);
        //修改DEX SHA1 文件头
        fixSHA1Header(newdex);
        //修改DEX CheckSum文件头
        fixCheckSumHeader(newdex);


        String str = "F:\AndroidUnidbg\untitled\src\classes.dex";
        File file = new File(str);
        if (!file.exists()) {
            file.createNewFile();
        }

        FileOutputStream localFileOutputStream = new FileOutputStream(str); //保存加密之后的dex
        localFileOutputStream.write(newdex);
        localFileOutputStream.flush();
        localFileOutputStream.close();


    } catch (Exception e) {

        e.printStackTrace();
    }
}
```

## 3、解密 apk

这一步可能会难住许多朋友，因为有一些小细节需要我们去注意。小细节导致我们加固实验不能正常进行。  
这里记录一下自己踩得坑，详细的加固代码在 github 会有很多。

1、解壳 app 其实不需要有启动界面的，我们可以直接在 androidmanifest.xml

[![](assets/1706771621-fad86a18d0a0e7b27a976c3c7696393b.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240129164648-f04083c4-be82-1.png)  
这里是解壳 APP 的 androidmanifest.xml，可以看到主启动 activity 为我们待加固 app 的入口

[![](assets/1706771621-92c29530164c1681288261ed8eaa896e.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240129164704-fa14bf96-be82-1.png)  
因为壳 apk 不需要进入 UI 界面交互，需要在壳 apk 运行起来，对源 app 进行解密然后运行。这里我们直接替换，后面也不需要进行修改了。

2、解壳 app 的 androidmanifest.xml 增加一个描述信息，用于在动态运行的时候，将壳的 application 替换为源 app 的 application，完成 app 的正常启动

```plain
<meta-data
    android:name="APPLICATION_CLASS_NAME"
    android:value="com.android.sourceapp.XApplication"/>
```

[![](assets/1706771621-96ff9cac08ca1d3cb972c8054cb4952c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240129164727-07885a3e-be83-1.png)  
这其实导致了我失败很多次，我一直都没搞明白，这个 com.android.sourceapp.XApplication 指的是源 app 的 Activity 还是壳的 Activity，经过修改总是失败，最后在大佬的帮助下才明白，这个是源 app 的一个类  
他的定义如下：  
甚至不需要做什么操作，只是作为源 app 加载的入口（如果这里写成 MainActivity，经过实验发现失败）

```plain
package com.android.sourceapp;

import android.app.Application;

public class XApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
    }
}
```

3、我们将这些做完之后，使用壳程序进行加密，生成加密之后的 dex，直接替换原 app，不用签名安装。

[![](assets/1706771621-2f17129ff34aa3d6c6662f7011b0e7eb.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240129164800-1b8199d8-be83-1.png)

[![](assets/1706771621-290a55fe25b4cd6b6d661c4bef037255.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240129164812-224b5f60-be83-1.png)

## 4、实战运行结果

打开壳 apk

[![](assets/1706771621-9a4ba9706c6410bfa533e6d3da92dde5.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240129164837-319723f0-be83-1.png)

稍等 1 秒后（第一次运行会进行解密，第二次就直接运行解密之后的 apk 了，因此叫落地加载）

[![](assets/1706771621-22e8227f7bcf9ee013800ca0654a10e7.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240129164855-3c66f3d2-be83-1.png)

## 5、参考

Jack\_Jia：Android APK 加壳技术方案【1】【2】
