

# 某在线监控信息管理平台-java 代审 - 先知社区

某在线监控信息管理平台-java 代审

- - -

## 前言

针对客户需求，进行了一次甲方系统的代码审计。

## Spring 方法未拦截

通过审计发现系统存在权限校验，针对 login.jsp 中的方法不进行拦截  
[![](assets/1709518946-0c9c46ddeb777daac11929ca14e2ebbc.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301195231-2f30bbc8-d7c2-1.png)

## 任意文件下载

login.jsp 中存在相关可用功能，发现存在任意文件下载漏洞

[![](assets/1709518946-600ab8224d02d64cac4779847f2b0f07.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301195533-9be6668c-d7c2-1.png)

## 任意文件包含及任意文件上传

该功能点存在任意文件包含以及任意文件上传漏洞，经利用，发现任意文件上传漏洞回显路径未知，任意文件下载漏洞成功利用

[![](assets/1709518946-b0161cb9aad7d5faaf8c34e22c285128.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301195646-c730f0b4-d7c2-1.png)

[![](assets/1709518946-0c9673a8701b786d5900997178b715ea.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301195701-d061d824-d7c2-1.png)

[![](assets/1709518946-4c51fee8a7b3e06f28a0195f539c4627.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301195724-de5bb9b8-d7c2-1.png)

## 未授权任意密码修改

通过审计发现 login.jsp 中存在密码修改功能，初步判定为存在未授权任意密码修改漏洞，但由于传递参数为 userId，无法根据 userId 匹配到相关用户名，初步判定利用困难

[![](assets/1709518946-a4384627d0e1fa9dac02b4e11d775078.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301195755-f0c30854-d7c2-1.png)

[![](assets/1709518946-a66b4300638956ef58acbbb504726e80.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301195817-fd934954-d7c2-1.png)

## 默认口令

通过审计发现代码中存在默认密钥 XXX，经审计代码逻辑发现功能为添加用户功能，自动赋予默认口令，但用户登陆后会强制用户修改为强口令，利用可能较小

[![](assets/1709518946-2aa92f1676d4b9c9a92ab201cf512fbd.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301195906-1b29fe22-d7c3-1.png)

## 敏感信息泄露

经过审计，发现代码中存在邮箱

[![](assets/1709518946-404a31b0ff91b6b0794c4df09f9963d4.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301200004-3dc30dca-d7c3-1.png)
