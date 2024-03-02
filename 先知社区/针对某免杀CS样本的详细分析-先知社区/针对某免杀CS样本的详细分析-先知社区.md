

# 针对某免杀 CS 样本的详细分析 - 先知社区

针对某免杀 CS 样本的详细分析

- - -

# 前言概述

某微信好友求助笔者，可能自己中招了，让笔者看看是啥东西，具体做了什么，如下所示：  
[![](assets/1707954719-6fcc2591ffba7bad7e6219eca029dacb.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211112914-bad6fa0a-c88d-1.png)  
随后发来了样本和解压密码，如下所示：  
[![](assets/1707954719-5ca989772b8f5cdff550f82c82b24048.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211112928-c33c2954-c88d-1.png)  
笔者通过 VT 查了一下，发现 VT 上已经有样本了，但是检出率还挺低的，如下所示：  
[![](assets/1707954719-2fa3d1c00a9484adfaf87218cc034276.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211112959-d5a6c1b2-c88d-1.png)  
这么低的检出率，难倒是样本给错了？这瞬间激起了笔者想一探研究的兴趣，笔者平时没事的时候就喜欢研究一些有趣的好玩的对抗型攻击样本，如果样本在 VT 已经标记的很清楚了，我倒真没啥兴趣分析，越有趣越好玩越高级越新鲜的对抗型攻击样本，我反而是越有兴趣深入分析研究，今天就让我们看看它究竟是什么？

# 详细分析

1.样本解压之后，如下所示：  
[![](assets/1707954719-171516fafbf2a93586ac5bd5243b7fe4.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211113044-f02faf6c-c88d-1.png)  
2.里面包含一个 EXE 程序和一个 XML 文件，XML 文件内容，如下所示：  
[![](assets/1707954719-2bd7c00fb850d5c778a7d9542f0c392f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211113108-fec39b92-c88d-1.png)  
好像没什么特别的内容，为啥样本压缩包里会包含这个 XML 文件呢？难倒是样本运行之后会检测这个 XML 文件？

3.通过分析 EXE 程序，发现该程序确实会针对 XML 文件进行相关操作，如下所示：  
[![](assets/1707954719-6cf85669e8c965f1fe759af3fbe5ae6d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211113138-10c9cb90-c88e-1.png)  
4.该 EXE 程序会检测 XML 文件，检测到 XML 文件，则跳转到恶意代码，如下所示：  
[![](assets/1707954719-14142a2d9aa8aa90fcda137052f56b1b.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211113205-20b4ce88-c88e-1.png)  
5.恶意代码前面是一段数据解密操作，获取核心函数地址，后面解密 Payload 的核心代码，如下所示：  
[![](assets/1707954719-651eee94b10b77fe2889d589a306e8ee.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211113226-2d87a3c4-c88e-1.png)  
6.分配内存空间，用于解密 ShellCode 数据，如下所示：  
[![](assets/1707954719-0e1c11365dc58f9b397c220ea28f70ad.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211113251-3c5bff8a-c88e-1.png)  
7.解密 ShellCode 数据，如下所示：  
[![](assets/1707954719-6999ee231a1e3593c7342b278655b551.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211113324-4fe53cb0-c88e-1.png)  
8.将解密的 ShellCode 数据移动到此前分配的内存空间，如下所示：  
[![](assets/1707954719-11fdb599bb85b37c3661f5bd7f2cce9e.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211113431-77f560fe-c88e-1.png)  
9.最后通过 CreateThreadpoolWait 执行解密的 ShellCode 代码，如下所示：  
[![](assets/1707954719-f9ff9d1248288f55e333e315243c0c23.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211113500-88de83b4-c88e-1.png)  
10.跳转到 ShellCode 代码执行，如下所示：  
[![](assets/1707954719-80763372f1648710dd2fb779e18554db.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211113524-9736e6fe-c88e-1.png)  
11.解密出第二段 ShellCode 代码，如下所示：  
[![](assets/1707954719-22b15465c3d048cbdc5dfc477bea2db7.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211113547-a4fa355c-c88e-1.png)  
12.执行第二段 ShellCode 代码，加载执行里面的 Payload 代码，如下所示：  
[![](assets/1707954719-ffa09bb4b20195f52c2f7142496310fb.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211113610-b2e042e2-c88e-1.png)  
13.Payload 代码与此前母体核心代码相似，分配内存空间，用于存放第三阶段的 ShellCode 代码，如下所示：  
[![](assets/1707954719-18aff53d809a46ce56a60c85f4f0a0bd.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211113633-c0a28d2c-c88e-1.png)  
14.解密第三阶段的 ShellCode 代码，如下所示：  
[![](assets/1707954719-a4e6f6db69e0585a2853efdcbfdee4d6.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211113658-cf6157bc-c88e-1.png)  
15.解密出来的第三阶段的 ShellCode 代码，如下所示：  
[![](assets/1707954719-31c77f7c90897f412232b98d7b2decf2.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211113723-de898d68-c88e-1.png)  
16.第三阶段的 ShellCode 代码，从黑客远程服务器读取第四阶段 ShellCode 代码，然后在内存中加载执行，如下所示：  
[![](assets/1707954719-57acbbd556d7bfb6dd0b4deaa0e9dcb4.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211113748-ed5bdbde-c88e-1.png)  
17.ShellCode 代码，如下所示：  
[![](assets/1707954719-9d406729eacdec0f5f37930f89c5be81.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211113810-fa32c372-c88e-1.png)  
18.通过分析发现该 ShellCode 代码通过异或算法解密出第五阶段的 ShellCode 代码并加载执行，如下所示：  
[![](assets/1707954719-6ab00c6290332cae576f6d864114f105.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211113834-08ace996-c88f-1.png)  
19.该 ShellCode 加载执行里面的 CS 木马，如下所示：  
[![](assets/1707954719-ac78e19d4f22d2357111ac978cbe3ef4.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211113859-173d3218-c88f-1.png)  
20.解析出该 CS 的配置信息，如下所示：  
[![](assets/1707954719-c7bf9f0b34b898824678daa2405d3896.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211113934-2ca0db78-c88f-1.png)

# 关联分析

通过该域名关联到相关的样本，如下所示：  
[![](assets/1707954719-01ddd68f49b12866db6aa6d8f21eda97.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211114026-4b514198-c88f-1.png)  
分析关联到的样本 123.exe，编译时间为 2023 年 8 月 14 日，如下所示：  
[![](assets/1707954719-528598aac10e255626857e62cec59732.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211114052-5aa37008-c88f-1.png)  
该样本就是上面母体样本解密出来的第二阶段 ShellCode 里面包含的 Payload 代码，如下所示：  
[![](assets/1707954719-71806957ea2f66c426f0f16c4fbd45db.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211114625-21942cac-c890-1.png)  
分析下面两个 ProgramData 压缩包的样本，发现里面的 EXE 是一样的，与上面的样本一样都包含一个 XML 文件，如下所示：  
[![](assets/1707954719-27adb2f4b3ef31d941f595de6374c6b7.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211114656-33b78258-c890-1.png)  
样本的编译时间为 2023 年 8 月 7 日，应该是比上面母体样本更早的攻击样本，如下所示：  
[![](assets/1707954719-8a510180663af0c6af3b96b2d79f4d66.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211115506-57982bfe-c891-1.png)  
该样本需要加载读取 XML 文件内容才能执行后面的恶意代码，如果没有读取到 XML 文件相应的内容，则直接报错误，如下所示：  
[![](assets/1707954719-1b610bafa891f443827941dbebb1f946.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211115530-66049222-c891-1.png)  
加载 XML 成功之后，执行跳转到后面的恶意代码，如下所示：  
[![](assets/1707954719-6aecaa9ccd7e7b0a0fca97c611d6c626.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211115559-7760f95c-c891-1.png)  
恶意代码解密 ShellCode 代码，如下所示：  
[![](assets/1707954719-fc62d421cd64ae5dbf1dbc1b0617c30e.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211115623-86019034-c891-1.png)  
解密出来的 ShellCode 代码，如下所示：  
[![](assets/1707954719-201bc6996fa160d81bb04d41bdac82c9.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211115659-9b1c2880-c891-1.png)  
最后通过 CreateThread 执行 ShellCode 代码，如下所示：  
[![](assets/1707954719-9e4b08deec0982f4bc00d661a4f97866.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211115722-a8e375b8-c891-1.png)  
该样本解密出来的 ShellCode 代码，与上面样本解密出来的第三阶段的 ShellCode 代码基本一致，如下所示：  
[![](assets/1707954719-ff3f06b1903736e94578f9ae5b521a0c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211115744-b5d8126a-c891-1.png)  
通过上面的分析，可以得出此前我们分析的样本应该是关联到样本的更新版本，关联到的样本编译时间为 2023 年 8 月 07 日，我们分析的样本编译时间为 2023 年 9 月 11 日，同时去年八九月份正好在进行相关的攻防演练，可以判断该样本大概率是某个公鸡队的样本。

# 威胁情报

[![](assets/1707954719-2e85f29123a91a33ae55bfe36acb4f85.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211115822-ccdfd790-c891-1.png)

# 总结

做安全，免杀是一个永恒的话题，是一场猫捉老鼠的游戏，通过研究一些对抗型的攻击样本，可以更好的了解攻击者在使用什么技术。
