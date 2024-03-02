

# 【深蓝实验室】红队信息收集&移动安全从 0-1 - FreeBuf 网络安全行业门户

# 红队信息收集&移动安全从 0-1

## 企业信息

```plain
天眼查、企查查、企业信用信息公示系统、企业组织架构
企业邮箱收集，企业架构画像、人员统计、人员职责、部门、WiFi、常用部门密码、人员是否泄露过密码、人员平时爱逛的站点、OA/erp/crm/sso/mail/等入口、网络安全设备（waf,ips,ids,router 等统计）、内部使用的代码托管平台 (gitlab、daocloud 等)，bug 管理平台、服务器域名资产统计
注册公司、基金会、校友会、出版社、校医院、
site:xxx 直属单位/site:xxx 机构设置
```

## 空间搜索引擎

-   FOFA https://fofa.so (已关闭)
    
-   Quake https://quake.360.cn/quake/#/index
    
-   Hunter https://hunter.qianxin.com
    
-   Censys https://search.censys.io/
    
-   Shadon https://www.shodan.io
    
-   ZoomEye https://www.zoomeye.org
    
-   Sumap https://sumap.dbappsecurity.com.cn/
    
-   Soall https://soall.org/login
    

常用语法

```plain
title="目标中文名" && country=CN 
title="目标中文名" && region="xx 省" 
title="目标中文名" && city="xx 市" 
header="目标中文名" && country=CN 
header="目标中文名" && region="xx 省" 
header="目标中文名" && city="xx 市" 
domain="目标域名" 
host="目标域名" 
cert="目标域名或者证书关键字" && country=CN cert="目标域名或者证书关键字" && region="xx 省" cert="目标域名或者证书关键字" && city="xx 市" ="目标中文名或者目标域名
cert="China Merchants Bank" or cert="xx 银行" or cert="xxx.com" or cert="xxx"
icon==""
#Google
site:xxx.com "shen 份证号" 
site:xxx.com "shen 份证号" 
filetype:doc site:xxx.com "shen 份证号" 
filetype:xls site:xxx.com "工号" 
site:xxx.com inurl:login 
site:xxx.com inurl:upload

#Github
"xxx.com" "shen 份证号" 
"xxx.com" "username" 
"xxx.com" "password"
```

## Whois 信息

站点注册人注册过的其他网站 (对注册人、邮箱、电话的反查)，对查到的站点的深入

-   站长之家 http://whois.chinaz.com
    
-   Bugscanner http://whois.bugscaner.com
    
-   国外 BGP https://bgp.he.net
    
-   who.is https://who.is/
    

## 一级域名

-   企查查 https://www.qichacha.com
    
-   天眼查 https://www.tianyancha.com
    
-   爱企查 https://aiqicha.baidu.com
    

## 子域名

```plain
老站、同样架构或同源码的子站
爆破，接口查询
  https://phpinfo.me/domain/
  https://d.chinacycc.com/index.php?m=Login&a=index
  subDomainBrute、knockpy
OWA 发现、dig adfs、dig mail
https://dns.bufferover.run/dns?q=baidu.com
http://api.hackertarget.com/reversedns/?q=target.com
```

-   Amass https://github.com/OWASP/Amass
    
-   OneForAll https://github.com/shmilylty/OneForAll
    
-   ksubdomain https://github.com/knownsec/ksubdomain
    
-   subDomainsBrute https://github.com/lijiejie/subDomainsBrute
    
-   Sonar https://omnisint.io/
    
-   查子域 https://chaziyu.com/ (在线)
    
-   笨米 https://www.benmi.com/ 挂了
    

## HOST 碰撞

```plain
https://github.com/cckuailong/hostscan
https://github.com/fofapro/Hosts_scan
```

很多时候，访问目标网站时，某些内网是无法在外网通过 IP 进行访问的 但是这时候有一种 Host 碰撞方法 他是收集外网的域名和 IP 然后在本机 把外网的 IP 和内网的域名进行绑定这是因为 nginx 配置了禁止直接 IP 访问。

## 同站

-   火狐插件 Flagfox
    

## 旁站

-   在线 http://stool.chinaz.com/same
    
-   在线 https://site.ip138.com
    

## B、C 段信息

```plain
Banner、是否存在目标的后台或其他入口/其他业务系统
ASN  BGP 路由协议
1-64511 公有 AS
64512-65535 私有 AS
https://www.cidr-report.org/cgi-bin/as-report?as=china&view=2.0
```

## BypassCDN

```plain
Ping
多地 Ping
国外 Ping
查找老域名
查找关联域名
信息泄露/配置文件
Phpinfo
网页源码
Svn
Github
Shodan/fofa/zoomeye/hunter/360quake/censys
SSL 证书记录
		https://crt.sh/
		https://developers.facebook.com/tools/ct
		https://ui.ctsearch.entrust.com/ui/ctsearchui
设置 xff/x-remote-ip/x-remote-addr 为 127.0.0.1/或 ipv6 地址
RSS 订阅/邮件头
APP 反编译搜索/截取 APP 的请求信息
修改 hosts 文件指向
```

### DNS 历史解析

-   DNS db https://dnsdb.io/zh-cn/
    
-   dns 检测 https://tools.ipip.net/dns.php
    
-   Xcdn https://github.com/3xp10it/xcdn
    
-   在线 https://ipchaxun.com
    
-   DNSdumpster https://dnsdumpster.com/
    

## IP 定位

```plain
https://www.opengps.cn/Default.aspx
https://www.chaipip.com/aiwen.html
https://cz88.net/
```

## 网站架构/服务器指纹/CMS 识别/容器

```plain
网页源代码
请求头/响应头
网站底部，顶部，左上角右上角
网站报错信息
域名/install
CMS 漏洞
  定位版本对应已知漏洞检查
  CMS 未知漏洞挖掘
Web 容器已知漏洞 (解析漏洞这种)
中间件、组件
Weblogic、tomcat、zabbix、struts、axis 等
```

-   WhatWeb https://github.com/urbanadventurer/WhatWeb
    
-   Builtwith https://builtwith.com/zh/
    
-   FortyNorthSecurity https://github.com/FortyNorthSecurity/EyeWitness
    
-   云悉 https://www.yunsee.cn/
    
-   Wappalyzer https://www.wappalyzer.com/?utm\_source=popup&utm\_medium=extension&utm\_campaign=wappalyzer
    
-   EHole https://github.com/EdgeSecurityTeam/EHole
    
-   TideFinger https://github.com/TideSec/TideFinger
    
-   ObserverWard https://github.com/0x727/ObserverWard
    
-   ShuiZe https://github.com/0x727/ShuiZe\_0x727
    
-   AlliN https://github.com/P1-Team/AlliN
    
-   Bufferfly https://github.com/dr0op/bufferfly（初步处理资产小工具）
    

## 目录扫描

-   Dirmap https://github.com/H4ckForJob/dirmap
    
-   dirsearch https://github.com/maurosoria/dirsearch
    

## JS

-   JSFinder https://github.com/Threezh1/JSFinder
    
-   LinkFinder https://github.com/GerbenJavado/LinkFinder
    
-   Packer-Fuzzer https://github.com/rtcatc/Packer-Fuzzer (webpack)
    
-   搜索关键接口
    

```plain
1. config/api
2. method:"get"
3. http.get("
4. method:"post"
5. http.post("
6. $.ajax
7. service.httppost
8. service.httpget
```

## URL 提取

```plain
http://www.bulkdachecker.com/url-extractor/
```

## WAF 识别

-   wafw00f https://github.com/EnableSecurity/wafw00f
    

## 蜜罐识别

```plain
https://honeyscore.shodan.io/
hunter 也有
https://github.com/cnrstar/anti-honeypot    浏览器插件（误报还是挺多的，虚拟机 + 代理）
```

## 随手测试

```plain
单引号
admin/123456
admin/admin
万能密码
```

## 信息泄露

```plain
电话、邮箱，姓名
目录遍历
网盘文件
		https://www.chaonengsou.com/
备份文件
 (www.zip,xx.com.zip,www.xx.com.zip,wwwroot.zip)
.svn/.git/sql/robots/crossdomin.xml/DS_Store
论坛 ID=1 通常为管理员
网页上客服的 QQ(先判断是企业的还是个人，用处有时不太大，看怎么用，搞个鱼叉什么的)
```

## 弱密码

```plain
https://default-password.info/
https://www.routerpasswords.com/ 
国内的百度&日常积累
```

## 网页缓存

```plain
http://www.cachedpages.com/ 
https://web.archive.org/
```

## 图片反查

```plain
百度识图、googleimage、tineye
原图查询坐标
```

## 社交

```plain
QQ、weibo、支付宝、脉脉、知乎、领英、咸鱼、短视频、人人、贴吧、论坛、推特、ins、脸书等
手机加入通讯录匹配各个 APP 用户信息，用户名之类的方便做字典
```

## 常用注册

```plain
Sms
  https://www.materialtools.com/
  http://receivefreesms.com/
Email
  https://10minutemail.net/
  http://24mail.chacuo.net/
  https://zh.mytrashmailer.com/
  http://24mail.chacuo.net/enus
  https://www.linshiyouxiang.net/
Fake id
  https://www.fakenamegenerator.com/
  http://www.haoweichi.com/
  https://www.fakeaddressgenerator.com/
```

## 邮箱收集

```plain
https://hunter.io/
http://www.veryvp.com/
https://www.email-format.com/
https://www.yingyanso.cn/
https://verifyemailaddress.com/   邮箱有效确认
theHarvester
企查查
github
脉脉
库
snov.io 插件
```

## 历史资料

```plain
库
https://haveibeenpwned.com/
```

## Github/Gitee/码云

```plain
https://github.com/dxa4481/truffleHog
https://github.com/lijiejie/GitHack
https://github.com/MiSecurity/x-patrol
https://github.com/az0ne/Github_Nuggests
```

## 历史漏洞

```plain
http://zone-h.org/archive
乌云镜像：https://wooyun.x10sec.org
Seebug: https://www.seebug.org
Exploit Database: https://www.exploit-db.com
Vulners: https://vulners.com
Sploitus: https://sploitus.com
```

## APP

-   小蓝本 https://www.xiaolanben.com/pc
    
-   七麦 https://www.qimai.cn
    
-   AppStore https://www.apple.com/app-store
    
-   点点 https://www.diandian.com/
    

```plain
url、js、osskey、api 等信息查找
搜集到接口进行 FUZZ
https://github.com/TheKingOfDuck/ApkAnalyser
https://github.com/kelvinBen/AppInfoScanner
```

### 分析脱壳

-   幸运破解器（慎用）https://nalankang.lanzouo.com/b00u06nfa
    
-   核心破解（慎用）
    
-   BlackDex https://nalankang.lanzoui.com/b00um84kf
    
-   fdex2 https://nalankang.lanzoui.com/iBdENqkpmng
    
-   微脱壳 https://nalankang.lanzoui.com/iFfamqkpmoh
    
-   反射大师 https://nalankang.lanzoui.com/b00v2j5ud
    
-   APK Editor https://nalankang.lanzoui.com/b00u06q0d
    
-   Apktool https://nalankang.lanzoui.com/b00uud6kf
    
-   NP 管理器 https://nalankang.lanzoui.com/b00u7565e
    
-   Android Killer
    
-   以上的下载密码：ojbk
    

### 绕过限制抓包

-   HttpCanary(小黄鸟) https://nalankang.lanzoui.com/b00usn91c（可不ROOT）
    
-   Wicap https://nalankang.lanzoui.com/b00usn8te（安卓抓包用处不是很大）
    
-   Packet Capture https://nalankang.lanzoui.com/b00usn8yj（免 ROOT 抓包）
    
-   JustTrustMe++ https://github.com/JunGe-Y/JustTrustMePP（目前测试效果最好的）
    
-   Fildder + BP（主流）会有 https 的问题
    
-   Mitmproxy 配合 py 脚本做流量处理，或者单纯的脱一下 https
    
-   Charles（使用比较少）
    

## 公众号

```plain
https://weixin.sogou.com
微信搜索
```

## 小程序

```plain
https://www.xiaolanben.com/pc
微信直接搜索
小程序的更多信息
```

## 综合利用工具

-   Nuclei https://github.com/projectdiscovery/nuclei
    
-   Spiderfoot https://github.com/smicallef/spiderfoot
