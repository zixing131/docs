

# AsyncRAT 加解密技术剖析 - 先知社区

AsyncRAT 加解密技术剖析

- - -

## 概述

最近，笔者在浏览网络中威胁情报信息的时候，发现美国 Alien Labs 实验室于 2024 年 1 月 5 日发布了一篇《AsyncRAT loader: Obfuscation, DGAs, decoys and Govno》报告，报告中讲到美国关键基础设施在 2023 年期间遭受了大量 AsyncRAT 木马攻击，针对此类攻击事件，Alien Labs 实验室在报告中对其进行了对比及相同点介绍，同时还对 AsyncRAT 木马的部分隐蔽技术进行了剖析。

基于此，于是笔者对 Alien Labs 实验室报告中提到的 AsyncRAT 木马进行了简单的研究，通过网络调研，笔者发现 AsyncRAT 是 2019 年发布的一款开源远控工具，目前仍可在 github（[https://github.com/NYAN-x-CAT/AsyncRAT-C-Sharp](https://github.com/NYAN-x-CAT/AsyncRAT-C-Sharp) ）中下载使用，同时，笔者还发现网络中存在不少 AsyncRAT 木马的攻击案例，不少 APT 组织均会使用此款远控木马作为最终控制程序。

由于 AsyncRAT 是一款开源远控工具，因此，所有人均可下载编译，并对其开源代码进行升级优化，定制化生成自有的远控工具。基于此，笔者认为，AsyncRAT 木马未来一定会以各式各样的形态出现在各类网络攻击及 APT 攻击事件中，因此，如果能够提前对 AsyncRAT 木马的各类技术进行详细的剖析，其一定会在日后的网络攻击对抗中发挥作用。

针对此思路，笔者准备编写系列文章从不同角度对 AsyncRAT 远控工具进行详细剖析，届时欢迎各位大佬关注并指点。

系列文章大致分三篇，分别为：

-   AsyncRAT 加解密技术剖析：针对 AsyncRAT 木马使用的各类加解密技术进行详细剖析，例如：解密配置信息、解密通信数据等；同时还会输出针对 AsyncRAT 配置信息解密的自动化脚本。
-   AsyncRAT 通信模型剖析及自动化解密脚本实现：针对 AsyncRAT 木马的通信模型进行详细对比剖析，从字节角度对通信数据结构进行深层次的理解，输出针对 AsyncRAT 网络通信数据解密的自动化脚本，便于快速批量的对 AsyncRAT 木马通信数据包进行解密及分析。
-   AsyncRAT 攻防技术对抗：从受控端角度对 AsyncRAT 木马使用的相关技术进行剖析，并提出临检取证的技术方案，便于 AsyncRAT 木马的检测发现分析。

网络中 AsyncRAT 木马利用案例如下：

[![](assets/1706771148-0b7dd58df6e2c74b7dcfa33c3b947b54.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240131085205-f4242e22-bfd2-1.png)

[![](assets/1706771148-9fce1f154941f26a3fb9fec936cc2af6.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240131085218-fc268a20-bfd2-1.png)

[![](assets/1706771148-7db01587b39238fccb1a005e94823186.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240131085233-04c1b7c2-bfd3-1.png)

## 开源 AsyncRAT 利用分析

由于 AsyncRAT 是开源远控工具，因此直接访问 github 即可对其工具源码、release 版本程序进行下载研究。

通过对开源项目进行简单分析，发现：

-   AsyncRAT 远控工具于 2019 年 9 月 15 日首次发布，目前共推出 8 次迭代，最新版本为 2023 年 10 月 17 日发布的 AsyncRAT v0.5.8 版本；
-   AsyncRAT 开源项目的维护人员较少，目前只有三人维护，2000 余个标星，756 个账号 forks 拷贝项目；
-   AsyncRAT远控工具的所有模块均为C#代码编写，比较核心的模块为Client端、Server端、Plugin插件模块；
-   AsyncRAT 远控工具的远控功能较全面，常见的远控功能均支持；

相关截图如下：

[![](assets/1706771148-4e66c68beed4a49614914f1c6a363b73.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240131085246-0ca264aa-bfd3-1.png)

### 生成 AsyncRATClient 端木马

下载 release 版本程序并直接运行即可打开 AsyncRAT 控制端的 GUI 界面，AsyncRAT 控制端运行后将在当前目录中生成 ServerCertificate.p12 文件，此文件中存放了 AsyncRAT 安全加密通信的密钥和证书；

在 GUI 界面中选择【BUILDER】菜单即可对 AsyncRATClient 端木马进行自定义配置，相关截图如下：

[![](assets/1706771148-4a9e046ade79c7980438678830674388.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240131085301-15bd7494-bfd3-1.png)

### 木马上线

在受控主机中运行 AsyncRATClient 端木马程序，即可成功实现木马上线，上线后即可实现对受控主机的远控管理，相关截图如下：

[![](assets/1706771148-9db61b9171889cf420a125ae371780db.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240131085314-1d83726e-bfd3-1.png)

## 配置信息解密

由于 AsyncRATClient 端木马程序可根据攻击者意图进行自定义配置，因此，笔者准备以此作为切入点开启对 AsyncRAT 木马的分析。

通过分析，笔者发现，在 Client 命令空间的 Settings 类中，存在大量的加密配置信息数据，经过分析，发现此数据即为生成 AsyncRATClient 端木马环节中的自定义数据，相关截图如下：

[![](assets/1706771148-0ea28a54a163df86c1cd8449c225046f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240131085330-2713da8a-bfd3-1.png)

尝试对上述配置信息进行解密尝试，发现可成功对其进行解密：

-   配置信息相关字符串最外层均采用 base64 进行编码；
-   配置信息的最终载荷使用 aes 进行加密；
-   配置信息中的 Key 数据将用于 PBKDF2 算法的输入，通过迭代计算生成两个加密密钥，一个用于 aes 运算，密钥长度 32 字节，一个用于 HMACSHA256 哈希值计算，密钥长度 64 字节；
-   base64 解码后的配置信息字符串将分为三段使用：前 32 字节将 HMACSHA256 校验，中间 16 字节将用于 aes 运算的 IV 值，最后字节数据即为实际最终加密载荷数据；

解密尝试如下：

```plain
1.key 处理，生成 aes key 及 HMACSHA256 key
aes key : 09f892078304d5723d9a841fac16c59674f84cc830fc4f98e892efc0c5d29f3d
HMACSHA256 key: 371e838b11793bc4c8ad13eae04adc61da8e8e553827388ef2ab1dc9d5daaf63ac6808abfecb80b84746c9091176a57bac3ead5756688ace0bd048447e74ea1f

//源数据
public static string Ports = "cOErtB+zERnjbUteYOnvETnEOYqfBvJapeKqKq9EuRCgBS8Ra/wmUUaV71BIyFAtxgc1rYoQ/SLMQOZ+0be38g==";

2.base64 解码
70e12bb41fb31119e36d4b5e60e9ef1139c4398a9f06f25aa5e2aa2aaf44b910a0052f116bfc26514695ef5048c8502dc60735ad8a10fd22cc40e67ed1b7b7f2

3.拆分
70e12bb41fb31119e36d4b5e60e9ef1139c4398a9f06f25aa5e2aa2aaf44b910    //HMACSHA256 校验值
a0052f116bfc26514695ef5048c8502d    //AES IV 值
c60735ad8a10fd22cc40e67ed1b7b7f2    //最终加密载荷

4.AES 解密
6606,7707,8808
```

相关截图如下：

[![](assets/1706771148-2f141b5c6c9094cf7c6be6f3383d3fc0.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240131085356-3635fc1e-bfd3-1.png)

### 自动化解密脚本

由于 AsyncRAT 木马中的加密配置字符串较多，如果手动或动态调试进行解密势必比较麻烦，因此，笔者基于加解密算法原理，使用 Go 语言编写了一个自动化解密脚本，供大家实现快速批量解密，解密效果如下：

```plain
C:\Users\admin\GolandProjects\awesomeProject10>awesomeProject10.exe
Ports:6606,7707,8808
Hosts:192.168.153.131
Version:0.5.8
Install:false
InstallFolder：%AppData%
InstallFile：
MTX:VxPUZhab0vai
Certificate:MIIE8jCCAtqgAwIBAgIQAOrgUXSLtVlPwb+xjHXpTzANBgkqhkiG9w0BAQ0FADAaMRgwFgYDVQQDDA9Bc3luY1JBVCBTZXJ2ZXIwIBcNMjQwMTIzMDIwNDMwWhgPOTk5OTEyMzEyMzU5NTlaMBoxGDAWBgNVBAMMD0FzeW5jUkFUIFNlcnZlcjCCAiIwDQYJKoZIhvcNAQEBBQADggIPADCCAgoCggIBAIHfdsDvlXy5S+HA3bxkFlIQ9W0oIvyooNn5ur1zyu2c7ldg2+NUN3ScMJTJRbsAOZdJ648pAiE1Gw7roGQEIbC1tJzFT/6pGDL42dM6Ej18PbBFqWfMhAKLk2M4OG1Y87iMvXE0NUeriqo1VWL9fhQrHtdEjv7NrHvmdZtkPW7ue0uKLzxWLKlp5X8CQjxmm4t73zD4+rSOHxmmM45siD67bQ15o1jl4sOYRs6hPED0tZvtTSmnBQG/4vcu/dJo6vqxbpZ7kBOmIM2EEI9zcXZvVLTZnuU9pyS3tZe2f9C4/QmL0eAzxMnNROqIys1QOt2PrvTCCNLPlJWMeC8kRnUaVuncK9Ucad/3CvOIVUyXKsCVU4vuVYwjxg0WPgU30NiNtaTz6auolH+ABmpLvGqJ00truGW9lnvdFuFVl9jStzoggwlqhhj1lwvKql+V0nmyHBLxZT85Ealo3+GcwrrU0LCoJDMMMLNWjqiYr+FXd6BR3JtfTlIBe9oVAn6iHStLL9uRubeSDo32+lWYgG9cm1seGcLjbl8k5WWFQAxsljKdwC5AkkKmeyFyVUdyyZ1abgbWs6REkVuv86Ant3z4ElwqNn4ZV/g3svRmT3Gh3tZcTPUz2u84IK4ta9j4FK+n/xjPk0rdL0APs44W5nDouSQhEB+OIb351Ks9Se/zAgMBAAGjMjAwMB0GA1UdDgQWBBRAmU6yKUmiYLyoRMisNdJh+KwnfzAPBgNVHRMBAf8EBTADAQH/MA0GCSqGSIb3DQEBDQUAA4ICAQBVAQ5JIvN5GKdRKyID49kE2uO2UCsGKTpZXucdFsZ8wbrtVuM2h6+8xerLJ1T2ccUT2mdHG2Xqmy8U3Uh6UcEaqJzqljRfQ/OWZIWGYQ4P43yLIoSdbkFil0H/OoWtkpxidNLHkUZ3mSpCusb6iwO3LKX8tNtZk0Gfrn5mJQBhhVFAHsElIuephI2tsgVeuTOyRKYCCq2Vl9VusgrY0NkGwKxEqlSK5YCw2hlEbEGQ2wkIwrVMWOAZEMa7XS0S/kYVjtk8FStOX+RqDd6gy+RjHCnE/VJAFdXPiNmlgyu8AmFJXdwGLU2Oj/ho3jqPqT9WX46o7i21XroJHgFbnV859yW+tTXoRMIZxC4JgFJau2tpxugM/P8fXrhTbAGQwN1xscbvqDPBAhc1TEtPyG8j+/1xCIxBGzb5ILDiwnDOzZdU7eSqdaxye/6OUa9PYxGNqN/0m8koYjUqI5HLkAv2+kA5r+cUYetUtxJZ1pIphT5HbNYvBi07oD3M+3DRVAtRCplXtCEUnL+Y90eFJ8gECB7L8gE03boS25hsRBpVfc5pNUURp8yNZmf9BtNEMiGdZROYRs3/F+YhAU84gKfMylvj2TEjMR/S1bcFrpH5qqPGvjHFEkKMPcaK/IF/RfR/iBwXaEG4VIFGBCsKZUo4Fm6MRCmmoIekVy2Uv5qiXQ==
Serversignature:EY1lMQqmxNj55Y0mK5J1h/0OjB+4U6/kRTEc4/IG8mGADCsR9cad7G3ulycZy07csTrolQTSlg05e9YWbODG0+5XznZdUfweBgBWU4YKtq77+4o4lbQtIG1fDKnBGG8ifYyDZ2aC+VEA9mm+yF/jSB0BIDp6s3h02Xrf3zu6bTXzTtEtlEsk8V8ZGwFtl4EuioRbQLTrRN8yFuQWd7lL9MlwXij13kNc1u7fohfWSMWwKbHkeYSpQZQ07HLCt0qgRcZ0uJ1T8kQtKetIHSa+XUycNhs6iuHw4Tb8ZORSPBTDyfkvsC9ceBFTV54BVsRwqUiF3uOi7I3kodRMYGJNncn12bz3sUxckY4cw+i2yZ9MR4tmmZkmkELOUNF01MLQVBmfbkORk82iS4oi3tQDU1zTyHgVO4aB3TaO6dePmf7obQs2cT2UuDcBOZWfAgU8ya37+WFfMogVUaInRadRXS2Lwx4X7fhBNLIif9NM3Q3RHTi7B2NoiROnx8kEtpfjqMVG0YJyvvGP/bMya/NsIjhb2XhfgJcQx21IuoxBnRYkg0YcS4pJC3r2J8HcOY7nuY7Z7Xt54Vc4rabzekadI1eUGz1/lcp57Bu8M/ai4n35dl9CD9okH9ZFy3iB5k715KhQU3tWQg0JZ4xNg8BUZSN3/0ciWpH8oReaBJYYoSQ=
Anti:false
Pastebin:null
BDOS:false
Delay：3
Group:Default

C:\Users\admin\GolandProjects\awesomeProject10>
```

相关截图如下：

[![](assets/1706771148-75735ce90ede16b2ba32f918f0bcc195.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240131085421-453d4050-bfd3-1.png)

自动化脚本输入文件格式如下：**（备注：为提升解密便利性，笔者在程序中对配置信息字符串进行了处理，因此只需将 dnspy 工具中反编译后的配置信息代码片段复制进输入文件中即可实现自动化脚本解密）**

[![](assets/1706771148-fd038439c74afcc6de4c930b943fca28.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240131085438-4f3d5c8e-bfd3-1.png)

### 代码实现

代码结构如下：

[![](assets/1706771148-75a8e7350d1255ceb7cbb843569193c8.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240131085455-59b0804c-bfd3-1.png)

-   main.go

```plain
package main

import (
    "awesomeProject10/common"
    "encoding/hex"
    "fmt"
    "strings"
)

func main() {
    key := ""

    strs := common.FileToSlice("C:\\Users\\admin\\Desktop\\1.txt")
    for _, str := range strs {
        if strings.Contains(str, "public static string Key = ") {
            key = strings.Split(strings.Split(str, ` = "`)[1], `";`)[0]
        }
    }
    for _, str := range strs {
        if strings.Contains(str, "public static string Ports = ") {
            Ports := strings.Split(strings.Split(str, ` = "`)[1], `";`)[0]
            fmt.Print("Ports:")
            decrypt_asyncRAT_str(key, Ports)
        } else if strings.Contains(str, "public static string Hosts = ") {
            Hosts := strings.Split(strings.Split(str, ` = "`)[1], `";`)[0]
            fmt.Print("Hosts:")
            decrypt_asyncRAT_str(key, Hosts)
        } else if strings.Contains(str, "public static string Version = ") {
            Version := strings.Split(strings.Split(str, ` = "`)[1], `";`)[0]
            fmt.Print("Version:")
            decrypt_asyncRAT_str(key, Version)
        } else if strings.Contains(str, "public static string Install = ") {
            Install := strings.Split(strings.Split(str, ` = "`)[1], `";`)[0]
            fmt.Print("Install:")
            decrypt_asyncRAT_str(key, Install)
        } else if strings.Contains(str, "public static string InstallFolder = ") {
            InstallFolder := strings.Split(strings.Split(str, ` = "`)[1], `";`)[0]
            fmt.Println("InstallFolder：" + InstallFolder)
        } else if strings.Contains(str, "public static string InstallFile = ") {
            InstallFile := strings.Split(strings.Split(str, ` = "`)[1], `";`)[0]
            fmt.Println("InstallFile：" + InstallFile)
        } else if strings.Contains(str, "public static string MTX = ") {
            MTX := strings.Split(strings.Split(str, ` = "`)[1], `";`)[0]
            fmt.Print("MTX:")
            decrypt_asyncRAT_str(key, MTX)
        } else if strings.Contains(str, "public static string Certificate = ") {
            Certificate := strings.Split(strings.Split(str, ` = "`)[1], `";`)[0]
            fmt.Print("Certificate:")
            decrypt_asyncRAT_str(key, Certificate)
        } else if strings.Contains(str, "public static string Serversignature = ") {
            Serversignature := strings.Split(strings.Split(str, ` = "`)[1], `";`)[0]
            fmt.Print("Serversignature:")
            decrypt_asyncRAT_str(key, Serversignature)
        } else if strings.Contains(str, "public static string Anti = ") {
            Anti := strings.Split(strings.Split(str, ` = "`)[1], `";`)[0]
            fmt.Print("Anti:")
            decrypt_asyncRAT_str(key, Anti)
        } else if strings.Contains(str, "public static string Pastebin = ") {
            Pastebin := strings.Split(strings.Split(str, ` = "`)[1], `";`)[0]
            fmt.Print("Pastebin:")
            decrypt_asyncRAT_str(key, Pastebin)
        } else if strings.Contains(str, "public static string BDOS = ") {
            BDOS := strings.Split(strings.Split(str, ` = "`)[1], `";`)[0]
            fmt.Print("BDOS:")
            decrypt_asyncRAT_str(key, BDOS)
        } else if strings.Contains(str, "public static string Delay = ") {
            Delay := strings.Split(strings.Split(str, ` = "`)[1], `";`)[0]
            fmt.Println("Delay：" + Delay)
        } else if strings.Contains(str, "public static string Group = ") {
            Group := strings.Split(strings.Split(str, ` = "`)[1], `";`)[0]
            fmt.Print("Group:")
            decrypt_asyncRAT_str(key, Group)
        }
    }
}

func decrypt_asyncRAT_str(key string, input_str string) {
    input := common.Base64_Decode(input_str)
    hmacsha := input[:32]
    aes_iv := input[32:48]
    encode_data := input[48:]

    rfc2898DeriveBytes := common.Pbkdf2_Rfc2898DeriveBytes(common.Base64_Decode(key))
    aeskey := rfc2898DeriveBytes[:32]
    _authKey := rfc2898DeriveBytes[32:]

    if hex.EncodeToString(common.HMACSHA256(_authKey, input[32:])) != hex.EncodeToString(hmacsha) {
        fmt.Println("Error")
        return
    }
    output, _ := common.Aes_decrypt_cbc(encode_data, aeskey, aes_iv)
    fmt.Println(string(output))
}
```

-   common/common.go

```plain
package common

import (
    "bufio"
    "crypto/aes"
    "crypto/cipher"
    "crypto/hmac"
    "crypto/sha1"
    "crypto/sha256"
    "encoding/base64"
    "encoding/hex"
    "fmt"
    "golang.org/x/crypto/pbkdf2"
    "os"
)

func FileToSlice(file string) []string {
    fil, _ := os.Open(file)
    defer fil.Close()
    var lines []string
    scanner := bufio.NewScanner(fil)
    for scanner.Scan() {
        lines = append(lines, scanner.Text())
    }
    return lines
}

func Base64_Decode(encodedMessage string) []byte {
    decodedMessage, err := base64.StdEncoding.DecodeString(encodedMessage)
    if err != nil {
        fmt.Println("Base64_Decode Error:", err)
        return nil
    }
    return decodedMessage
}

func Pbkdf2_Rfc2898DeriveBytes(masterKey []byte) []byte {
    //样本中提取
    salt, _ := hex.DecodeString("BFEB1E56FBCD973BB219022430A57843003D5644D21E62B9D4F180E7E6C33941")
    iterations := 50000
    keyLength := 96 // 密钥长度为字节单位
    derivedKey := pbkdf2.Key(masterKey, salt, iterations, keyLength, sha1.New)
    return derivedKey
}

func HMACSHA256(key, message []byte) []byte {
    h := hmac.New(sha256.New, key)
    h.Write(message)
    // 计算消息的 HMACSHA256 哈希值
    sha := h.Sum(nil)
    return sha
}

func Aes_decrypt_cbc(data []byte, key []byte, iv []byte) ([]byte, error) {
    data_new := []byte{}
    data_new = append(data_new, iv...)
    data_new = append(data_new, data...)

    block, err := aes.NewCipher(key)
    if err != nil {
        return nil, err
    }

    if len(data_new) < aes.BlockSize {
        return nil, fmt.Errorf("加密数据太短")
    }

    iv = data_new[:aes.BlockSize]
    data_new = data_new[aes.BlockSize:]

    if len(data_new)%aes.BlockSize != 0 {
        return nil, fmt.Errorf("加密数据长度不是块大小的整数倍")
    }

    mode := cipher.NewCBCDecrypter(block, iv)
    mode.CryptBlocks(data_new, data_new)

    data_new = pkcs5UnPadding(data_new)

    return data_new, nil
}

func pkcs5UnPadding(ciphertext []byte) []byte {
    length := len(ciphertext)
    unpadding := int(ciphertext[length-1])
    return ciphertext[:(length - unpadding)]
}
```

## 通信数据解密

对 AsyncRAT 木马上线过程及远程控制过程进行流量抓取分析，发现其通信数据主要分为两层：

-   第一层加密：调用 TLS1.0 对通信数据进行加密；
-   第二层加密：调用 gzip 对通信载荷及传输模块进行加密；

### TLS 解密

通过分析，发现 AsyncRAT 木马最外层通信为 TLS，使用的密钥套件为 TLS\_RSA\_WITH\_AES\_128\_CBC\_SHA，相关截图如下：

[![](assets/1706771148-ae3e3fd1cde0db956a1681ac56f4498f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240131085525-6b646768-bfd3-1.png)

结合笔者上一篇《Remcos RAT 通信模型剖析及攻防技术对抗》文章中【通信解密尝试】章节中的 TLS 各版本的加解密方法，发现使用私钥或 CLIENT\_RANDOM 形式的 Master-Secret 即可实现对此款 AsyncRAT 木马的 TLS 通信数据的解密。

### TLS 解密 - 私钥提取

由于在开源 AsyncRAT 利用分析阶段，我们发现 ServerCertificate.p12 文件中存放了 AsyncRAT 木马安全加密通信的密钥和证书，因此我们即可尝试对 ServerCertificate.p12 文件进行密钥提取。

```plain
P12 文件介绍：P12 文件是一种常见的数字证书文件格式，通常用于存储与安全相关的密钥和证书。这种文件通常包含了私钥、公钥和证书链等信息，常见于安全通信和加密技术中，比如 SSL/TLS 加密通信、代码签名等领域。P12 文件通常需要密码来保护其中的私钥信息，以确保安全性。
```

为有效提取 ServerCertificate.p12 文件的内置保护密码，笔者对 AsyncRAT 源码进行了分析，最终发现 AsyncRAT 工具并未对 ServerCertificate.p12 文件设置保护密码，相关代码截图如下：

[![](assets/1706771148-98cc9e124769f54db47c7b17efff494c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240131085544-7683f50a-bfd3-1.png)

因此，直接使用空密码即可实现从 ServerCertificate.p12 文件中提取私钥，相关操作指令如下：

```plain
//从 P12 文件中提取证书并将其输出为 PEM 格式，-nodes：导出私钥
root@kali:~/Desktop# openssl pkcs12 -in ServerCertificate.p12 -out certificate.pem -nodes
Enter Import Password:
MAC verified OK
root@kali:~/Desktop# cat certificate.pem 
Bag Attributes
    localKeyID: 01 00 00 00 
    friendlyName: BouncyCastle-3d4bc3cc-2470-436d-ac6c-f21b33016162
    Microsoft CSP Name: Microsoft Enhanced RSA and AES Cryptographic Provider
Key Attributes
    X509v3 Key Usage: 10 
-----BEGIN PRIVATE KEY-----
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
ktFif6PSZD7QCth8XD+eRSYpmDJEbHA=
-----END PRIVATE KEY-----
Bag Attributes
    localKeyID: 01 00 00 00 
subject=/CN=AsyncRAT Server
issuer=/CN=AsyncRAT Server
-----BEGIN CERTIFICATE-----
MIIE8jCCAtqgAwIBAgIQAOrgUXSLtVlPwb+xjHXpTzANBgkqhkiG9w0BAQ0FADAa
MRgwFgYDVQQDDA9Bc3luY1JBVCBTZXJ2ZXIwIBcNMjQwMTIzMDIwNDMwWhgPOTk5
OTEyMzEyMzU5NTlaMBoxGDAWBgNVBAMMD0FzeW5jUkFUIFNlcnZlcjCCAiIwDQYJ
KoZIhvcNAQEBBQADggIPADCCAgoCggIBAIHfdsDvlXy5S+HA3bxkFlIQ9W0oIvyo
oNn5ur1zyu2c7ldg2+NUN3ScMJTJRbsAOZdJ648pAiE1Gw7roGQEIbC1tJzFT/6p
GDL42dM6Ej18PbBFqWfMhAKLk2M4OG1Y87iMvXE0NUeriqo1VWL9fhQrHtdEjv7N
rHvmdZtkPW7ue0uKLzxWLKlp5X8CQjxmm4t73zD4+rSOHxmmM45siD67bQ15o1jl
4sOYRs6hPED0tZvtTSmnBQG/4vcu/dJo6vqxbpZ7kBOmIM2EEI9zcXZvVLTZnuU9
pyS3tZe2f9C4/QmL0eAzxMnNROqIys1QOt2PrvTCCNLPlJWMeC8kRnUaVuncK9Uc
ad/3CvOIVUyXKsCVU4vuVYwjxg0WPgU30NiNtaTz6auolH+ABmpLvGqJ00truGW9
lnvdFuFVl9jStzoggwlqhhj1lwvKql+V0nmyHBLxZT85Ealo3+GcwrrU0LCoJDMM
MLNWjqiYr+FXd6BR3JtfTlIBe9oVAn6iHStLL9uRubeSDo32+lWYgG9cm1seGcLj
bl8k5WWFQAxsljKdwC5AkkKmeyFyVUdyyZ1abgbWs6REkVuv86Ant3z4ElwqNn4Z
V/g3svRmT3Gh3tZcTPUz2u84IK4ta9j4FK+n/xjPk0rdL0APs44W5nDouSQhEB+O
Ib351Ks9Se/zAgMBAAGjMjAwMB0GA1UdDgQWBBRAmU6yKUmiYLyoRMisNdJh+Kwn
fzAPBgNVHRMBAf8EBTADAQH/MA0GCSqGSIb3DQEBDQUAA4ICAQBVAQ5JIvN5GKdR
KyID49kE2uO2UCsGKTpZXucdFsZ8wbrtVuM2h6+8xerLJ1T2ccUT2mdHG2Xqmy8U
3Uh6UcEaqJzqljRfQ/OWZIWGYQ4P43yLIoSdbkFil0H/OoWtkpxidNLHkUZ3mSpC
usb6iwO3LKX8tNtZk0Gfrn5mJQBhhVFAHsElIuephI2tsgVeuTOyRKYCCq2Vl9Vu
sgrY0NkGwKxEqlSK5YCw2hlEbEGQ2wkIwrVMWOAZEMa7XS0S/kYVjtk8FStOX+Rq
Dd6gy+RjHCnE/VJAFdXPiNmlgyu8AmFJXdwGLU2Oj/ho3jqPqT9WX46o7i21XroJ
HgFbnV859yW+tTXoRMIZxC4JgFJau2tpxugM/P8fXrhTbAGQwN1xscbvqDPBAhc1
TEtPyG8j+/1xCIxBGzb5ILDiwnDOzZdU7eSqdaxye/6OUa9PYxGNqN/0m8koYjUq
I5HLkAv2+kA5r+cUYetUtxJZ1pIphT5HbNYvBi07oD3M+3DRVAtRCplXtCEUnL+Y
90eFJ8gECB7L8gE03boS25hsRBpVfc5pNUURp8yNZmf9BtNEMiGdZROYRs3/F+Yh
AU84gKfMylvj2TEjMR/S1bcFrpH5qqPGvjHFEkKMPcaK/IF/RfR/iBwXaEG4VIFG
BCsKZUo4Fm6MRCmmoIekVy2Uv5qiXQ==
-----END CERTIFICATE-----
root@kali:~/Desktop#
```

### TLS 解密 - 通信解密

直接将 certificate.pem 文件配置于 wireshark 中，或者将 certificate.pem 文件中私钥文件内容另存并配置于 wireshark 中，均可实现对 TLS 通信数据的解密，相关截图如下：

[![](assets/1706771148-c0a3e00eb030926fe5eb84fe27a71d30.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240131085610-85dea0ae-bfd3-1.png)

[![](assets/1706771148-9cede5234a69a6530d4ef378c84bb09d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240131085631-9267c5da-bfd3-1.png)

[![](assets/1706771148-7ea6fd6c1a7ce253b3b7b8a521077a93.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240131085651-9ea477e4-bfd3-1.png)

### 通信载荷解密

为进一步推进 TLS 解密后的通信数据解密，笔者对 TLS 解密后的通信数据进行了二进制对比分析，发现 TLS 解密后的通信数据中存在大量 1f8b08 二进制数据（gzip 二进制头），相关截图如下：

[![](assets/1706771148-2767ba2e24399f4bbf7a2a0fac73ceef.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240131085706-a7738d1a-bfd3-1.png)

提取 gzip 压缩数据并对其解压，发现可成功解压提取通信数据，相关截图如下：

[![](assets/1706771148-be2e5efef897dd7fa373c4b103914f31.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240131085721-b030f35c-bfd3-1.png)
