# Docker基础

## 概述

[Docker](https://www.docker.com) 是个划时代的开源项目，它彻底释放了计算虚拟化的威力，极大提高了应用的维护效率，降低了云计算应用开发的成本！使用 Docker，可以让应用的部署、测试和分发都变得前所未有的高效和轻松！

无论是应用开发者、运维人员、还是其他信息技术从业人员，都有必要认识和掌握 Docker，节约有限的生命。

### 背景

- 云计算，敏捷开发，高频度的弹性伸缩需求
- 环境依赖问题：一种既能够屏蔽操作系统差异，又能够以不降低性能的方式来运行应用的技术

### 容器与Docker

- 容器：容器技术起源于Linux，是一种内核虚拟化技术，基于 Linux 内核的 cgroup，namespace，以及 OverlayFS 类的 Union FS 等技术，提供轻量级的虚拟化，隔离进程和资源，属于**操作系统层面**的虚拟化技术。由于隔离的进程独立于宿主和其它的隔离的进程，因此也称其为容器。

- Docker：Docker是第一个使容器能在不同机器之间移植的系统。它不仅简化了打包应用的流程，也简化了打包应用的库和依赖，甚至整个操作系统的文件系统能被打包成一个简单的可移植的包，这个包可以被用来在任何其他运行Docker的机器上使用。

### Docker与虚拟机

**Docker** 在容器的基础上，进行了进一步的封装，从文件系统、网络互联到进程隔离等等，极大的简化了容器的创建和维护。容器和虚拟机具有相似的资源隔离和分配方式，容器虚拟化了**操作系统**而不是硬件，更加便携和高效。

- 虚拟机：传统虚拟机技术是虚拟出一套硬件后，在其上运行一个完整操作系统，在该系统上再运行所需应用进程；

- 容器：容器内的应用进程直接运行于宿主的内核，容器内没有自己的内核，而且也没有进行硬件虚拟。因此容器要比传统虚拟机更为轻便。

<img src="https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202131238633.png" alt="image-20220213123817566" style="zoom: 80%;" />

### 微服务发展历程

| 物理机                                                       | 虚拟机                                                       | 容器化                                                       |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 1. 安装系统<br />2. 依赖环境<br />3. 运行应用程序<br />4. 增加物理机，提高并发量 | 1. 将系统及依赖环境打包为系统镜像<br />2. 构建虚拟机<br />3. 运行应用程序 | 1. 基于基础镜像打包依赖环境和应用，生成应用程序镜像<br />2. 运行在任何部署有docker环境的机器上 |

### 为什么使用docker

作为一种新兴的虚拟化方式，Docker 跟传统的虚拟化方式相比具有众多的优势。

- 更高效的利用系统资源：容器不需要进行硬件虚拟以及运行完整操作系统等额外开销，性能接近原生
- 更快速的启动时间： Docker 容器应用直接运行于宿主内核，无需启动完整的操作系统，因此可以做到秒级、甚至毫秒级启动
- 一致的运行环境： Docker 的镜像提供了除内核外完整的运行时环境，确保了应用运行环境一致性
- 持续交付和部署：使用 Docker 可以通过定制应用镜像来实现持续集成、持续交付、部署
- 更轻松的迁移：Docker 确保了执行环境的一致性，使得应用的迁移更加容易
- 更轻松的维护和扩展：Docker 使用的分层存储以及镜像的技术，一大批高质量的 官方镜像，可直接生产使用，也可定制开发。

## docker安装

- VM16虚拟机+CentOS7.9、或使用阿里云抢占式实例（2核4G￥0.04/时）
- docker、docker-compose安装（在线、离线方式）
- 注意事项：开机启动，生产配置等

## 基本概念

### 三大组件

- 镜像（Image）：容器镜像是容器应用打包的标准格式，封装了应用程序及其所有软件依赖的二进制数据。镜像不包含任何动态数据，其内容在构建之后也不会被改变。在部署容器化应用时可以指定镜像，镜像可以来自于Docker Hub（官方仓库），公有云镜像服务，或者用户的私有镜像仓库。镜像ID可以由镜像所在仓库URI和镜像Tag（默认为`latest`）唯一确认。
- 镜像仓库（Registry）：容器镜像仓库是一种存储库，用于存储基于容器应用开发的容器镜像。

- 容器（Container）：Docker容器通常是一个Linux容器，它基于Docker镜像被创建。一个运行中的容器是一个运行在Docker主机上的进程，但它和主机，以及所有运行在主机上的其他进程都是隔离的。这个进程也是资源受限的，意味着它只能访问和使用分配给它的资源（CPU、内存等）。

镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的类和实例一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。

### 流程

1. 首先开发者在开发环境机器上开发应用并**制作镜像**。Docker执行命令，构建镜像并存储在机器上。
2. 开发者发送**上传镜像**命令。Docker收到命令后，将本地镜像上传到镜像仓库。
3. 开发者向生产环境机器发送运行镜像命令。生产环境机器收到命令后，Docker会从镜像仓库拉取镜像到机器上，然后基于镜像运行容器。

<img src="https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202131247956.png" alt="image-20220213124725872" style="zoom: 67%;" />

## 常用命令

### 镜像

- 在官方仓库中搜索镜像，建议直接在[网页](https://registry.hub.docker.com/)中搜索，信息更全

```shell
docker search nginx
```

- 拉取镜像，默认从官方仓库拉取镜像，未指明Tag时默认为Latest

```shell
docker pull nginx
docker pull l2c.harbor.cn/library/nginx:1.17-alpine 
```

- 推送镜像至远程仓库

```shell
docker push l2c.harbor.cn/library/nginx:1.17-alpine 
```

- 标记（Tag）,常用于重新指定仓库名，这样才能推送到远程仓库

```shell
docker tag nginx:1.17-alpine  l2c.harbor.cn/library/nginx:1.17-alpine 
```

- 迁移镜像

```shell
# 1.源主机上打包
docker save [repo:tag] > [file.tar]
# 2.目标主机上还原
docker load --input  [file.tar]
```

- 删除镜像，若有容器基于此镜像运行，则无法删除

```shell
docker rmi l2c.harbor.cn/library/nginx:1.17-alpine 
```

### 容器

```shell
# 查询
docker ps -a

# 停止
docker stop [容器名/ID]

# 删除（停止后才可删除）
docker rm [容器名/ID]

# 强制删除
docker rm -f [容器名/ID]

# 重启
docker restart [容器名/ID]

# 查看日志
docker logs -f [容器名/ID]
```

- 启动

```shell
# 后台启动（-d）
docker run -d --name nginx-test -p 8001:80 nginx:1.17-alpine 

# 前台启动,打开一个交互终端，并指定--rm，退出时自动删除容器
docker run -it --rm --name nginx-test -p 8001:80 nginx:1.17-alpine 
```



## 简单示例

### Nginx

#### docker run启动

```
docker run -d --name nginx-test -p 8001:80 nginx:1.17-alpine 
```

- 进入容器，修改主页

```shell
docker exec -it nginx-test sh

# 进入主页路径
cd /usr/share/nginx/html

# 追加一行
echo "hello nginx!" >> index.html
```

- 持久化

> 注：持久化挂载会将宿主机目录覆盖容器内的目录，从而导致容器目录下为空

```shell
docker run --name nginx-test -p 8001:80 \
-v /opt/app/docker/119/nginx/html:/usr/share/nginx/html \
-d nginx:1.17-alpine 
```

### MySQL

```
version: '3'
services:
  mysql:
    restart: always
    image: mysql:5.7
    container_name: mysql
    environment:
      - "MYSQL_ROOT_PASSWORD=mysql@1ot"
      - "TZ=Asia/Shanghai"
    ports:
      - 32326:3306
    volumes:
      - ./data/mysql:/var/lib/mysql
      - ./data/conf/my.cnf:/etc/my.cnf
```
