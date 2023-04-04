# Docker及Docker-compose安装（简易版）

> 详细配置可参考 [02-K8S/01-K8S安装篇/01-公共配置篇/02-Runtime安装.md](/02-K8S/01-K8S安装篇/01-公共配置篇/02-Runtime安装.md)

## Docker安装

- docker安装要求：centos7 64位，系统内核3.10以上

```shell
# 查看系统内核：
uname -r
```

### 在线安装

> 官方安装指南：[Install Docker Engine on CentOS](https://docs.docker.com/engine/install/centos/)

- 移除电脑上原有的dockers

```
yum remove docker \
           docker-client \
           docker-client-latest \
           docker-common \
           docker-latest \
           docker-latest-logrotate \
           docker-logrotate \
           docker-engine
```

- 安装docker

```shell
# 安装所需的软件包: yum-utils提供了yum-config-manager,用于管理yum仓库
yum install -y yum-utils

# 添加阿里源仓库
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
 
# 安装 Docker Engine-Community
yum install docker-ce
```

- 配置镜像仓库加速器（阿里云），使用加速器可以提升获取Docker官方镜像的速度

```shell
mkdir -p /etc/docker
 
tee /etc/docker/daemon.json <<-'EOF'
{
"registry-mirrors": ["https://ohdpuoqu.mirror.aliyuncs.com"]
}
EOF
```

- 运行，并设置为开机启动

```shell
systemctl daemon-reload &&
systemctl restart docker &&
systemctl enable docker &&
systemctl status docker
```

- 验证

```shell
docker run hello-world
```

- 卸载

```shell
# 删除安装包
yum remove docker-ce

# 删除镜像、容器、配置文件等内容
rm -rf /var/lib/docker
```

### 离线安装

- 手动下载[docker安装包](https://download.docker.com/linux/static/stable/x86_64/)，并传输到服务器上
- 解压，拷贝至执行目录

```shell
# 解压  
tar -zvxf docker-20.10.9.tgz
# 拷贝至执行目录  
cp docker/* /usr/bin/     
```

- 生成docker.service，通过systemctl运行

```shell
vim /etc/systemd/system/docker.service
```

```Ruby
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
```

-  运行，并设置为开机启动

```shell
# 设置权限为可执行  
chmod +x /etc/systemd/system/docker.service

# 设置开机启动  
systemctl daemon-reload &&   
systemctl restart docker &&   
systemctl enable docker &&   
systemctl status docker
```

### 重要配置

- 配置镜像仓库加速器（阿里云），提升获取Docker官方镜像的速度

- Live Restore Enabled：重启docker，但不重启容器
- 生产环境最好独立挂载SSD硬盘

```shell
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
    },
  "storage-driver": "overlay2",
  "registry-mirrors": ["https://ohdpuoqu.mirror.aliyuncs.com"],
  "live-restore": true
}
EOF
```

## Docker-compose安装

> 官方安装指南：[Install Docker Compose](https://docs.docker.com/compose/install/)

- 下载并安装[docker-compose](https://github.com/docker/compose/releases/)，建议安装1.x的最后一个版本，2.x版本资源限制似乎未生效

```shell
# 可连接github，直接一键安装
curl -L "https://github.com/docker/compose/releases/download/v2.2.3/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose


# 或自行下载，然后拷贝至远程主机，重命名，赋予权限，并移动至可执行目录下
chmod +x [docker-compose]
mv [docker-compose] /usr/local/bin

# 验证
docker-compose version
```