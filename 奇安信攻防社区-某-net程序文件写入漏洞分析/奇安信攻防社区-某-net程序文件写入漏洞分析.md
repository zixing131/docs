

# 奇安信攻防社区-某.net程序文件写入漏洞分析

### 某.net程序文件写入漏洞分析

前段时间看到公开了一个漏洞payload很奇怪，正好有空就分析了一下，也是第一次接触了.net的代码审计，一通分析下来，发现和审计java和php的思路是一样的...

前段时间看到公开了一个漏洞payload很奇怪，正好有空就分析了一下，也是第一次接触了.net的代码审计，一通分析下来，发现和审计java和php的思路是一样的...

在拿到源码后，发现这个oa是用.net编写的，基本所有的预编译的ASP.NET的Web应用程序使用的.Net程序集（通常是DLLs）都默认放在`bin`文件下，这一点和java写的程序中将jar都放在`WEB-INF/lib`中类似。.NET的DLL和Java的JAR都是用于打包和分发代码的文件格式。.NET的DLL是用于.NET Framework的动态链接库，而Java的JAR是用于Java平台的Java归档文件。

通常反编译.net的工具dnSpy, ILSpy等，但是由于ILSpy一直在维护更新，大部分人都偏向于使用这个，虽然dnSpy工具以及在2020年已经归档，但是在github通过`dnspy fork:only`搜索到仍有非官方的爱好者对该工具进行维护，例如项目地址:[dnSpy](https://github.com/dnSpyEx/dnSpy)。

# 0x01 路由分析

首先查看`Global.asax`，他 是 ASP.NET 应用程序中的一个文件，它作为处理应用程序级事件和自定义应用程序行为的中心位置。通常被称为全局应用程序类。其中的`Inherits`中定义了定义供页继承的代码隐藏类

![image-20240117141338001](assets/1709602506-57975bf7e23c4c582cde8e837a20ade1.jpg)

然后再dnSpy中搜索这个类目（最好打开全词匹配排除其他干扰项），然后在`Application_Start`中定义了在应用程序启动时运行的代码

![image-20240117141504795](assets/1709602506-6d821bf92d81a84180627a9ce261c85a.jpg)

通过`WebApiConfig.Register`可以看到这里定义了通过api访问的路由，定义了包含控制器和方法

在 ASP.NET Web API 中，*控制器*是处理 HTTP 请求的类。控制器的公共方法称为*操作方法*或简称为*action*。当 Web API 框架收到请求时，它将请求路由到操作。

![image-20240116182828997](assets/1709602506-3e7f4df06eb8fdd0a745719379ba00da.jpg)

然后在`WebApi.Areas.Api.Controllers`命名空间中寻找到所有控制器和方法

![image-20240117144434626](assets/1709602506-be54fe1d992259d34afcb0998a609678.jpg)

# 0x02 漏洞分析

首先查看payload

```php
POST /api/files/UploadFile HTTP/1.1
Host: your-ip
Accept-Encoding: gzip
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_3) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/12.0.3 Safari/605.1.15
Content-Type: application/x-www-form-urlencoded
Content-Length: 1926

token=zxh&FileName=/../../manager/a.aspx&pathType=1&fs=[60,37,64,...]
```

通过路由`/api/files/UploadFile`锁定`WebApi.Areas.Api.Controllers.filesController.UploadFile`方法

![image-20240117151647439](assets/1709602506-290f7683915e40945bd58aa5b11012f0.jpg)

在这个方法中接受四个参数，分别为token,fs,FileName,pathType。

首先会经过`base.IsAuthorityCheck()`进行鉴权

![image-20240117151918695](assets/1709602506-fd76750961550afd0f54f3c9f295bb79.jpg)

这里的`byValue`通过参数`token`获得，并且这里判断的内容直接被硬编码，所以直接`token=zxh`即可`return new UserInfo();`

接着返回UploadFile方法看fs

```php
fs = JsonConvert.DeserializeObject<byte[]>(base.getByValue("fs"));
```

`JsonConvert.DeserializeObject<byte[]>` 是使用 Json.NET 库（Newtonsoft.Json）中的 `JsonConvert` 类来将 JSON 字符串反序列化为 `byte[]` 数组的过程。所以这里只需要传入一个数组就可以了

![image-20240117162553752](assets/1709602506-7a315d8c32218540c455ac7542ed0780.jpg)

因在在最后的`SingleBase<fileServices>.Instance.UploadFile(fs, FileName, pathType)`中

![image-20240117160455901](assets/1709602506-342b21b28146e34bf43433b5f78ac4ea.jpg)

会直接将byte类型直接通过`WriterTo`写入，所以可以用python生成一个webshell的字节数据

```php
webshell = '<%now()%>'
print(str([ord(x) for x in webshell]).replace(' ', ''))
```

![image-20240117161828039](assets/1709602506-305f39a4771dc5c945bfcd22044607a3.jpg)

然后在看写入的路径

![image-20240117160642227](assets/1709602506-2698b4f70a382fed4ba864ff83913dea.jpg)

写入的路径是通过传入的`pathType`决定的，在`GetFolderPathByType`中定义了`pathType`的值对应的路径

![image-20240117160727483](assets/1709602506-2cb58d25b79ef786117cc07c890ddc62.jpg)

在`web.config`中定义了各个值对应的路径，但是程序实在manage里，如果上传到这些目录也无法访问到

![image-20240117160937318](assets/1709602506-dfbe1bd552a2def7d19b24726e9adebf.jpg)

所以这里的FileName需要通过路径穿越到web目录。最后`memoryStream.WriteTo(fileStream);`将webshell写入

# 0x03 more

## 1、任意文件写入

在分析这个漏洞的时候，发现在这个控制器下还有四个类似上传方法

![image-20240117172054824](assets/1709602506-f3023b1800d0b4dde5b039f84a4f2781.jpg)

这里找到一处UploadFile2，这里获取路径的方式是通过pathType控制，但是多了两个参数startPoint和dataSize

![image-20240117173215411](assets/1709602506-34399fcce466fecc6efba5b7890515fc.jpg)

这里会判断startPoint是否为0，如果0就直接写入文件，如果不为0，则需要被写入的文件存在，且startPoint的值必须为被写入文件的长度，然后通过append追加的方式写入，然后datasize是写入的位数，需要与写入文件的字符个数相同。

![image-20240117173206283](assets/1709602506-d8700be6af055d0005feb0e27ac634aa.jpg)

payload就不放了，能看懂的人自然能构建出来，后面的`UploadBillFile2`原理也是一样的，只不过是上传的路径变了，依然可以利用文件名进行目录穿梭上传到web目录下。

![image-20240117174128940](assets/1709602506-a79beb81595e6b68c36b1d91e01f2c5b.jpg)

接着看下一处，但是这里鸡肋，因为这里的路径是通过`Path.GetTempPath()`获取的，也就是说只能上传到`%TMP%`目录下，但依然可以路径穿越到启动目录上传木马等。

![image-20240117172955317](assets/1709602506-7dec48225569c7c2f51ba762348f9d92.jpg)

## 2、任意文件读取

![image-20240118144955838](assets/1709602506-e76c478874f66e5daed3309b50da92c4.jpg)

同样在files控制器中，我们发现了很多与下载相关的接口，并且都是通过硬编码进行鉴权的。

![image-20240118145100494](assets/1709602506-612998e51155c3cd3ea51293f717578e.jpg)

例如这个接口，获取两个参数requestFileName和pathType

![image-20240118145152494](assets/1709602506-d98fe2c9afed493a0ca3ee346f904e4b.jpg)

pathType同样是选择不同的路径，可以用requestFileName进行路径穿越读到我们想要的文件

payload:

```php
POST /api/files/DownloadFile HTTP/1.1

token=zxh&requestFileName=../../manager/web.config&pathType=1
```
