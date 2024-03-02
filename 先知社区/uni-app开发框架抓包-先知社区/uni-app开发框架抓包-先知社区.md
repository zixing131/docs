

# uni-app 开发框架抓包 - 先知社区

uni-app 开发框架抓包

- - -

# 1、前言

作为应用移动框架开发，uni-app 与 React Native 和 Flutter 等移动开发框架具有很强的市场性。出现了很多 app 都有其身影，因此借遇到的 uni-app 框架的 app 并研究其抓流量的手法。  
uni-app 是一个使用 Vue.js 开发所有前端应用的框架，开发者编写一套代码，可发布到 iOS、Android、Web（响应式）、以及各种小程序（微信/支付宝/百度/头条/飞书/QQ/快手/钉钉/淘宝）、快应用等多个平台。

# 2、uni-app 框架

如图所示  
[![](assets/1698893058-f72daf26575b1384e280eedc9bf7439d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231101130426-2135e5a0-7874-1.png)  
从图中可以找到 uni-app 大部分在开发的时候都使用了 js 去直接调用 Android 或 IOS 的 API，在运行时分成了视图层和逻辑层，业务代码的执行最终会直接调用 Android 或 IOS 的原生 API 实现。正因为该快应用框架都基于 js 去调用 api 从而实现功能，所以关注点可以在业务 js 上。  
在官网的开发文档中可以得知 uni-app 是通过 uni.request(OBJECT) 发起请求，利用 sslVerify 参数去验证 ssl 证书，实现应用对 ssl 证书的校验。  
[![](assets/1698893058-b1f3a85f3e369c2e328a15d657fd85e4.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231101131828-174881b8-7876-1.png)

# 3、手法

## 1, Hook bypass ssl pinning

利用 frida 去 hook 掉其证书的校验，对于常规针对使用证书校验的思路来说，这个方法是可行。但是如果存在其他的认证的手法，像本地的证书校验等也会导致失效，具体情况具体分析。大部分的情况下是通用的。

## 2、hook 网络请求框架

通过逆向确定网络请求框架，对网络请求框架进行分析，利用 hook 网络请求框架中的关键函数，打印请求报文和相应报文。  
像比较流行的 okhttp 网络框架，特点为  
[![](assets/1698893058-bba61e6ae51afaabe2c7babf4a84c752.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231101133554-86c2ca7e-7878-1.png)  
该框架支持异步和同步请求，无论是异步还是同步请求，都需要调用 OkHttpClient 对象的 newCall(Request) 方法，而 newCall 方法最终会调用 RealCall 的 newRealCall 方法获取到一个 RealCall 对象，异步请求对应 RealCall 的 enqueue 方法，而同步请求则对应 execute 方法，如果继续阅读 OkHttp 框架源码不难发现 enqueue 最终还是会回到 execute。可以对 RealCall 这个类的 getResponseWithInterceptorChain 方法进行 Hook，通过反射获取到 Request 对象 originalRequest，然后通过 getResult 获取到返回值，也就是 Reponse 对象，后面就是报文的打印。

# 4、案例

某个 app 用 postern 抓不到包同时存在加密情况。看到资源文件中的 www 文件夹联想到是 uni-app 的框架，直接 hook okhttp3。如下

```plain
function main(){
  Java.perform(function () {

    var RequestBody = Java.use("dc.squareup.okhttp3.RequestBody");
    RequestBody["create"].overload('dc.squareup.okhttp3.MediaType', 'java.lang.String').implementation = function (mediaType, str) {
      console.log(str);
      var ret2 = this.create(mediaType, str);
      return ret2;
    };


  });
}
setImmediate(main);
```

[![](assets/1698893058-bf9be53d469c5e9da5de8d2c762d4423.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231101133803-d3382aac-7878-1.png)  
其加密的内容可以通过 js 中去寻找答案，即  
[![](assets/1698893058-a7b1b10baf5252395ec2664f20d05e1a.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231101133947-1158d4f8-7879-1.png)  
根据 MVC 规范，对数据的加密方法一般在 app-config.js 或 app-server.js 里。

## 其他

另外对于 app 的传输过程中的加密，一是可能存在于 app-config 或 app-server 中，二是可以通过 js 调试即将其变换为 h5 页面去调试抓包，利用 usb 和 chrome 浏览器可做到调试，如图  
[![](assets/1698893058-4f4a56e7575d2aaafac18bc4b7c0652a.png)](https://xzfile.aliyuncs.com/media/upload/picture/20231101134113-44e5155c-7879-1.png)
