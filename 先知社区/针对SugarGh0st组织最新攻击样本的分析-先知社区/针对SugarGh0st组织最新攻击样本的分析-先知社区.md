

# 针对 SugarGh0st 组织最新攻击样本的分析 - 先知社区

针对 SugarGh0st 组织最新攻击样本的分析

- - -

# 前言概述

2023 年 11 月份 Cisco Talos 发现了一起针对乌兹别克斯坦外交部和韩国用户的攻击活动，攻击活动使用了一种新型的 RAT 恶意软件 SugarGh0st，SugarGh0st 是 Gh0st RAT 的最新的变种之一。

2023 年 12 月份哈萨克斯坦国家技术服务中心发现 SugarGh0st 组织针对哈萨克斯坦进行大规模网络钓鱼的攻击活动，该攻击活动使用了 SugarGh0st 在 2023 年 12 月更新的最新攻击样本。

对比 2023 年 11 月 SugarGh0st 组织的攻击样本，2023 年 12 月的攻击样本主要在免杀方面进行了相关的更新，加载脚本变成了 VBS 脚本，利用 Windows 登录自启动来加载执行 DLL 恶意模块，同时增加了相应的免杀技术，对 DLL 文件进行增肥，加壳等技术手段以逃避安全软件的检测。  
[![](assets/1708507117-1c305354db10631971f97fa935a5614e.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240220204639-175084f6-cfee-1.png)  
笔者从 vx-underground 下载到最新的攻击样本，并针对 SugarGh0st 组织最新的攻击样本进行了相关的分析。

# 样本分析

1.钓鱼压缩包自解压之后，会在相应的目录生成几个恶意文件，如下所示：  
[![](assets/1708507117-11272204cb0a8f40c0a45c735d733f86.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240220204707-283bcdfc-cfee-1.png)  
2.自解压之后会调用 VBS 脚本，VBS 脚本会注册恶意 DLL 模块为 Windows 登录自启动脚本来建立持久性，如下所示：  
[![](assets/1708507117-4bfe489b03d4ec64923ea2b06e58851a.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240220204721-308d5958-cfee-1.png)  
该攻击技术曾被多个 APT 组织使用，相关 ATT&CK 实例与 APT 攻击组织，如下所示：  
[![](assets/1708507117-c3484b5e38ac1638b92947dfa4b5a497.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240220204739-3afea9dc-cfee-1.png)  
3.加载的恶意 DLL 模块 update.dll 有两百多 M，并使用 VMP 加壳处理，如下所示：  
[![](assets/1708507117-9b3d0159b0b1995738cce4397d59b4d7.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240220204755-446b25d6-cfee-1.png)  
4.VMP 入口点代码特征，如下所示：  
[![](assets/1708507117-d824e2619398e010605e0d3d4666ed38.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240220204807-4c030052-cfee-1.png)  
5.动态调试，脱壳之后，到达恶意 DLL 模块的入口点处，如下所示：  
[![](assets/1708507117-5712ad747b863e614a8cfbf38eaaa288.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240220204822-54ea310e-cfee-1.png)  
6.读取 Temp 目录下的 authz.lib 文件内容，并在内存中解密，如下所示：  
[![](assets/1708507117-71fe7fd34fcc13dcfd4afbd76fc91bf6.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240220204839-5eb7a11c-cfee-1.png)  
7.动态调试，读取 authz.lib 文件内容到内存，如下所示：  
[![](assets/1708507117-bc4a033ad92f881ac8141854ce39382d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240220204852-66d52554-cfee-1.png)  
8.异或解密文件内容，如下所示：  
[![](assets/1708507117-da359413599dd47e1316be89cbc9c8b5.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240220204906-6ee14322-cfee-1.png)  
9.解密之后，如下所示：  
[![](assets/1708507117-16f82fc8d2538e8b0868c35aeaac69ef.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240220204919-76a550f8-cfee-1.png)  
10.最后跳转到解密的 shellcode 代码执行，如下所示：  
[![](assets/1708507117-9fbecff43ed66c18fa07739ecef5b85f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240220204937-8126c69c-cfee-1.png)  
11.shellcode 获取关键函数地址，然后调用 VirtualAlloc 函数分配内存空间，如下所示：  
[![](assets/1708507117-a9fd6061f2a9566134d651a4bab50ce1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240220204951-89bdbb9e-cfee-1.png)  
12.解密出 payload 数据到内存中，如下所示：  
[![](assets/1708507117-0e209d5e357e5deeb17c1228629e8456.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240220205010-94e85588-cfee-1.png)  
13.解密出来的 payload 就是 SugarGh0st RAT，与此前 Cisco Talos 发布的 payload 相似，创建一个互斥变量，变量名就是该 RAT 的 C2 域名，如下所示：  
[![](assets/1708507117-95abb77d6591688571b5b8dd40b5623c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240220205027-9f65ecd2-cfee-1.png)  
14.启动键盘记录功能，在相应目录下生成记录文件，如下所示：  
[![](assets/1708507117-c5f4fbb23a221f120ded5b1d2dc9bb2d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240220205041-a7894422-cfee-1.png)  
15.硬编码记录了样本的编译时间为 2023 年 12 月，如下所示：  
[![](assets/1708507117-f9c6b6a6f9124323254ca9c93bdffdc5.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240220205054-af8d3dea-cfee-1.png)  
16.通过硬编码的 C2 域名和端口，与黑客远程服务器建立连接，如下所示：  
[![](assets/1708507117-976989e02cd58d5e8b5253419310669a.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240220205108-b7dcac06-cfee-1.png)  
17.对进程进行提权相关操作，如下所示：  
[![](assets/1708507117-33d219845ef5db2cb7950937991120c7.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240220205122-bffb7728-cfee-1.png)  
18.获取主机相关信息发送到远程服务器，然后执行 C2 命令通信，如下所示：  
[![](assets/1708507117-8329fdf5bd15b8e37657b8b67665af0c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240220205136-c8384952-cfee-1.png)  
19.文件操作相关的 C2 命令，如下所示：  
[![](assets/1708507117-dd8cee97c60533e8ca0c12a99786952d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240220205150-d066f3e4-cfee-1.png)  
其他 C2 指令可以参考 Cisco Talos 此前发布的相关报告，这里就不一一列举了。

# 总结结尾

安全对抗会一直持续存在，有攻必有防，攻与防就是矛与盾的关系，攻击者会不断更新自己的攻击样本和攻击技术，持续对抗安全产品的检测和查杀，安全研究人员需要持续不断的提升自己的安全能力，深入研究这些对抗技术，才能做到知已知彼，更快的发现这些安全问题。

笔者一直从事与恶意软件威胁情报相关的安全分析与研究工作，包含各种各样的不同类型的恶意软件，通过深度分析和研究这些恶意软件，了解全球黑客组织最新的攻击技术以及攻击趋势。
