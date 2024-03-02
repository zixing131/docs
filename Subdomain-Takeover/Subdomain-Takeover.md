
# [Subdomain Takeover](https://www.raingray.com/archives/4368.html)

我发现最早的资料是 2014 年《[Hostile Subdomain Takeover using Heroku/Github/Desk + more](https://labs.detectify.com/2014/10/21/hostile-subdomain-takeover-using-herokugithubdesk-more/)》一文提到子域接管（Subdomain Takeover）。原理是正常应用 CNAME 所指向的域名过期，被攻击者注册，攻击者通过控制过期域名来接管应用。

## 目录

-   [目录](#%E7%9B%AE%E5%BD%95)
-   [CNAME](#CNAME)
-   [NS](#NS)
-   [MX](#MX)
-   [参考资料](#%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99)

## CNAME

自己指定 CNAME 场景很常见，在爆破域名时如果发现 CNAME 对应的域名过期可注册，就很有可能存在子域名接管。

这里发现 test.raingray.com 指向 vulwiki.com，并没有发现被注册，可能是忘了取消 CNAME。

```plaintext
[root@blog ~]# dig -t A +nostats +noquestion test.raingray.com

; <<>> DiG 9.11.26-RedHat-9.11.26-6.el8 <<>> -t A +nostats +noquestion test.raingray.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 63174
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; ANSWER SECTION:
test.raingray.com.  120 IN  CNAME   vulwiki.com.
```

此时就可以注册 vulwiki.com 域名。当你访问 test.raingray.com 会自动最终请求到 vulwiki.com 资源，浏览器地址栏不会有变化。总结成一句话，你不用我用。

那能造成什么危害？

1.  窃取 Cookie
    
    当 Coookie Domain 属性设置为顶级域时，请求所有子域名也会带上此 Coookie。攻击者可以记住 Referer 和 Cookie 来判断用户从哪里跳转过来，利用 Coookie 凭证冒充用户，操作功能。
    
2.  钓鱼站点
    
    由于访问时无感知，可以创建钓鱼页面，利用域名可信度来钓取凭证。
    
3.  点击劫持
    
    X-Frame-Options 限制不严格
    
    ```plaintext
     X-Frame-Options: SAMEORIGIN
    ```
    
4.  OAuth CSRF 授权码劫持
    
    如果接管的域名在白名单内，可以尝试将请求转发到接管域名上。
    

知道原理后可以建立 Check SOP 来自动化过程：

1.  检查 CNAME 记录
    
2.  域名过期能注册
    

尝试自动化。子域接管是种 low hanging fruit 漏洞，因此发现后要立马提交报告，手慢无。

```plaintext
dig -t CNAME +short vulwiki.com | wc -l
```

```plaintext
httpx -l subdomains.txt -cname cnames.txt
```

**第三方服务商**

大部分服务提供商比如云存储、虚拟主机创建完后，大部分会为你生成一个域名，子域名是你可以自定义的，有可能是博客名称、存储桶名称，只是顶级域是服务商的。

这里以 CSGO 官网图片云存储举例。在实际中很多企业都用 OSS 存储，在创建完 Bucket 会自动以 Bucket Name 名生成子域名。这里 img.csgo.com.cn 是自定义的 Bucket 名称，wsglb0.com 是网宿顶级域名。他们会把自己域名 CNAME 指向 Bucket 域名上方便管理。

```plaintext
[root@blog ~]# dig -t CNAME +nostats +noquestion +nocmd +noopcode img.csgo.com.cn
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 42982
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; ANSWER SECTION:
img.csgo.com.cn.    600 IN  CNAME   img.csgo.com.cn.wsglb0.com.
```

一旦 Bucket 删除，对应子域名 img.csgo.com.cn.wsglb0.com 无法访问，原有域名 img.csgo.com.cn 的 CNAME 记录忘了删除，那么我们可以创建名为 img.csgo.com.cn 对应存储桶，最终访问 img.csgo.com.cn.wsglb0.com 就是取攻击者存储桶资源，等同于接管了此 Bucket。

有些应用会存在另一种情况，比如 test.example.com 域名 CNAME 所指向域名第三方服务商域名 test.example.com.raingray.com，访问页面返回 404 或者提示不存在，此时通过创建名为 test.example.com 的服务还不能接管，需在应用上主动设置绑定域名 test.example.com 才可成功利用。

这种情况通常有安全意识的应用研发，绑定 test.example.com 域名时，会让你在解析记录中添加 TXT 记录，做验证确保域名是你的，这种情况下无法利用。

具体哪些应用可以利用 [can-i-take-over-xyz](https://github.com/EdOverflow/can-i-take-over-xyz) 项目给出了参考表格。

> | Engine | Status | Verified by CI/CD | Domains | Fingerprint | Discussion | Documentation |
> | --- | --- | --- | --- | --- | --- | --- |
> | AWS/Elastic Beanstalk | Vulnerable | 🟩  | us-east-1.elasticbeanstalk.com | `NXDOMAIN` | [Issue #194](https://github.com/EdOverflow/can-i-take-over-xyz/issues/194) |     |
> | AWS/Load Balancer (ELB) | Not vulnerable | 🟥  | elb.amazonaws.com | `NXDOMAIN` | [Issue #137](https://github.com/EdOverflow/can-i-take-over-xyz/issues/137) |     |
> | AWS/S3 | Vulnerable | 🟩  | s3.amazonaws.com | `The specified bucket does not exist` | [Issue #36](https://github.com/EdOverflow/can-i-take-over-xyz/issues/36) |     |
> | ...... |     |     |     |     |     |     |

注意这是参考表格，但不要依赖给出的清单，就判定一定有或没有漏洞。如 FreshService 应用，[@naglinagli](https://twitter.com/naglinagli/status/1377381633846767621) 在 2021 年明明有提交绕过报告，他们也没及时更正过。

**自动化工具检测**

说了半天 CNAME 和第三方服务商子域名接管缺陷，那该如何自动检测？

第三方服务商检查目前只有 [subzy](https://github.com/LukaSikic/subzy) 项目，使用 can-i-take-over-xyz 的 fingerprints.json 指纹和其结果来判定是否存在缺陷。这清单统计的都是国外应用，对它们相对友好，可是国内里面的应用服务提供商却一个都没，如果真的要检测还需要适配国内应用指纹。目前就国内环境整体收益不高，不是很有兴趣去做，吃力不讨好。

其次就是 [dnsReaper](https://github.com/punk-security/dnsReaper) 看起来相对好用，能够自动检查 AWS / Azure / CloudFlare 几个常见国际云服务提供商，还包括了 CNAME 配置错误。

```plaintext
┌──(raingray㉿raingray)-[~/桌面/dnsReaper-1.8.1]
└─$ python main.py  single --domain test.raingray.com --out subdomain_takeover_result.txt --out-format csv 
          ____              __   _____                      _ __       
         / __ \__  ______  / /__/ ___/___  _______  _______(_) /___  __                                                                    
        / /_/ / / / / __ \/ //_/\__ \/ _ \/ ___/ / / / ___/ / __/ / / /                                                                    
       / ____/ /_/ / / / / ,<  ___/ /  __/ /__/ /_/ / /  / / /_/ /_/ /                                                                     
      /_/    \__,_/_/ /_/_/|_|/____/\___/\___/\__,_/_/  /_/\__/\__, /                                                                      
                                             PRESENTS         /____/                                                                       
                              DNS Reaper ☠                                                                                                 

             Scan all your DNS records for subdomain takeovers!                                                                            

Domain 'test.raingray.com' provided on commandline
Testing with 60 signatures


We found 2 takeovers ☠
-- DOMAIN 'test.raingray.com' :: SIGNATURE '_generic_cname_found_but_unregistered' :: CONFIDENCE 'CONFIRMED'
CNAME: vusal223a.com                                                                                                                       
-- DOMAIN 'test.raingray.com' :: SIGNATURE '_generic_cname_found_doesnt_resolve' :: CONFIDENCE 'POTENTIAL'
CNAME: vusal223a.com                                                                                                                       

⏱  We completed in 1.07 seconds
...Thats all folks!
```

```json
[
    {
        "domain": "test.raingray.com",
        "signature": "_generic_cname_found_but_unregistered",
        "info": " The defined domain has a CNAME record configured but the CNAME is not registered. You should look to see if you can register this CNAME.",
        "confidence": "CONFIRMED",
        "a_records": [],
        "aaaa_records": [],
        "cname_records": [
            "vusal223a.com"
        ],
        "ns_records": [],
        "more_info_url": ""
    },
    {
        "domain": "test.raingray.com",
        "signature": "_generic_cname_found_doesnt_resolve",
        "info": " The defined domain has a CNAME record configured but the CNAME does not resolve. You should look to see if you can register or takeover this CNAME.",
        "confidence": "POTENTIAL",
        "a_records": [],
        "aaaa_records": [],
        "cname_records": [
            "vusal223a.com"
        ],
        "ns_records": [],
        "more_info_url": ""
    }
]
```

大家还很推崇的 [subjack](https://github.com/haccer/subjack)，也是很常见的一款工具，但没用过。

## NS

CNAME 案例待看文章：

-   [Hacktivity - Search results for "Subdomain Takeover".](https://hackerone.com/hacktivity?querystring=Subdomain%20Takeover)
-   [✍️ Subdomain Takeover in Azure: making a PoC](https://godiego.tech/posts/STO/)
-   [✍️ All \*.intercom.help subdomains vulnerable to Subdomain Takeover from intercom Service](https://www.mohamedharon.com/2020/06/all-intercomhelp-subdomains-vulnerable.html)
-   [✍️ How i bought my way to subdomain takeover on Tokopedia](https://medium.com/bugbountywriteup/how-i-bought-my-way-to-subdomain-takeover-on-tokopedia-8c6697c85b4d)
-   [✍️ Subdomain takeover via pantheon](https://smaranchand.com.np/2019/12/subdomain-takeover-via-pantheon/)
-   [✍️ Subdomain Takeover via Campaignmonitor.com](https://www.mohamedharon.com/2019/11/subdomain-takeover-via.html)
-   [✍️ Escalating subdomain takeovers to steal cookies by abusing document.domain](https://blog.takemyhand.xyz/2019/05/escalating-subdomain-takeovers-to-steal.html)
-   [✍️Subdomain takeover Awarded $200](https://medium.com/@friendly_/subdomain-takeover-awarded-200-8296f4abe1b0)
-   [✍️ Subdomain Takeover via Wufoo Service in a Private Program](https://www.mohamedharon.com/2019/02/subdomain-takeover-via-wufoo-service-in.html)
-   [✍️ Subdomain Takeover via HubSpot](https://www.mohamedharon.com/2019/02/subdomain-takeover-via-hubspot.html)
-   [✍️ Souq.com Subdomain Takeover via jazzhr.com service](https://www.mohamedharon.com/2019/02/souqcom-subdomain-takeover-via.html)
-   [✍️ Subdomain Takeover — New Level](https://medium.com/bugbountywriteup/subdomain-takeover-new-level-43f88b55e0b2)
-   [✍️ Subdomain takeover dew to missconfigured project settings for Custom domain .](https://medium.com/@prial261/subdomain-takeover-dew-to-missconfigured-project-settings-for-custom-domain-46e90e702969)
-   [✍️ Subdomain Takeover via Shopify Vendor ( blog.exchangemarketplace.com ) with Steps](https://www.mohamedharon.com/2018/10/subdomain-takeover-via-shopify-vendor.html)
-   [✍️ Subdomain Takeover via Unsecured S3 Bucket Connected to the Website](https://blog.securitybreached.org/2018/09/24/subdomain-takeover-via-unsecured-s3-bucket/)
-   [✍️ Subdomain Takeover worth 200$](https://medium.com/@alirazzaq/subdomain-takeover-worth-200-ed73f0a58ffe)
-   [✍️ Subdomain Takeover via Campaignmonitor](https://www.mohamedharon.com/2018/09/subdomain-takeover-via-campaignmonitor.html)
-   [✍️ How to do 55.000+ Subdomain Takeover in a Blink of an Eye](https://medium.com/@thebuckhacker/how-to-do-55-000-subdomain-takeover-in-a-blink-of-an-eye-a94954c3fc75)
-   [✍️ Subdomain Takeover: Yet another Starbucks case](https://0xpatrik.com/subdomain-takeover-starbucks-ii/)
-   [✍️ Shipt Subdomain TakeOver via HeroKu ( test.shipt.com )](https://www.mohamedharon.com/2018/08/Shipttakeover.html)
-   [✍️ Subdomain Takeover: Starbucks points to Azure](https://0xpatrik.com/subdomain-takeover-starbucks/)
-   [✍️ UBER Wildcard Subdomain Takeover | BugBounty POC](https://blog.securitybreached.org/2017/11/20/uber-wildcard-subdomain-takeover)
-   [✍️ Bugcrowd’s Domain & Subdomain Takeover vulnerability!](https://blog.securitybreached.org/2017/10/10/bugcrowds-domain-subdomain-takeover-vulnerability)
-   [✍️ Subdomain Takeover Through Expired Cloudfront Distribution](https://blog.securitybreached.org/2017/10/10/subdomain-takeover-lamborghini-hacked/)
-   [✍️ Authentication bypass on Uber’s Single Sign-On via subdomain takeover](https://www.arneswinnen.net/2017/06/authentication-bypass-on-ubers-sso-via-subdomain-takeover/)
-   [✍️ Authentication bypass on Ubiquity’s Single Sign-On via subdomain takeover](https://www.arneswinnen.net/2016/11/authentication-bypass-on-sso-ubnt-com-via-subdomain-takeover-of-ping-ubnt-com/)

NS 接管工具：

-   [https://github.com/pwnesia/dnstake](https://github.com/pwnesia/dnstake)
-   [https://github.com/indianajson/can-i-take-over-dns](https://github.com/indianajson/can-i-take-over-dns)

NS 或者 MX 接管文章：

-   [https://thehackerblog.com/the-international-incident-gaining-control-of-a-int-domain-name-with-dns-trickery/index.html](https://thehackerblog.com/the-international-incident-gaining-control-of-a-int-domain-name-with-dns-trickery/index.html)
-   [https://thehackerblog.com/the-orphaned-internet-taking-over-120k-domains-via-a-dns-vulnerability-in-aws-google-cloud-rackspace-and-digital-ocean/](https://thehackerblog.com/the-orphaned-internet-taking-over-120k-domains-via-a-dns-vulnerability-in-aws-google-cloud-rackspace-and-digital-ocean/)
-   [https://thehackerblog.com/respect-my-authority-hijacking-broken-nameservers-to-compromise-your-target/](https://thehackerblog.com/respect-my-authority-hijacking-broken-nameservers-to-compromise-your-target/)
-   [https://thehackerblog.com/the-international-incident-gaining-control-of-a-int-domain-name-with-dns-trickery/](https://thehackerblog.com/the-international-incident-gaining-control-of-a-int-domain-name-with-dns-trickery/)
-   [https://thehackerblog.com/hacking-guatemalas-dns-spying-on-active-directory-users-by-exploiting-a-tld-misconfiguration/](https://thehackerblog.com/hacking-guatemalas-dns-spying-on-active-directory-users-by-exploiting-a-tld-misconfiguration/)
-   [https://thehackerblog.com/the-journey-to-hijacking-a-countrys-tld-the-hidden-risks-of-domain-extensions/](https://thehackerblog.com/the-journey-to-hijacking-a-countrys-tld-the-hidden-risks-of-domain-extensions/)

## MX

MX 接管工具：

-   [https://github.com/musana/mx-takeover#attack-scenario-for-mailgun](https://github.com/musana/mx-takeover#attack-scenario-for-mailgun)

MX 利用点：Spoof mails, If SPF record whitelists this subdomain.

## 参考资料

-   [A Guide To Subdomain Takeovers](https://www.hackerone.com/application-security/guide-subdomain-takeovers)
-   [Test for Subdomain Takeover](https://github.com/OWASP/wstg/blob/master/document/4-Web_Application_Security_Testing/02-Configuration_and_Deployment_Management_Testing/10-Test_for_Subdomain_Takeover.md)
-   [ubdomain Takeover: Basics](https://0xpatrik.com/subdomain-takeover-basics/)
-   [Subdomain Takeover: Going beyond CNAME](https://0xpatrik.com/subdomain-takeover-ns/)
-   [Subdomain Takeover: Proof Creation for Bug Bounties](https://0xpatrik.com/takeover-proofs/)
-   [Subdomain Takeover: Finding Candidates](https://0xpatrik.com/subdomain-takeover-candidates/)
-   [Subdomain Takeover: Yet another Starbucks case](https://0xpatrik.com/subdomain-takeover-starbucks-ii/)
-   [十三、子域劫持](https://wizardforcel.gitbooks.io/web-hacking-101/content/13.html)
-   [Hacktivity - Search results for "Subdomain Takeover".](https://hackerone.com/hacktivity?querystring=Subdomain%20Takeover)
-   [What I learnt from reading 217\* Subdomain Takeover bug reports.](https://medium.com/@nynan/what-i-learnt-from-reading-217-subdomain-takeover-bug-reports-c0b94eda4366)

发布时间：2023 年 03 月 28 日 08:51:00
