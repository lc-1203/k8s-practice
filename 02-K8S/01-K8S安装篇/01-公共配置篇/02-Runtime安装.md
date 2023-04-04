# Runtime安装

> 参考自杜老师（github.com/dotbalo）的[基于世界500强的k8s实战课程](https://ke.qq.com/course/2738602)。
- [1. Containerd](#1-containerd)
- [2. Docker](#2-docker)
如果安装的版本低于1.24，选择Docker和Containerd均可，高于1.24选择Containerd作为Runtime。

> K8S官方容器运行时[安装指南](https://kubernetes.io/zh/docs/setup/production-environment/container-runtimes/)
>
> K8S与Docker等的版本[依赖关系](https://github.com/kubernetes/kubernetes/blob/master/build/dependencies.yaml)
>
> Docker离线安装包[下载地址](https://download.docker.com/linux/static/stable/x86_64/)

## 1. Containerd

- 安装docker-ce-20.10：

```shell
yum install docker-ce-20.10.* docker-ce-cli-20.10.* -y
```

- 无需启动Docker，只需要配置和启动Containerd即可
- 配置Containerd所需的模块（安装和配置的先决条件）

```shell
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

- 加载模块

```shell
modprobe -- overlay &&
modprobe -- br_netfilter
```

- 配置Containerd所需内核

```shell
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

- 加载内核

```shell
sysctl --system
```

- 配置Containerd的配置文件

```shell
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
```

- 修改配置文件（使用 systemd cgroup 驱动程序）

```shell
vim /etc/containerd/config.toml
---------------------------------------
# 将Containerd的Cgroup改为Systemd，找到containerd.runtimes.runc.options，添加SystemdCgroup = true
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

```shell
# 同时将sandbox_image的Pause镜像改成符合自己版本的地址：
# registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.6
```



- 启动Containerd，并配置开机自启动

```shell
systemctl daemon-reload &
systemctl enable --now containerd
```

- 配置crictl客户端连接的运行时位置

```shell
cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF
```

## 2. Docker

如果选择Docker作为Runtime，安装步骤较Containerd较为简单，只需要安装并启动即可

- 安装docker-ce 20.10

```
yum install docker-ce-20.10.* docker-ce-cli-20.10.* -y
```

- 配置docker

```shell
mkdir -p /etc/docker
```

```shell
# 由于新版Kubelet建议使用systemd，所以把Docker的CgroupDriver也改成systemd,
# 同时配置镜像仓库加速器（阿里云），提升获取Docker官方镜像的速度
# overlay2：默认的，需文件系统为xfs，且支持d_type
# 注意格式，勿漏掉逗号
---------------------------------------------------------
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
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

- 设置开机自启动Docker

```shell
systemctl daemon-reload && systemctl enable --now docker
```

- docker生产环境重要配置

```shell
Docker Root Dir  # 生产环境最好独立挂载SSD硬盘
Live Restore Enabled   # 重启docker，但不重启容器，上文中已配置
```

