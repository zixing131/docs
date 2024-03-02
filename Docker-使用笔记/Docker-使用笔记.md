
# Docker 使用笔记

- - -

## [#](#_0x00-%E5%89%8D%E8%A8%80) 0x00 前言

平时在使用 Docker 时，经常会碰到忘记相关命令的情况，因此平时忘记一个就会记录一个，经过多年的记录，Docker 相关的笔记已经记录了不少。

最近在看代码审计的时候又提到了 Docker，正好借着这个机会好好的把原来记录的比较乱的 Docker 笔记整理一下。

如果你也面临过「在使用 Docker 时，时不时就会忘记某条命令」的情况，那么我相信本篇文章应该会对你有所帮助。

## [#](#_0x01-%E5%AE%89%E8%A3%85) 0x01 安装

### [#](#_1%E3%80%81%E5%AE%89%E8%A3%85-docker) 1、安装 Docker

```text
curl -fsSL https://get.docker.com/ | sh
```

或者

```text
wget -qO- https://get.docker.com/ | sh
```

在命令中输入以下命令，如果输出 helloword 表示 Docker 安装成功。

```text
docker run ubuntu echo "helloworld"
```

![](assets/1708582975-82511bd397c0c6525edf409a0d6a0d5d.png)

### [#](#_2%E3%80%81%E5%AE%89%E8%A3%85-docker-compose) 2、安装 Docker-Compose

```text
sudo curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

### [#](#_3%E3%80%81docker-%E8%AE%BE%E7%BD%AE%E5%9B%BD%E5%86%85%E9%95%9C%E5%83%8F%E6%BA%90) 3、Docker 设置国内镜像源

```text
vi /etc/docker/daemon.json

{
    "registry-mirrors": ["http://hub-mirror.c.163.com"]
}

systemctl restart docker.service
```

国内加速地址如下：

```text
Docker 中国区官方镜像
https://registry.docker-cn.com

网易
http://hub-mirror.c.163.com

中国科技大学
https://docker.mirrors.ustc.edu.cn

阿里云容器  服务
https://cr.console.aliyun.com/
```

## [#](#_0x02-%E4%BD%BF%E7%94%A8) 0x02 使用

### [#](#_1%E3%80%81%E6%90%9C%E7%B4%A2%E9%95%9C%E5%83%8F) 1、搜索镜像

```text
docker search centos
```

### [#](#_2%E3%80%81%E6%8B%89%E5%8F%96%E9%95%9C%E5%83%8F) 2、拉取镜像

```text
docker pull centos
```

### [#](#_3%E3%80%81%E6%9F%A5%E7%9C%8B%E9%95%9C%E5%83%8F%E6%96%87%E4%BB%B6) 3、查看镜像文件

```text
docker images
```

查看镜像层级关系

```text
docker images tree

# 以前这个命令是：

docker images --tree
```

### [#](#_4%E3%80%81%E6%9F%A5%E7%9C%8Bdocker%E6%89%80%E6%9C%89%E8%BF%9B%E7%A8%8B) 4、查看 docker 所有进程

```text
docker ps -a
```

### [#](#_5%E3%80%81%E5%BC%80%E5%90%AF%E5%AE%B9%E5%99%A8) 5、开启容器

开启指定容器，这里的容器名为 Web

```text
docker start web
```

启动所有容器

```text
docker start $(docker ps -aq)
```

### [#](#_6%E3%80%81%E8%BF%9B%E5%85%A5%E6%AD%A3%E5%9C%A8%E8%BF%90%E8%A1%8C%E7%9A%84%E5%AE%B9%E5%99%A8) 6、进入正在运行的容器

docker 创建的

```text
docker attach web
```

docker-compose 创建的

container\_name 需要在 docker-compose.yml 文件中查看

```text
docker-compose exec container_name bash
```

### [#](#_7%E3%80%81%E6%8C%87%E5%AE%9A%E7%AB%AF%E5%8F%A3%E5%90%AF%E5%8A%A8%E5%88%9B%E5%BB%BA%E8%BF%9B%E5%85%A5%E5%AE%B9%E5%99%A8) 7、指定端口启动创建进入容器

```text
docker run -p 9992:80 -p 8882:8888 -it ubuntu /bin/bash
docker run --name web1 -p 9991:80 -p 8881:8888 -it centos /bin/bash
```

### [#](#_8%E3%80%81%E5%AF%BC%E5%87%BA%E5%AF%BC%E5%85%A5%E9%95%9C%E5%83%8F) 8、导出导入镜像

export\\import 导入导出

```text
docker export web > /home/docker_web.tar
docker import /home/docker_web.tar
```

save\\load 导入导出

```text
docker save 9610cfc68e8d > /home/docker_web.tar
docker load < /home/docker_web.tar
```

export\\import 与 save\\load 的区别：

-   export\\import 导出的镜像文件大小要小于 save\\load 导出的镜像
    
-   export\\import 是根据容器拿到的镜像，再导入时会丢失镜像所有的历史，所以无法进行回滚操作；而 save\\load 的镜像，没有丢失镜像的历史，可以回滚到之前的层。
    

核心原因是 export 是针对容器的导出，所以只有所有层组合的最终版本；而 save 则是针对镜像的，所以可以看到每一层的信息。

### [#](#_9%E3%80%81%E4%BF%AE%E6%94%B9%E6%AD%A3%E5%9C%A8%E8%BF%90%E8%A1%8C%E7%9A%84%E5%AE%B9%E5%99%A8%E7%AB%AF%E5%8F%A3%E6%98%A0%E5%B0%84) 9、修改正在运行的容器端口映射

a、停止容器

b、停止 docker 服务 (systemctl stop docker)

c、修改这个容器的 hostconfig.json 文件中的端口（原帖有人提到，如果 config.v2.json 里面也记录了端口，也要修改）

```text
cd /var/lib/docker/3b6ef264a040* 	# 这里是 CONTAINER ID

vi hostconfig.json

# 如果之前没有端口映射，应该有这样的一段：
"PortBindings":{}

# 增加一个映射，这样写：
"PortBindings":{"3306/tcp":[{"HostIp":"","HostPort":"3307"}]}

# 前一个数字是容器端口，后一个是宿主机端口
# 而修改现有端口映射更简单，把端口号改掉就行
```

d、启动 docker 服务 (systemctl start docker)

e、启动容器

### [#](#_10%E3%80%81%E6%96%87%E4%BB%B6%E4%BC%A0%E8%BE%93) 10、文件传输

```text
docker cp 本地文件路径 ID 全称：容器路径

# 或者

docker cp ID 全称：容器文件路径 本地路径
```

### [#](#_11%E3%80%81%E5%90%8E%E5%8F%B0%E8%BF%90%E8%A1%8Cdocker) 11、后台运行 docker

启动全新的容器，该命令会在后台运行容器，并返回容器 ID

```text
docker run -d
```

对于现有的容器

```text
ctrl+P+Q
```

## [#](#_0x03-%E5%8D%B8%E8%BD%BD) 0x03 卸载

### [#](#_1%E3%80%81%E5%81%9C%E6%AD%A2%E5%AE%B9%E5%99%A8) 1、停止容器

停止指定容器

```text
docker stop web
```

停止所有容器

```text
docker stop $(docker ps -aq)
```

### [#](#_2%E3%80%81%E5%88%A0%E9%99%A4%E5%AE%B9%E5%99%A8%E5%92%8C%E9%95%9C%E5%83%8F) 2、删除容器和镜像

删除指定容器

```text
docker container rm d383057928b4	# 指定容器 ID
```

删除所有已退出的容器

```text
docker rm $(docker ps -q -f status=exited)
```

删除所有已停止的容器

```text
docker rm $(docker ps -a -q)
```

删除所有正在运行和已停止的容器

```text
docker stop $(docker ps -a -q)
docker rm $(docker ps -a -q)
```

删除所有容器，没有任何标准

```text
docker container rm $(docker container ps -aq)
```

Docker 资源清理

```text
docker container prune	# 删除所有退出状态的容器
docker image prune			# 删除 dangling 或所有未被使用的镜像
docker network prune		# 删除所有未使用的网络
docker volume prune			# 删除未被使用的数据卷
docker system prune			# 删除已停止的容器、dangling 镜像、未被容器引用的 network 和构建过程中的 cache，安全起见，这个命令默认不会删除那些未被任何容器引用的数据卷，如果需要同时删除这些数据卷，你需要显式的指定 --volumns 参数
docker system prune --all --force --volumns 	# 这次不仅会删除数据卷，而且连确认的过程都没有了！注意，使用 --all 参数后会删除所有未被引用的镜像而不仅仅是 dangling 镜像
```

删除所有镜像

```text
docker rmi $(docker images -q)
```

### [#](#_3%E3%80%81%E5%8D%B8%E8%BD%BDdocker) 3、卸载 Docker

```text
yum list installed | grep docker
yum -y remove docker.x86_64
```

### [#](#_4%E3%80%81%E5%8D%B8%E8%BD%BDdocker-compose) 4、卸载 Docker-compose

```text
rm /usr/local/bin/docker-compose
```

> 参考资料：
> 
> [https://blog.csdn.net/a906998248/article/details/46236687 (opens new window)](https://blog.csdn.net/a906998248/article/details/46236687)
> 
> [https://blog.csdn.net/wesleyflagon/article/details/78961990 (opens new window)](https://blog.csdn.net/wesleyflagon/article/details/78961990)
