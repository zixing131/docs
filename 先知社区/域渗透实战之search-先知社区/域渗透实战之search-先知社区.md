

# 域渗透实战之 search - 先知社区

域渗透实战之 search

- - -

# 信息收集

## 端口扫描

首先使用 nmap 去探测存活端口。

[![](assets/1706146273-1e37104088705b510dd2f291a447652f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123102952-49cddb94-b997-1.png)

接着去探测端口的具体信息

[![](assets/1706146273-21b1dbe168573f68c93206d9486fa05b.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103002-4fe48712-b997-1.png)  
[![](assets/1706146273-f3c76578e72145630b0a045906ddf1cb.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103013-567e7754-b997-1.png)  
访问 80 端口，发现一个网页。

[![](assets/1706146273-add4f7be88f4f59ead13887d5b32cf68.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103025-5dc67e9e-b997-1.png)

## 目录爆破

使用工具对其进行目录爆破

[![](assets/1706146273-ec191f7eb8ad283a459fc857ac2127ad.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103039-66052f24-b997-1.png)

## SMB 未授权访问

使用 smbmap 尝试进行未授权访问。

[![](assets/1706146273-ad48970b984e498bfc89f5bd83f78039.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103048-6b7f5cf4-b997-1.png)

## 用户暴力破解

使用在网页中获取到的用户进行枚举密码。

[![](assets/1706146273-91632edc0584ebdad6328b7d47d81a11.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103100-7247627a-b997-1.png)

## SMB 未授权访问

发现主机中有一些共享文件。

[![](assets/1706146273-a761f1e9758f30c2d91c7f0fa672d242.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103110-7894e378-b997-1.png)

在 RedirectedFolders$有一些用户信息。

[![](assets/1706146273-49936448d408844a0a510bed79eef1f4.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103120-7e11d3ce-b997-1.png)

接着进行查看目录中的文件

[![](assets/1706146273-1d4135db895cefd9cb2a7bf7328bd9c8.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103128-8335b7bc-b997-1.png)

## LDAP 未授权访问

使用 ladpsearch 来枚举 ladp 信息。

[![](assets/1706146273-17a097251f8ed744b3b608de29a9565d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103137-883184bc-b997-1.png)

经过身份验证的 LDAP 搜索验证。

[![](assets/1706146273-7669d5f3aafe85614af02a294f1c603b.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103146-8df54e88-b997-1.png)

## LDAP 域转储

使用 ldapdomaindump 来转存域用户信息。

[![](assets/1706146273-1f3641f66e7890d59cd590f69033edce.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103157-9412e974-b997-1.png)

然后它会建立 html 网页，这样查找起来更方便。

[![](assets/1706146273-680534e8f8e7ac8711a88f0e4c57d74e.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103213-9df4fd92-b997-1.png)

有一些“帮助用户”的帐户和不同的基于位置的帮助台组

[![](assets/1706146273-6c5c83b870eaf54a92cb636e2f6b6462.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103222-a36fcaf4-b997-1.png)

[![](assets/1706146273-dc8de78c1f05ff70156a5e16b52e4b82.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103231-a8b42690-b997-1.png)

发现了一个临时用户：web\_svc

[![](assets/1706146273-873604a5276c06e99e0704d869830e6f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103240-add5efbe-b997-1.png)

## Bloodhound 分析

运行 Bloodhound.py 来收集域信息。

[![](assets/1706146273-7202a3bfeed1a5136220edb8d54b99c7.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103315-c2f04f5c-b997-1.png)

查看数据，hope.sharp 无法访问任何有趣的内容。  
列出所有 Kerberoastable 帐户”查询返回两个用户：

[![](assets/1706146273-452dc981980befe6f6e23aafd690f756.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103323-c76a7440-b997-1.png)

# Edgar.Jacobs

## Kerbero

使用 GetUserSPNs.py 来获取用户 hash

[![](assets/1706146273-6c8f8d5fd7eba19ab5a6bc8410b2a12c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103259-b93a41ac-b997-1.png)

接着进行 hashcat 爆破，然后使用 crackmapexec 来验证用户。

[![](assets/1706146273-33d5ae80df8521fb6a44e988b0f8810d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103345-d47e6934-b997-1.png)

## SMB 枚举

接着进行 SMB 枚举，来获取有用信息。

[![](assets/1706146273-f14f6717c5737b65b2ae56f9e2f9ef22.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103352-d90deb82-b997-1.png)

然后进行密码喷洒

[![](assets/1706146273-444e790c7fe315419621df5b4dc02c22.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103400-dd9e8f1c-b997-1.png)

# Siearra.Frye

## Bloodhound 分析/LDAP

标记 Edgar.Jacobs 拥有的 Bloodhound，然后继续进行分析。

[![](assets/1706146273-7b15d9765badcef62315aa422fbc9eb1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103409-e3394fa2-b997-1.png)

查看用户组之间的关系。

[![](assets/1706146273-468e33a154e236514a66fd3aa9739ceb.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103416-e6e9efda-b997-1.png)

[![](assets/1706146273-c48da88a1eef1627a8f6ecfd9628545c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103424-ebfe7540-b997-1.png)

## SMB 未授权

使用 smbmap 来进行文件的未授权访问。

[![](assets/1706146273-aeade4cd3e6aa872d2003314b56c2a86.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103432-f0c625be-b997-1.png)

发现它是空的。

[![](assets/1706146273-9e631e3d1f32415b74820e13b5083094.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103439-f4ca241c-b997-1.png)

发现一个.xlsx 文件，发现大量用户和密码。

[![](assets/1706146273-8c88e398723df14419f16423a43e8553.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103449-fafe66cc-b997-1.png)

然后打开发现没有完整的密码，有解压密码。

[![](assets/1706146273-62cd033bb60347c81ae0bc0762bedb77.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103457-ffb2cec4-b997-1.png)

密码 01082020 选项卡有 14 行，分别是第一个、最后一个和用户名：

[![](assets/1706146273-face8f805c05847bfdae24315a6b235f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103504-0410e6a4-b998-1.png)

然后解压看看。

[![](assets/1706146273-03500ce98a2c989eedc71b2a567e192f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103512-08790a5a-b998-1.png)

接着进行用户名和密码爆破。然后获取到可用用户，进行登录。

[![](assets/1706146273-8a06a4d94b4a7a6b530850aaaef53b35.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103519-0cdb0c60-b998-1.png)

然后来检验用户的可用性。

[![](assets/1706146273-29f4bfabc218adbf793aa3d6ee6dab38.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103526-10b34118-b998-1.png)

## 获取 user.txt

成功获取 user.txt

[![](assets/1706146273-f18847cf5e7552780a6e9b3ea1f2496d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103534-157006aa-b998-1.png)

## 导入证书

[![](assets/1706146273-bd1f630e92a4dd25da3bf017c13a5274.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103543-1b1121f2-b998-1.png)

发现登录需要密码。

[![](assets/1706146273-e1974b71cee77c0aeb1e658163db92df.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103550-1f6f5af2-b998-1.png)

## 破解密码

使用 pfx2john 来破解证书的密码。

[![](assets/1706146273-df506ad798913495f422191534b977d7.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103605-283a1b22-b998-1.png)

[![](assets/1706146273-5aa3b0e60d27222fd40173a2f789d9a1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103614-2d73bc56-b998-1.png)

## 获取 shell

访问 https 页面，成功获取一个 shell。

[![](assets/1706146273-d1ace6decf2de5dda42172c36ecdd941.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103620-314ba91a-b998-1.png)

# Tristan.Davies

回到 Bloodhound，我会将 Sierra.Fry 标记为已拥有

[![](assets/1706146273-b4b472ad2e4502bc1269e80a7fcbff60.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103628-35cd93ae-b998-1.png)

## 获取密码

组托管服务帐户 (GMSA) 是 Windows 服务器通过为帐户生成长随机密码来管理帐户密码的地方  
将其保存在其保存在一个变量。

[![](assets/1706146273-50323c13d4f71d486b9acb696cd9d207.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103644-3f9b3404-b998-1.png)

## 重置用户密码

用上面的帐户密码创建的对象 Invoke-Command 作为 BIR-ADFS-GSMA$ 运行：PSCredential

[![](assets/1706146273-62da6e486ec40e621fa1eaec307266f7.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103652-44417518-b998-1.png)

[![](assets/1706146273-83d9483d235d1f80e1e92f8f7ab4fa76.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103659-481dc63c-b998-1.png)

[![](assets/1706146273-e457983638b615a6db24b43f53062fe3.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103707-4cdaeeac-b998-1.png)

使用 crackmapexec 来验证密码是否重置。

[![](assets/1706146273-81dba03f0542d3d6c1b5df93695d9bf3.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103716-52772894-b998-1.png)

## 获取 root.txt

成功获取 root.txt  
接着使用 wmiexec.py 进行登录。

[![](assets/1706146273-1e2488fe4b696e0802eba65789bcf729.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240123103726-587f3b50-b998-1.png)
