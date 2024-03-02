
# [Linux - 网络](https://www.raingray.com/archives/2203.html)

## 目录

-   [目录](#%E7%9B%AE%E5%BD%95)
-   [配置文件](#%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6)
-   [常用命令](#%E5%B8%B8%E7%94%A8%E5%91%BD%E4%BB%A4)
    -   [ifcong](#ifcong)
    -   [nestat](#nestat)
    -   [route](#route)
-   [ss](#ss)
-   [ip](#ip)
    -   [ntop](#ntop)
    -   [ngrep](#ngrep)
    -   [tcpdump](#tcpdump)
-   [参考链接](#%E5%8F%82%E8%80%83%E9%93%BE%E6%8E%A5)

## 配置文件

Debian 下的网卡配置文件`/etc/network/interfaces`  
Debian 下 DNS 配置文件`/etc/resolv.conf`  
CentOS 网卡配置文件`/etc/sysconfig/network-scripts/`

## 常用命令

在 CentOS7[官方 FAQ](https://www.raingray.com/archives/495.html) 中提到已经移除 ifconfig 和 netstat 所用软件包 net-tools，改用 ip 和 ss 命令 (软件包 iproute)。

> 7\. What have you done with ifconfig/netstat? The ifconfig and netstat utilities have been marked as deprecated in the man pages for CentOS 5  
> The ifconfig and netstat utilities have been marked as deprecated in the man pages for CentOS 5 and 6 for nearly a decade and Redhat made the decision to no longer install the net-tools package by default in CentOS 7. One reason to switch is that ifconfig does not show all details of ip addresses assigned to interfaces - use the ip command instead. The replacement utilities are ss and ip. If you really really need ifconfig and netstat back then you can yum install net-tools.

尽管已经剔除，在工作中 CentOS6 还是见得到，这类工具也需要掌握最基本的使用方法。

### ifcong

ifconfig #查看接口信息

inet 是对应的 IP，netmask 为子网掩码，broadcast 是广播地址，ether 是 MAC 地址，RX packets 是收到了多少字节的包，TX packets 是发送了多少包，对应的 error 可能是丢弃或者是什么原因没传输成功。

```plaintext
[root@iZ2ze3excf14rd7l4rsgn3Z ~]# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.241.36  netmask 255.255.240.0  broadcast 172.17.255.255
        ether 00:16:3e:0c:1f:1a  txqueuelen 1000  (Ethernet)
        RX packets 70247083  bytes 9217052363 (8.5 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 64130399  bytes 11258594120 (10.4 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 6744950  bytes 11121955004 (10.3 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 6744950  bytes 11121955004 (10.3 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

禁止/启动网卡

ifup

ifdown

### nestat

netstat -pantu #查看网络连接信息

### route

route -n #查看路由表，避免解析为主机名因此使用 -n

Destination 是你要请求的目标地址，Gateway 会将 Destination 的流量通过 Gateway 转发出去。

```plaintext
[root@iZ2ze3excf14rd7l4rsgn3Z ~]# route                                      
Kernel IP routing table                                                      
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         gateway         0.0.0.0         UG    0      0        0 eth0 
link-local      0.0.0.0         255.255.0.0     U     1002   0        0 eth0 
172.17.240.0    0.0.0.0         255.255.240.0   U     0      0        0 eth0 
[root@iZ2ze3excf14rd7l4rsgn3Z ~]# route -n                                   
Kernel IP routing table                                                      
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.17.255.253  0.0.0.0         UG    0      0        0 eth0 
169.254.0.0     0.0.0.0         255.255.0.0     U     1002   0        0 eth0 
172.17.240.0    0.0.0.0         255.255.240.0   U     0      0        0 eth0 
[root@iZ2ze3excf14rd7l4rsgn3Z ~]#
```

## ss

ss -lntp

\-t 查看所有 tcp 连接  
\-l 插件监听端口  
\-n 禁止 IP 解析  
\-p 查看对应进程  
\-a 查看所有 tcp、udp 信息

## ip

使用方法

`ip [ OPTIONS ] OBJECT { COMMAND | help }`

```plaintext
[root@iZ2ze3excf14rd7l4rsgn3Z ~]# ip --help
Usage: ip [ OPTIONS ] OBJECT { COMMAND | help }
       ip [ -force ] -batch filename
where  OBJECT := { link | address | addrlabel | route | rule | neigh | ntable |
                   tunnel | tuntap | maddress | mroute | mrule | monitor | xfrm |
                   netns | l2tp | fou | macsec | tcp_metrics | token | netconf | ila |
                   vrf }
       OPTIONS := { -V[ersion] | -s[tatistics] | -d[etails] | -r[esolve] |
                    -h[uman-readable] | -iec |
                    -f[amily] { inet | inet6 | ipx | dnet | mpls | bridge | link } |
                    -4 | -6 | -I | -D | -B | -0 |
                    -l[oops] { maximum-addr-flush-attempts } | -br[ief] |
                    -o[neline] | -t[imestamp] | -ts[hort] | -b[atch] [filename] |
                    -rc[vbuf] [size] | -n[etns] name | -a[ll] | -c[olor]}
```

其中 OBJECT 里面的内容是有子功能

像是 `ip address` 执行后只显示简单网卡内容。

```plaintext
[root@iZ2ze3excf14rd7l4rsgn3Z ~]# ip address
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:16:3e:0c:1f:1a brd ff:ff:ff:ff:ff:ff
    inet 172.17.241.36/20 brd 172.17.255.255 scope global dynamic eth0
       valid_lft 284021179sec preferred_lft 284021179sec
```

但在 man 中 `SEE ALSO` 章节还有许多介绍，其中 `ip-address` 还会有许多使用方法。

```plaintext
SEE ALSO
       ip-address(8), ip-addrlabel(8), ip-l2tp(8), ip-link(8), ip-maddress(8), ip-monitor(8), ip-mroute(8), ip-neighbour(8), ip-netns(8), ip-ntable(8), ip-route(8), ip-rule(8),
       ip-tcp_metrics(8), ip-token(8), ip-tunnel(8), ip-xfrm(8)
       IP Command reference ip-cref.ps
```

最简单就是使用 help 命令查看帮忙

```plaintext
[root@iZ2ze3excf14rd7l4rsgn3Z ~]# ip address help
Usage: ip address {add|change|replace} IFADDR dev IFNAME [ LIFETIME ]
                                                      [ CONFFLAG-LIST ]
       ip address del IFADDR dev IFNAME [mngtmpaddr]
       ip address {save|flush} [ dev IFNAME ] [ scope SCOPE-ID ]
                            [ to PREFIX ] [ FLAG-LIST ] [ label LABEL ] [up]
       ip address [ show [ dev IFNAME ] [ scope SCOPE-ID ] [ master DEVICE ]
                         [ type TYPE ] [ to PREFIX ] [ FLAG-LIST ]
                         [ label LABEL ] [up] [ vrf NAME ] ]
       ip address {showdump|restore}
IFADDR := PREFIX | ADDR peer PREFIX
          [ broadcast ADDR ] [ anycast ADDR ]
          [ label IFNAME ] [ scope SCOPE-ID ]
SCOPE-ID := [ host | link | global | NUMBER ]
FLAG-LIST := [ FLAG-LIST ] FLAG
FLAG  := [ permanent | dynamic | secondary | primary |
           [-]tentative | [-]deprecated | [-]dadfailed | temporary |
           CONFFLAG-LIST ]
CONFFLAG-LIST := [ CONFFLAG-LIST ] CONFFLAG
CONFFLAG  := [ home | nodad | mngtmpaddr | noprefixroute | autojoin ]
LIFETIME := [ valid_lft LFT ] [ preferred_lft LFT ]
LFT := forever | SECONDS
TYPE := { vlan | veth | vcan | dummy | ifb | macvlan | macvtap |
          bridge | bond | ipoib | ip6tnl | ipip | sit | vxlan | lowpan |
          gre | gretap | ip6gre | ip6gretap | vti | nlmon | can |
          bond_slave | ipvlan | geneve | bridge_slave | vrf | hsr | macsec }
```

每个命令通常有下面几种操作

add

delete

show/list

常用选项说明

\-s，stats 显示状态信息

\-h，human 讲一些数据显示为可读数据

\-d，details 显示更多信息

\-c，color 高亮显示

### ntop

可以抓取网络流量，被 DDOS 的情况下可以用这个查看。

有开启一个 Web 端来查看信息。

### ngrep

相当于一个抓取网络流量的 grep 工具

套餐

```plaintext
ngrep -W byline -d eth0 port 443
```

### tcpdump

一个命令行数据包分析工具，图形化可参见 [Wireshark 基本使用与网络协议简介](https://www.raingray.com/archives/495.html) 这篇文章。

Usages：`tcpdump [options]`

Options：

```plaintext
-i #网卡名
-s #为0不限制包内容
-w #保存为一个pcap包
-A #ASCII方式显示
-X #以十六进制方式显示
-r #读取一个pcap包
-V #读取一个文件列表
-n #不把地址转换为域名
```

Example：

```plaintext
#筛选器
src host address #只显示指定source地址
dst host address #只显示指定destination地址
[协议类型] port port_number #筛选你要的协议或端口
```

## 参考链接

最近更新：2023 年 10 月 18 日 16:17:07

发布时间：2020 年 01 月 22 日 22:21:00
