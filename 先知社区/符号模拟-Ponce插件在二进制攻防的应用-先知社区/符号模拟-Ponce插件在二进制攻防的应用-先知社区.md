

# 【符号模拟】Ponce 插件在二进制攻防的应用 - 先知社区

【符号模拟】Ponce 插件在二进制攻防的应用

- - -

# 1、介绍

Ponce 是一款使用 C/C++ 开发的 IDA Pro 插件，它使用户能够以简单直观的方式对二进制文件执行污点分析和符号执行。

[https://github.com/illera88/Ponce](https://github.com/illera88/Ponce)

下载 Release 里面，根据系统决定下载哪一个

[![](assets/1708919768-0bb1466b17e59ebed6ead9674e29bef0.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174421-20b66992-cd79-1.png)

安装方法也很简单：

Ponce works with both x86 and x64 binaries in any IDA version >= 7.0. Installing the plugin is as simple as copying the appropiate files from the [latest builds](https://github.com/illera88/Ponce/releases/latest) to the `plugins\` folder in your IDA installation directory.

[![](assets/1708919768-f47afb1ef32e3708f3deb09226592712.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174428-24c44752-cd79-1.png)

作为 2016 年 IDA 插件大赛获奖者，该插件应用的范围可以看到很广泛，毕竟是符号模拟插件

[![](assets/1708919768-af81aa53b7b3dabd35d251a25498f752.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174435-28be5afa-cd79-1.png)

# 2、练习

[https://github.com/illera88/Ponce/tree/master/examples](https://github.com/illera88/Ponce/tree/master/examples)

这是官方给我们提供的练习程序，我们使用 crackme\_xor.cpp

```plain
#include <stdio.h>
#include <stdlib.h>

const char *serial = "\x31\x3e\x3d\x26\x31";

int check(char *ptr)
{
    int i = 0;

    while (i < 5){
        if (((ptr[i] - 1) ^ 0x55) != serial[i])
            return 1;
        i++;
    }
    return 0;
}

int main(int ac, char **av)
{
    int ret;

    if (ac != 2)
        return -1;

    ret = check(av[1]);
    if (ret == 0)
        printf("Win\n");
    else
        printf("fail\n");

    return 0;
}
```

打开程序

[![](assets/1708919768-18a92444709872c0acaae938fac3f0e2.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174500-37fb180a-cd79-1.png)

自动弹出 Ponce 的配置页面

进入 main 函数

[![](assets/1708919768-0ce80ec20eb5eed8464d109f1ea54af1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174506-3b7f6756-cd79-1.png)

ponce 需要动态运行，所以还需要配置一下

[![](assets/1708919768-31f906827a71cf1bebe8c168dbcd7ef7.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174514-3ff7cbfc-cd79-1.png)

接下来需要在关键比较的地方下断点：

1、符号化关键比较

[![](assets/1708919768-f308f7260e96e9f8763e40597e32600b.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174519-42ec82a8-cd79-1.png)

2、符号化关键数据

[![](assets/1708919768-3338c06b139631a02351f8bf736198fa.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174524-45f68336-cd79-1.png)

动态运行，在\[ebp+input\]的地方断下来

[![](assets/1708919768-434e93bd3257e0db5449168797855cb5.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174528-48ad6c20-cd79-1.png)

符号化，也可以使用快捷键

[![](assets/1708919768-a703c6500ba2266e6ad0646c53339247.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174537-4d80faaa-cd79-1.png)

[![](assets/1708919768-018f1ce1a29f06a612e99337ccbd31cb.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174539-4ec621f6-cd79-1.png)

确定

[![](assets/1708919768-cd90124d7d1b2086e2b89a1e654a1ce7.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174546-53022314-cd79-1.png)

再次运行，此时不断闪烁的边是程序即将跳转的分支

[![](assets/1708919768-f5e5ec9c5fd31c386006baf414332ec8.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174552-56b9680a-cd79-1.png)

接下来右键关键比较的地方

[![](assets/1708919768-ec226b4b7c9ebf57cfadf159327b0fe7.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174601-5beb131e-cd79-1.png)

创建 SMT 模拟

```plain
右键断点位置-->SMT Solver--> Negate and Inject
```

[![](assets/1708919768-243dc6d61d67b254077e67914d492877.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174609-61062fbe-cd79-1.png)

发现是成功得到了结果

不断重复过程

[![](assets/1708919768-06a83998f2272758df19d575cf258839.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174614-63f1d9c6-cd79-1.png)

跳转到另一边，然后再运行

美中不足的是每次动态调试都要重新设置符号变量

[![](assets/1708919768-fffd7e43be7c2b6b06647ee7fa9718f1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174620-675b5b8c-cd79-1.png)

不过用起来真的很爽

[![](assets/1708919768-39fb0c47943c5fed9a4248b1d7e7083c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174629-6cad6d3c-cd79-1.png)

# 3, 2023N1CTF-addtion plus

[![](assets/1708919768-31d9403fa42a03af201ad74a3aa0a1ac.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174743-98a11092-cd79-1.png)

废话不多说，IDA 打开分析即可

[![](assets/1708919768-95997e5ed5be0468a0c281da8cb26439.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174750-9cda58d0-cd79-1.png)

生成的假 flag

```plain
111111111111111111111111111111111111111111111111
```

这是动态调用加密函数，call 一个地址

[![](assets/1708919768-0a7648b3207afb5330c45119727f7785.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174754-9fb728b2-cd79-1.png)

该函数极大，单纯分析的话，真的费时间。

预期解为使用 z3 模式求解

[![](assets/1708919768-c7d9d189637f79fa8a428061a1d912f3.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174800-a2bd0a86-cd79-1.png)

会调用 0x30 次，说明对输入的每一个 flag 进行加密

[![](assets/1708919768-db8e06ceaa7b08f37555e8eeccbcf475.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174804-a57e61c0-cd79-1.png)

继续分析，找到关键比较的地方

[![](assets/1708919768-f5b9c9726154d4ec0d45219ecd4f319f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174811-a98db356-cd79-1.png)

可以看到是每 16 字节进行比较，正好分为了 3 部分

如果同时爆这 48 位，很难爆出，我们缩减思路，每 8 位进行爆破，因为加密的时候就是每 8 个字节进行加密的（8 个字节为一组，依次调用不同的加密的函数）

[![](assets/1708919768-cac2e805717a5067aefe64a666434010.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174815-abe36f60-cd79-1.png)

对第一块比较进行分析

[![](assets/1708919768-ddcd604e7b4dca1b58a2bd4cf7e4cac9.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174821-af978aa6-cd79-1.png)

8 字节爆破，需要修改一些东西

[![](assets/1708919768-cb6a843a7363d580a05c6b26c59fef64.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174841-bbc2293a-cd79-1.png)

修改完成后就达到了 8 字节爆破的目的

[![](assets/1708919768-679fb574f2007c741178db0af4c32b3d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174848-bfe83e78-cd79-1.png)

先符号化前 8 位

[![](assets/1708919768-766ad26371d1cc81c047af8ea47434eb.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174855-c4192bc4-cd79-1.png)

[![](assets/1708919768-76c1259fed0e5d4b5ef33da822fd334d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174903-c8dc9bfa-cd79-1.png)

[![](assets/1708919768-c776b66ef6bb50dac771a3758a7efa16.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174907-cab2ac9e-cd79-1.png)

需要进行简单 patch，这里需要注意的是远程 Linux 拷贝的文件，需要进行删除，然后重新 copy！

[![](assets/1708919768-5b8102ae555a92b960b34fac94dbee6b.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174912-cdd59d6e-cd79-1.png)

之后的效果如下：

[![](assets/1708919768-39813e3852ff7439d48df26e537ed034.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174922-d3b09de2-cd79-1.png)

等待大概 20 分钟，虽然很方便，但是很费时间

期间弹出的窗口还是点击确认

[![](assets/1708919768-4256d4eb1cb272cf9b0148bc2182cde1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174927-d70e80ee-cd79-1.png)

[![](assets/1708919768-89af2310b073b7fcce6f7a74cbee9214.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240217174931-d96ffb06-cd79-1.png)

前 8 位爆破要了半小时

后面比较中间 17-32 的时候，需要将加密的起始位置修改为 17 开始，相同的操作，就是很费时间。

# 总结：

使用 ponce，我们需要标记（符号化）

1、关键数据地址

2、关键比较

然后就静等片刻咯
