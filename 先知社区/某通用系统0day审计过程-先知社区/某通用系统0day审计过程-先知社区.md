

# 某通用系统 0day 审计过程 - 先知社区

某通用系统 0day 审计过程

- - -

# 前言

本次审计的目标是一套也是各大高校使用的通用系统，发出来给大家学习下审计的过程

# 路由分析

直接看登录接口

[![](assets/1709022901-d1945bf6148db5413dac54a2db528afc.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240226191106-bc8eb80e-d497-1.png)

路由为 /setting.php/index/login

找对应源码

[![](assets/1709022901-ef42465bec546c7b8b480d9c459018d4.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240226191114-c1aac544-d497-1.png)

第一个接口 setting 对应 Application 下的文件

第二个接口 Index 对应 Setting 下的 Controller 文件名字

第三个接口为 Controller 的方法名字

# 代码审计

## 上传

Application\\Admin\\Controller\\UploadController.class.php

在这个路径下继承了一个 upload 控制器父类

[![](assets/1709022901-ca5a9e5666e457e571e9c68726cd39f6.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240226191124-c725ea08-d497-1.png)

所以调用的接口如下

[![](assets/1709022901-4f432f200a3f4a6fd525bfaa0a78b862.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240226191143-d2ef5f40-d497-1.png)

实例化了一个`UploadFile()`​对象然后给他的属性赋值

重点关注以下代码

```plain
$upload->exts = array('jpg', 'gif', 'png', 'jpeg'); // 设置附件上传类型
```

发现这里会调用`UploadFile()`​的魔术方法 跟进

```plain
public function __set($name,$value){
        if(isset($this->config[$name])) {
            $this->config[$name]    =   $value;
        }
    }
```

这里其实就是问题的本身，要是传进来的在 config 里头不存在则返回空，那么我们去看一下 config 的设置

```plain
private $config =   array(
        'maxSize'           =>  -1,    // 上传文件的最大值
        'supportMulti'      =>  true,    // 是否支持多文件上传
        'allowExts'         =>  array(),    // 允许上传的文件后缀 留空不作后缀检查
        'allowTypes'        =>  array(),    // 允许上传的文件类型 留空不做检查
        'thumb'             =>  false,    // 使用对上传图片进行缩略图处理
        'imageClassPath'    =>  'ORG.Util.Image',    // 图库类包路径
        'thumbMaxWidth'     =>  '',// 缩略图最大宽度
        'thumbMaxHeight'    =>  '',// 缩略图最大高度
        'thumbPrefix'       =>  'thumb_',// 缩略图前缀
        'thumbSuffix'       =>  '',
        'thumbPath'         =>  '',// 缩略图保存路径
        'thumbFile'         =>  '',// 缩略图文件名
        'thumbExt'          =>  '',// 缩略图扩展名    
        'thumbRemoveOrigin' =>  false,// 是否移除原图
        'thumbType'         =>  0, // 缩略图生成方式 1 按设置大小截取 0 按原图等比例缩略
        'zipImages'         =>  false,// 压缩图片文件上传
        'autoSub'           =>  false,// 启用子目录保存文件
        'subType'           =>  'hash',// 子目录创建方式 可以使用 hash date custom
        'subDir'            =>  '', // 子目录名称 subType 为 custom 方式后有效
        'dateFormat'        =>  'Ymd',
        'hashLevel'         =>  1, // hash 的目录层次
        'savePath'          =>  '',// 上传文件保存路径
        'autoCheck'         =>  true, // 是否自动检查附件
        'uploadReplace'     =>  false,// 存在同名是否覆盖
        'saveRule'          =>  'uniqid',// 上传文件命名规则
        'hashType'          =>  'md5_file',// 上传文件 Hash 规则函数名
        );
```

emmm 根本没有 `exts`​ 所以说后缀根本没有检测，可以从从调用的 upload 中查看

断到一个叫自动检查附件

[![](assets/1709022901-881be9ea2f2e0f34fe6e0040fdd2d41a.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240226191156-dab1b4ee-d497-1.png)

步进一下进入 check 函数存在检查文件类型

[![](assets/1709022901-58ecd44865f1be561a54fee17f4f39e8.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240226191204-deee5800-d497-1.png)

再次跟进一下 checkExt 发现是一个很强的校验白名单

```plain
private function checkExt($ext) {
        if(!empty($this->allowExts))
            return in_array(strtolower($ext),$this->allowExts,true);
        return true;
    }
```

但是重点是他并没有赋值进去

[![](assets/1709022901-f9e276046bc1f5904361ab671202d5b0.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240226191230-ee99d5a4-d497-1.png)

在这进行反向验证，讲前面的属性修改为在 config 里头的内容

[![](assets/1709022901-9feda8737ddc43946721857740ad9252.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240226191239-f3d0cb4a-d497-1.png)

再次进行断点跟到最后面的 config 的地方发现成功修改

[![](assets/1709022901-fa428efe064718b191b81e2cd1840d82.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240226191247-f8be92a4-d497-1.png)

所以这套系统只要存在 `$upload->exts =`​ 这个的上传接口 就存在任意文件上传

[![](assets/1709022901-cbfec5673852c38c4ac78f75a869ca92.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240226191255-fd79095a-d497-1.png)

## SQL(绕 redis 缓存)

直接看前台控制器了​​

找到这个路由存在 sql 的问题

[![](assets/1709022901-55a049ceb746bccc1a912525319bcd28.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240226191304-02c9e0be-d498-1.png)

在这个代码里头，可以发现 `$count`​ 跟 `$listJson`​ 是关键

`$count`​的设置是为了不频繁查询，所以这里只要设置随机伪造的 PHPSESSID 就可以了

`$listJson`​ 的设置就尤为关键了 if (!$listJson) 这里的语句是 我的 redis 去查 phone 就是要查不到 为 null 才能执行下面的 sql 语句，所以 phone 就是要不存在的手机号才行，所以就直接随机 phone 就可以了，但是他在后续的代码中把这跟手机号给设置进了`redis`​中

[![](assets/1709022901-3b2e4146befc120f660fc9dd6de349d9.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240226191312-07eb8020-d498-1.png)

所以必须要把这跟`phone`​随机化来绕`redis`​的缓存才能进行正常注入

## SSRF

这里全局搜索`curl_exec(`​ emm 感觉是这里 确实是可控的

\\Application\\Course\\Controller\\DocumentController.class.php

[![](assets/1709022901-68744740185b45707276720a58f33445.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240226191339-18168472-d498-1.png)

从逻辑上来看

`$_SERVER['HTTP_AUTHORIZATION']`​ 跟 `$_SERVER['HTTP_X_OSS_PUB_KEY_URL']`​ 这两个值都得存在 他就不会进行 403 的跳转 那么这两个都是我们可控的只要在 header 头加入即可，之后就会将我们传入的 base64 编码的值进行解码后给到 curl 进行直接 curl 执行

```plain
Authorization: Basic QWxhZGRpbjpPcGVuU2VzYW1l
X-Oss-Pub-Key-Url: aHR0cDovL2RxM2JlMC5kbnNsb2cuY24=
```

远程测了一下也是完全没问题的

[![](assets/1709022901-6f303bbdd281491df52fda40fcf84f23.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240226191358-231f2e28-d498-1.png)

加入后返回两百

[![](assets/1709022901-89f08f4a854241cec1afc58164649923.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240226191406-280d9348-d498-1.png)

最终也是测出了 SSRF

[![](assets/1709022901-9656a478f4292bb8764d40fe559657a3.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240226191424-3290fdd2-d498-1.png)
