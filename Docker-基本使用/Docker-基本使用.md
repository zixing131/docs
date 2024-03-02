
# [Docker 基本使用](https://www.raingray.com/archives/2221.html)

## 目录

-   [目录](#%E7%9B%AE%E5%BD%95)
-   [简介](#%E7%AE%80%E4%BB%8B)
-   [下载镜像](#%E4%B8%8B%E8%BD%BD%E9%95%9C%E5%83%8F)
-   [从镜像创建容器运行程序](#%E4%BB%8E%E9%95%9C%E5%83%8F%E5%88%9B%E5%BB%BA%E5%AE%B9%E5%99%A8%E8%BF%90%E8%A1%8C%E7%A8%8B%E5%BA%8F)
-   [管理容器状态](#%E7%AE%A1%E7%90%86%E5%AE%B9%E5%99%A8%E7%8A%B6%E6%80%81)
-   [把容器打包成你自己的镜像](#%E6%8A%8A%E5%AE%B9%E5%99%A8%E6%89%93%E5%8C%85%E6%88%90%E4%BD%A0%E8%87%AA%E5%B7%B1%E7%9A%84%E9%95%9C%E5%83%8F)
-   [导出导入镜像](#%E5%AF%BC%E5%87%BA%E5%AF%BC%E5%85%A5%E9%95%9C%E5%83%8F)
-   [Docker-run 踩坑](#Docker-run+%E8%B8%A9%E5%9D%91)
-   [参考资料](#%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99)

## 简介

快速启动一个运行环境供你测试，对某一漏洞快速复现很有帮助。也可用 Vagrant 起坏境。

Dockers 查看帮助命令。

```bash
docker --help #查看有那些命令
docker command --help #查看某一条命令选项
```

## 下载镜像

```bash
[root@Centos7 ~]# docker search image_name #搜索出镜像
[root@Centos7 ~]# docker pull  #拉镜像到本地
[root@Centos7 ~]# docker images  #查看已经下载的镜像

[root@Centos7 ~]# docker rmi[-f] REPOSITORY_NAME[IMAGE_ID]
#删除镜像，必须停止运行才能删除，-f强制删除。
```

## 从镜像创建容器运行程序

\-d 放到后台运行，--name 对这个容器一个说明注释，-p 将容器端口映射到本机上，访问时只用访问本机端口就行，centos 是镜像名也可以写上镜像 id，init 是在容器执行的命令 (用于管理服务不太准确)。

容器在运行完命令后会退出，由于加了 -d 会常驻在后台。

> init 进程，是由内核启动的用户级进程。内核会在过去曾使用过 init 的几个地方查找它，它的正确位置（对 Linux 系统来说）是 /sbin/init。如果内核找不到 init，它就会试着运行 /bin/sh，如果运行失败，系统的启动也会失败。

```bash
[root@Centos7 ~]# docker run -d --name Nginx -p 宿主:容器 centos init
```

进入到容器中，exec 标识要执行命令，子选项 -i 是启动标准输入，-t 开启终端，在 2ba57dcfbd46（可以是 id 也可以是 name）容器中执行一个 bash 命令，exec 执行完成后会切换到 bash\_shell 界面。

```bash
[root@Centos7 ~]# docker exec -it 2ba57dcfbd46 bash
```

## 管理容器状态

```bash
[root@Centos7 ~]# docker stop image_name[image_id]
#stop关闭容器，start启动容器，restart重启容器。

[root@Centos7 ~]# docker ps[-a] image_name[image_id]
#ps查看运行中的容器，-a容器未运行也显示

[root@Centos7 ~]# docker rm[-f] CONTAINER_ID CONTAINER_ID
#删除容器，-f强制删除(运行中也会删)。

[root@Centos7 ~]# docker stop $(docker ps -q) & docker rm $(docker ps -aq)
#删除所有docker容器
```

## 把容器打包成你自己的镜像

给容器内服务设置为开机自启，再保存为镜像别人创建容器时就能直接使用了。

```bash
[root@Centos7 ~]# docker commit 容器id name:tag 
[root@Centos7 ~]# #name是容器名称必须小写，tag一般是填写版本信息。
```

## 导出导入镜像

```bash
[root@Centos7 ~]# docker save 容器id[容器name] -o path/fiel_name.tar #导出
[root@Centos7 ~]# docker load -i path/file_name.tar #导入
```

## Docker-run 踩坑

创建容器后无法使用 `systemctl` 提示：

```bash
[root@8817430daf55 /]# systemctl
Failed to get D-Bus connection: Operation not permitted
```

建立容器需要加上 `--privileged`

```bash
[root@Centos7 ~]# docker run --name Nginx --privileged -d images_id init
```

## 参考资料

-   [https://docs.docker-cn.com/](https://docs.docker-cn.com/), 官方文档
-   [https://www.runoob.com/docker/docker-tutorial.html](https://www.runoob.com/docker/docker-tutorial.html), 菜鸟教程
-   [https://blog.csdn.net/dt763C/article/details/82719332](https://blog.csdn.net/dt763C/article/details/82719332), exec & run 命令解释

最近更新：2023 年 03 月 13 日 11:06:55

发布时间：2020 年 01 月 23 日 23:20:00
