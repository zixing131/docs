
# Debian 11 nftables 防火墙的简单配置

** 技术漫步 ** 2023-06-10 PM ** 53℃ ** 0 条

**1、检测状态：**`systemctl status nftables`

**2、设置自启动：**`systemctl enable nftables`

**3、列出规则：**`nft list ruleset`

**4、备份文件：**`cp /etc/nftables.conf /etc/nftables.conf.bak`

**5、修改配置文件：**`vi /etc/nftables.conf`

```python
#!/usr/sbin/nft -f

flush ruleset

table inet filter { 
    chain input {
        type filter hook input priority 0;
        tcp dport { 22,80,443 } accept #放行端口
        iifname lo accept #允许本地环回

        icmp type echo-request counter drop #禁 ping ipv4
        icmpv6 type echo-request drop #禁 ping ipv6
        icmpv6 type { nd-neighbor-solicit,nd-neighbor-advert,nd-router-solicit,nd-router-advert } accept # 放行 ipv6 邻居发现等 (对于不开启 nftables 能正常 ping 通外部网络 而当开启了 nftables 则无法使用 ipv6 时 这很重要！)

        ct state related,established accept # 放行已建立的或相关的连接
        counter reject # 拒绝上述规则以外的入站请求
    }

    chain forward {
        type filter hook forward priority 0;
    }

    chain output {
        type filter hook output priority 0;
    }
}
```

**6、重启 nftables 防火墙：**`systemctl restart nftables`

**标签：[Debian 11](https://www.detachd.top/index.php/tag/Debian-11/)

非特殊说明，本博所有文章均为博主原创。

如若转载，请注明出处：[https://www.detachd.top/index.php/archives/53/](https://www.detachd.top/index.php/archives/53/)

上一篇 [Linux 安装配置 JDK](https://www.detachd.top/index.php/archives/36/ "Linux安装配置JDK")

下一篇 [PotPlayer 结合 alist 播放网盘视频](https://www.detachd.top/index.php/archives/57/ "PotPlayer结合alist播放网盘视频")

#### ** 评论啦~

  

提交评论
