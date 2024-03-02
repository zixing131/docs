

# 搭建 Anki 自托管同步服务器

  用的是容器的部署方式，一开始想用 Windows 客户端带的同步服务功能来部署自定义同步服务的，但是 Windows 客户端的可执行文件和相关依赖占用的空间实在太大了，遂改用自编译 Rust 二进制可执行程序作为服务组件来封装 Docker 容器。  
  期间一开始是考虑封装 Python 版的同步服务环境的，无奈有个 Anki 包库依赖在部署后运行服务时，其内部调用提示缺失，就放弃了 Python 的方案。Python 环境编译部署安装比较省时，但是封装完的镜像体积上，Rust 更有优势，占用的空间更小。  
   
   
1) 相关目录及文件

```markdown
# 相关目录结构
# tree .
.
├── Dockerfile
├── data.d
├── docker-compose.yml
└── envs
    ├── pub.env
    └── users.env

2 directories, 4 files
```

2) 准备一份 Dockerfile 镜像打包编排文件

```bash
# Dockerfile
FROM rust:1.71-alpine AS builder
ENV \
    RUSTUP_DIST_SERVER=https://mirrors.tuna.tsinghua.edu.cn/rustup \
    RUSTUP_UPDATE_ROOT=https://mirrors.ustc.edu.cn/rust-static/rustup

RUN set -aeux && sed -i "s/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g" /etc/apk/repositories
RUN set -aeux && apk add --no-cache binutils git musl-dev protobuf-dev

RUN set -aeux \
    && cargo install --git https://github.com/ankitects/anki.git --tag 23.10 anki-sync-server \
    && rm -rf /tmp/cargo-install*

RUN set -aeux \
    && mkdir -p /rootfs/ \
    && cp --parents $(which anki-sync-server) /rootfs/ \
    && cp --parents `ldd $(which anki-sync-server) | awk '{if (match($1,"/")){print $1}}')` /rootfs/ \
    && strip $(find /rootfs/ -type f -iname "*")


FROM alpine:latest
LABEL blogger="清雨@https://blog.gazer.win/"

# Init system envs
ENV \
    LANG=C.UTF-8 \
    TZ=Asia/Shanghai \
    SYNC_BASE=/opt/anki.d/sync.d

# Set work directory
WORKDIR /opt/anki.d

# Copy anki-sync-server
COPY --from=builder /rootfs/ /

# Set system environments
RUN set -aeux && sed -i "s/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g" /etc/apk/repositories
RUN set -aeux && apk add --no-cache ca-certificates tzdata

# Set the Timezone
RUN set -aeux \
    && ln -sf $(echo /usr/share/zoneinfo/${TZ}) /etc/localtime \
    && echo ${TZ} > /etc/timezone \
    && ln -sf /usr/local/cargo/bin/anki-sync-server /usr/local/bin/anki-sync-server

CMD ["anki-sync-server"]
　
```

切换工作目录到`Dockerfile`所在路径，执行镜像编译命令

```plain
docker build -t anki/syncd:v23.10 .
```

3) 准备一份容器服务`docker-compose.yml`编排文件

```yaml
version: '3.12'
services:
  anki:  # 服务名称
    container_name: anki  # 容器名称
    image: anki/syncd:v23.10
    restart: always

    env_file:
      - ./envs/pub.env
      - ./envs/users.env

    network_mode: bridge
    ports:
      - "8080:8080/tcp"

    sysctls:
      - net.ipv4.tcp_ecn=1
      - net.ipv4.tcp_ecn_fallback=1
      - net.ipv4.tcp_dsack=1
      - net.ipv4.tcp_fack=1
      - net.ipv4.tcp_sack=1
      - net.ipv4.conf.all.rp_filter=1
      - net.ipv4.conf.default.rp_filter=1
      # - net.ipv4.ip_forward=1
      - net.ipv4.tcp_keepalive_intvl=15
      - net.ipv4.tcp_keepalive_probes=5
      - net.ipv4.tcp_keepalive_time=75
      - net.ipv4.tcp_fastopen=3
      - net.ipv4.tcp_moderate_rcvbuf=1
      - net.ipv4.tcp_mtu_probing=2
      - net.ipv4.tcp_syncookies=1
      - net.ipv4.tcp_timestamps=1
      - net.ipv4.tcp_window_scaling=1

    volumes:
      - "./data.d/:/opt/anki.d/"
      - "/etc/localtime:/etc/localtime:ro"

# 配置参考资料:
# https://docs.docker.com/compose/compose-file/
# https://docs.ankiweb.net/sync-server.html#multiple-users
# https://docs.ankiweb.net/sync-server.html#with-cargo
　
```

4) 其它文件

```plain
# cat envs/pub.env
TZ=Asia/Shanghai
SYNC_BASE=/opt/anki.d/sync.d
SYNC_HOST=0.0.0.0
SYNC_PORT=8080
```

```plain
# cat envs/users.env
SYNC_USER1=user1:user1_password
SYNC_USER2=user2:user2_password
SYNC_USER3=user3:user3_password
```

最后使用`docker-compose up -d`命令启动即可  
   
   
配置及编译参考资料：  
1、[https://docs.ankiweb.net/sync-server.html#multiple-users](https://docs.ankiweb.net/sync-server.html#multiple-users)  
2、[https://docs.ankiweb.net/sync-server.html#with-cargo](https://docs.ankiweb.net/sync-server.html#with-cargo)  
 

[](https://blog.gazer.win/gradelayout.php)

最后修改：2023 年 11 月 14 日

© 允许规范转载
