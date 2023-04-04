# Linux-yum源配置

[toc]

## 配置为本地源（服务器无公网访问能力）

1. 确认挂载的iso文件完整路径

```shell
# 方式一：虚拟机右键--设置--CD/DVD，是否挂载ios，若无则挂载
# 方式二：系统内查看rom挂载是否存在,发现有挂载名为sr0的rom文件
lsblk
df -lh
# 查看光盘的完整路径名，/dev/cdrom
ls -l /dev | grep sr0
ls -l /dev | grep cdrom
```

2. 创建挂载目录

```shell
# 创建挂载目录
mkdir /iso
# 挂载
mount /dev/cdrom /iso
```

3. 修改yum源

> 注：这里与原生红帽系统稍有区别，centos系统已默认设置yum源

```shell
cd /etc/yum.repos.d/
vim CentOS-Base.repo
```

```shell
# 注释掉mirrorlist，放开baseurl，并配置为iso挂载目录
# 可同时修改名称，以标识为本地源
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os&infra=$infra
baseurl=file:///iso
```

```shell
[base]
name=CentOS-$releasever - Base - iso
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os&infra=$infra
baseurl=file:///iso
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#released updates 
[updates]
name=CentOS-$releasever - Updates - iso
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=updates&infra=$infra
baseurl=file:///iso
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#additional packages that may be useful
[extras]
name=CentOS-$releasever - Extras - iso
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=extras&infra=$infra
baseurl=file:///iso
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS-$releasever - Plus
mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=centosplus&infra=$infra
#baseurl=http://mirror.centos.org/centos/$releasever/centosplus/$basearch/
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
```

4. 更新yum源

```shell
# 清理缓存
yum clean all

# 重新生成缓存
yum makecache 

# 查看仓库
yum repolist
```

## 配置为阿里源（服务器具备公网访问能力）

1. 备份原文件

```shell
cd /etc/yum.repos.d/
cp CentOS-Base.repo CentOS-Base.repo.bak 
```

2. 下载阿里yum源配置文件

```shell
wget http://mirrors.aliyun.com/repo/Centos-7.repo
```

3. 替换yum源

```shell
mv Centos-7.repo CentOS-Base.repo
```

4. 更新yum源

```shell
# 清理缓存
yum clean all

# 重新生成缓存
yum makecache 

# 查看仓库
yum repolist
```

