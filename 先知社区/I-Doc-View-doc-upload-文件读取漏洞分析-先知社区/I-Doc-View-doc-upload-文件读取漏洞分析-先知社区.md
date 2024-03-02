

# I Doc View /doc/upload 文件读取漏洞分析 - 先知社区

I Doc View /doc/upload 文件读取漏洞分析

- - -

# I Doc View /doc/upload分析

## 鉴权绕过

直接访问/doc/upload，会提示：请先登录，或通过 token 来访问

[![](assets/1706959674-c5abf3057c96a3fb4c99984291218c96.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202164951-07132d9a-c1a8-1.png)  
传入 token 参数，会提示：不存在该应用

[![](assets/1706959674-0aab43cc0665836c62ca5cd7ef585506.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165006-0ff49ef8-c1a8-1.png)  
定位代码，可以发现调用了 this.appService.getByToken(token) 来获取 appPo，其不存在就会提示：不存在该应用

[![](assets/1706959674-65ffe5818a5c56bb9d0a8a23988c8ba6.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165019-17b09994-c1a8-1.png)  
查看 this.appService.getByToken 方法，其调用了 this.appDao.getByToken(token) 方法

[![](assets/1706959674-f176178d1fb7a685232b7c0804c3cd9b.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165029-1da6af3c-c1a8-1.png)  
appDao 有 2 个实现类，AppDaoMySQLImpl 和 AppDaoImpl

[![](assets/1706959674-2290ed6f26e4061b6b62879794cb6bca.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165042-25581964-c1a8-1.png)  
它们的 getByToken 方法都是从 app 表中获取根据 token 值进行查询，一个是 mysql、一个是 mongodb

[![](assets/1706959674-724cc1e1d9dc2b402af79e349a474e82.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165053-2c75f806-c1a8-1.png)

[![](assets/1706959674-90df3fc202fbaf1dde7fec0371408656.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165101-31073e16-c1a8-1.png)  
在数据库初始化文件中，可以看到 app 表中存在 token 值为 testtoken 的数据

[![](assets/1706959674-c4b18701ec86a233ab81a197ea778cd4.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165125-3f27f6c0-c1a8-1.png)

[![](assets/1706959674-09bef17d0b145ac189a254f8f074393b.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165133-43c80bac-c1a8-1.png)  
传入？token=testtoken，满足条件了

[![](assets/1706959674-95929565438ed68a7ccbcbb05bd71d3f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165143-4a1c6412-c1a8-1.png)

## 文件上传？

继续往下看，接受文件上传请求，会调用 this.docService.add 方法

[![](assets/1706959674-b267e39a79bfdac0bf7235b893b28a5c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165202-5549f192-c1a8-1.png)  
在 this.docService.add 方法中会调用 this.addDoc 方法

[![](assets/1706959674-5d033c2c58178722861f7124f17fe6a9.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165214-5ca8bce8-c1a8-1.png)  
this.addDoc 方法中会调用 FileUtils.writeByteArrayToFile 方法写入文件

[![](assets/1706959674-fb919549a6a6934eaebe6704935fb795.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165228-649121e8-c1a8-1.png)

[![](assets/1706959674-9a0602da8288871c239b7ef5635620ab.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165236-697b79ba-c1a8-1.png)  
文件的保存路径是 RcUtil.getPath(rid) 生成的，rid 前面有生成

[![](assets/1706959674-3ee375511c5a66f07041bf08124a48b7.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165247-7064adf0-c1a8-1.png)

[![](assets/1706959674-31068dee33f3c60498d6b8e960a7b81a.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165301-7860c142-c1a8-1.png)  
getPath 返回内容

[![](assets/1706959674-58a51809103bb6f5ea0e4861c2b80f19.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165316-8166b7ce-c1a8-1.png)  
getDirectoryByRid 返回内容

[![](assets/1706959674-e87d1142bd4a476c5b3da9a0a9208cfd.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165327-87d8eba4-c1a8-1.png)

[![](assets/1706959674-9d5287a39a86ed90f3a72c3b3efe67f7.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165335-8cd06c04-c1a8-1.png)  
getFileNameByRid 返回内容

[![](assets/1706959674-d72f6e2f93ebac7d7b1ae561957905cb.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165346-93629010-c1a8-1.png)  
上传请求

[![](assets/1706959674-713063398a5ace76b776cc33bc6bd101.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165357-99fb9840-c1a8-1.png)

[![](assets/1706959674-5176433e8d972d28573bb0d75eccc7ce.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165404-9e25c184-c1a8-1.png)  
不过这个无法 getshell，能上传和解析 html

## 文件读取

当不是上传请求时，接受 url 参数，调用 this.docService.addUrl 方法

[![](assets/1706959674-35309f386f50b4a0e97360f48f89e864.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165424-a9b06158-c1a8-1.png)  
在 this.docService.addUrl 方法最后，在判断 data 不为空后，会调用 this.add 方法

[![](assets/1706959674-373912c069f4ce89e4156697e0bf1b7c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165435-b08e03a4-c1a8-1.png)  
同样的也会保存文件到本地，data 就是文件内容

[![](assets/1706959674-4bdbf5757fab298bbbe29556f932ddf9.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165445-b674c8d4-c1a8-1.png)  
看下 data 是怎么赋值的，如果传入的 url 匹配 (file:///?)?(\\w{1}:.+)，那么就会利用 url.replaceFirst(localPathRegex, "$2");获取其第二个捕获组的内容作为文件名，读取文件内容

[![](assets/1706959674-de6bd3ef5ebedde1d90c50baeecef07f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165459-bebedb7e-c1a8-1.png)

[![](assets/1706959674-44e504a3b67af9690a8bdcef3d4252ce.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165507-c35f6a9a-c1a8-1.png)  
这里就能够读取服务器上的任意文件，比如 C:/windows/win.ini

[![](assets/1706959674-bea7fd20141ac66b706dd0bb440b979b.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165520-cb9b3aea-c1a8-1.png)

[![](assets/1706959674-eddc8991a6f5a624d0d06316c14fa4ac.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165529-d06acf22-c1a8-1.png)  
如果传入的 url 是以//开头的，就是读取网络共享文件的内容

[![](assets/1706959674-ab01085c3392147141daa1379cda9611.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165555-e04c7cce-c1a8-1.png)  
如果传入的 url 是以 ftp 开头，就是读取 ftp 服务上的文件内容

[![](assets/1706959674-f5a283f9c84ece148d450701c39f311f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165643-fcbcad34-c1a8-1.png)  
还有就是访问 http 或 https 的 web 服务

[![](assets/1706959674-0868aa24381f154afe45d3f17f4722ef.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240202165655-03ef622c-c1a9-1.png)  
data 为请求的返回值
