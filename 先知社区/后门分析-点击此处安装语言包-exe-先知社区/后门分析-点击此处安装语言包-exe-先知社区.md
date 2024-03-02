

# 后门分析---点击此处安装语言包.exe - 先知社区

后门分析---点击此处安装语言包.exe

- - -

# 前言概述

笔者在日常使用一些社交软件的时候，总是会遇到在一些群里发一些安装语言包之类的程序，又想骗我安装后门。

提醒大家，不管是群里发的各种破解软件、安装程序、安全工具，还是从 GitHub 等开源网站上，下载的各种安全工具、POC 等，只要是非正式官方上的各种安装程序，大家在下载安装使用的时候，都不要随意点击安装，可能一不小心，就被安装上了后门，但如果遇到了供应链攻击，就算是从官方渠道下载的软件可能都隐藏后门了，安全攻击，无处不在，防不甚防，哈哈哈哈，今天笔者给大家分享一下这些安装语言包背后隐藏的后门程序。

# 攻击流程

黑客组织攻击流程图，如下所示：  
[![](assets/1708920526-5cfe625f7f95ce41ba26fcd3e50b9c68.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211143113-2708cb40-c8a7-1.png)

# 详细分析

1.样本从远程服务器上下载加密的压缩数据包，如下所示：  
[![](assets/1708920526-d43bd74ac34a606f0de3cd85f95e55bd.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211143146-3ad5d32a-c8a7-1.png)  
2.加密的压缩包下载到临时目录下，如下所示：  
[![](assets/1708920526-f5fc919007cdacefb0d80cc9619fed53.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211143208-47ab2708-c8a7-1.png)  
3.读取加密的压缩包文件，然后通过硬编码的密钥信息解密压缩包数据，如下所示：  
[![](assets/1708920526-8088a66f669e026260a44a0906f4aadc.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211143231-55d11838-c8a7-1.png)  
4.将加密的压缩包数据逐字节读取到内存中，然后解密，如下所示：  
[![](assets/1708920526-7866c4b794b1be0d47c095801c5769a4.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211143304-697a3edc-c8a7-1.png)  
5.然后将解密的数据写入到另外一个压缩包文件当中，如下所示：  
[![](assets/1708920526-1fa7e89231dc1cce24cf3465b2c3b765.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211143326-764a9dfa-c8a7-1.png)  
6.解密过程，如下所示：  
[![](assets/1708920526-49e61a0667cce3d96da20280938abfd8.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211143346-8259bc98-c8a7-1.png)  
7.将解密的压缩包，解压缩释放到公共目录下，如下所示：  
[![](assets/1708920526-e20885794aa202e6366863d82a74aaa9.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211143405-8dc916e6-c8a7-1.png)  
8.最后调用该目录下的主程序 tapisrv.exe，通过白 + 黑的方式加载同目录下的恶意模块 IvsDrawer.dll，如下所示：  
[![](assets/1708920526-560a5e59d4b85e39345ea653c2497bb9.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211143428-9b37239a-c8a7-1.png)  
9.IvsDrawer.dll 导出函数调用目录下的 D3D.dll 的导出函数，如下所示：  
[![](assets/1708920526-356c6cc5bd99c97a558d654663f0da4d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211143448-a7790bf0-c8a7-1.png)  
10.DRAW\_Startup 和 DRAW\_InputTrackData 导出函数会执行相关的恶意操作，如下所示：  
[![](assets/1708920526-c2eff0c97a22e5d8438cd918a4670865.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211143511-b4b35884-c8a7-1.png)  
11.读取目录下的 donottrace.txt 加密数据文件，如下所示：  
[![](assets/1708920526-ca7217fc81f015122a0df8e67f0c74da.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211143531-c08e61b2-c8a7-1.png)  
12.将加密的数据读取到分配的内存当中，如下所示：  
[![](assets/1708920526-58a5336eaed1d78115d871e1a94d4580.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211143551-ccbae65e-c8a7-1.png)  
13.调用解密函数解密该内存数据，如下所示：  
[![](assets/1708920526-b149effc379e19a9beacb6de513b2d04.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211143704-f84ea38c-c8a7-1.png)  
14.解密后的数据，如下所示：  
[![](assets/1708920526-c372b924c41542d02752b7adaf541f52.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211143731-08187c2a-c8a8-1.png)  
15.该模块的导出函数，如下所示：  
[![](assets/1708920526-7dd5c967621c4ea08c3c458723eb791f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211143805-1c6a868c-c8a8-1.png)  
16.读取同目录下的 task.dat 文件内容，然后安装计划任务自启动项，如下所示：  
[![](assets/1708920526-8caf86b76d4068b8194ad275a463eade.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211143828-2a9b96a6-c8a8-1.png)  
17.然后利用程序中的硬编码密钥，解密程序中的相关数据，如下所示：  
[![](assets/1708920526-f86348eb80898a8f7f718cd5c835d6c1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211143851-3824ca7c-c8a8-1.png)  
18.解密之后，如下所示：  
[![](assets/1708920526-993d4030f2b9c208b24b1ec78b303e2a.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211143911-43d7f330-c8a8-1.png)  
19.解密出来的 PayLoad 是一个 Gh0st 修改版，如下所示：  
[![](assets/1708920526-57778d6a2ac8234a963847d8965eebbc.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211143932-50a95216-c8a8-1.png)  
创建两个线程，如下所示：  
[![](assets/1708920526-780fb9c9bf36f11ce1c3761887da9d3e.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211143954-5d8f85ea-c8a8-1.png)  
20.一个线程，通过读取远程 pastebin.com 服务器的数据，如下所示：  
[![](assets/1708920526-6c1e5dfb3ce5c92e08ccacc9aa779380.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211144017-6b762204-c8a8-1.png)  
21.远程 pastebin.com 服务器的数据，如下所示：  
[![](assets/1708920526-4e7d34ec526e32d2d2b353b462259f9f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211144040-78d86f7e-c8a8-1.png)  
22.或者通过解密自身硬编码的数据，解密出远服务器的 C2：hero.gettimi.top，如下所示：  
[![](assets/1708920526-c20fdefcabd0f0965c905543f7c6e2ec.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211144103-86b0d082-c8a8-1.png)  
23.与远程 C2 服务器进行通信，如下所示：  
[![](assets/1708920526-ff3f897a6890902cb31fd0a94a597ee9.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211144125-93c6fcc4-c8a8-1.png)  
24.另外一个通过解密自身硬编码的数据，解密出远服务器的 C2：news.cookielive.top，如下所示：  
[![](assets/1708920526-bcb5c3f2dabe2bec8f653a1c5c77a7bc.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211144144-9f29027e-c8a8-1.png)  
25.硬编码数据的解密函数，如下所示：  
[![](assets/1708920526-8efd393620b0c7e9d8be953fbb834a0e.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211144205-abc4a3c6-c8a8-1.png)

# 关联分析

通过下载链接中 link.jscdn.cn 域名，可以关联到一堆相关的样本，如下所示：  
[![](assets/1708920526-a0c2137ff046ce3543b78e34b0910085.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211144240-c0754be0-c8a8-1.png)  
查询解密出来的域名 hero.gettimi.top，关联到另外一个样本，如下所示：  
[![](assets/1708920526-e945a731424eae630190644538ce7ddb.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211144303-ce0fdd60-c8a8-1.png)  
关联样本下载解压缩之后，如下所示：  
[![](assets/1708920526-abad7280b3793e95f5d41363611b9120.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211144328-dd601546-c8a8-1.png)  
加载的恶意模块 pbvm90.dll 的代码与上面的 IvsDrawer.dll 模块代码基本一致，使用的 hash 不一样，如下所示：  
[![](assets/1708920526-22211dfabd44c452c40c9dfad5d299e7.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211144357-ee5696c2-c8a8-1.png)  
该样本从远程服务器上下载加密的压缩包数据，从下载的链接可以发现，最新的样本使用了与此前样本不同的远程服务器，说明黑客组织一直在更新自己的攻击样本，如下所示：  
[![](assets/1708920526-66b5ff9bb2ebeac520c7c7477c408476.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211144431-02a6e398-c8a9-1.png)  
查询解密出来的域名 news.cookielive.top，同样关联到一个样本，如下所示：  
[![](assets/1708920526-a570531e23580849dc3220f7bae260e1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211144453-10010820-c8a9-1.png)  
关联的样本解压缩之后，如下所示：  
[![](assets/1708920526-28b6e7955814240c3e1e8a8d8af29a2c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211144521-202fb9da-c8a9-1.png)  
加载的恶意模块 AndroidAssistHelper.dll 的代码与上面的 IvsDrawer.dll 模块代码基本一致，如下所示：  
[![](assets/1708920526-7b838826953fb0228b720006aaa165a5.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211144542-2d1ec30c-c8a9-1.png)  
使用的 hash 也是一样的，如下所示：  
[![](assets/1708920526-77bc8616812ce46ed497766fed59d260.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211144604-3a60c772-c8a9-1.png)  
通过上面的关联分析，可以发现黑客组织一直在更新自己的攻击样本，使用不同的下载服务器以及不同的软件加载方式等，同时通过域名关联，可以发现该攻击活动非常频繁，不仅仅使用语言安装包的方式，还使用了其他各种工具软件等进行传播。

# 威胁情报

[![](assets/1708920526-8ca3fc4d1619b5a79834a26ece364d0b.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211144632-4b0dcbd8-c8a9-1.png)

# 检测规则

[![](assets/1708920526-4901e120df71f30170955bab0595a3a3.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240211144656-59276012-c8a9-1.png)

# 结尾

黑客组织利用各种恶意软件进行的各种攻击活动已经无处不在，防不胜防，很多系统可能已经被感染了各种恶意软件，全球各地每天都在发生各种恶意软件攻击活动，这些攻击活动主要包含：勒索攻击、APT 窃密攻击等，笔者最近几年专注于针对勒索病毒黑客组织、APT 定向攻击黑客组织、以及各种黑灰产黑客组织进行跟踪分析和研究，发现这些组织一直在持续更新自己的攻击样本以及攻击技术，不断有企业被攻击，这些黑客组织从来没有停止过攻击活动，而且非常活跃，持续不断地更新攻击样本，采用新的攻击技术。

未来黑客会研究和采用更高级的攻击手法，使用更高级的攻击样本和攻击技术，会开发更为复杂的恶意软件，会使用更隐藏的免杀植入方式，会挖掘更多新的安全漏洞，安全对抗没有终点，如果想在安全行业走的更远，就踏踏实实不断提升自己的能力，黑客组织也在不断进步，安全从业人员更需要持续不断的学习进步，才能抵御未来各种网络安全攻击，而且未来高端的安全对抗会越来越激烈，安全厂商和安全研究人员需要持续不断的提升自己的安全能力。
