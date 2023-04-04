# docker-compose安装Harbor镜像仓库

[toc]

## 准备事项

> 安装准备，详见官网：[Harbor Installation Prerequisites](https://goharbor.io/docs/2.2.0/install-config/installation-prereqs/)

- 检测80端口是否被占用：`netstat -tunlp|grep 80`，若使用其它端口，需在2.2节配置的hostname中加上端口号
- harbor可能与ceph存在冲突，尽量选择其它节点部署
- 环境：docker、docker-compose、https

![image-20210324163804178](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210324163804.png)

### 安装docker

- 移除电脑上原有的dockers

```shell
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
yum install -y yum-utils
 
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
 
yum install -y docker-ce
```

- 配置阿里云镜像加速器

  > 加速器地址：阿里云控制台--容器镜像服务--镜像工具--镜像加速器

```shell
mkdir -p /etc/docker
 
tee /etc/docker/daemon.json <<-'EOF'
{
"registry-mirrors": ["https://ohdpuoqu.mirror.aliyuncs.com"]
}
EOF
```


  - 验证

```shell
sudo systemctl daemon-reload &&
sudo systemctl restart docker &&
sudo systemctl enable docker &&
sudo systemctl status docker
```

### 安装docker-compose

- [下载docker-compose](https://github.com/docker/compose/releases)
- 授权、移动（并改名）至可执行目录下

```shell
- 授权
chmod +x

- 移动并改名（去掉版本号）
mv [] /usr/local/bin/docker-compose

- 验证
docker-compose version
```

![image-20210620143258367](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210620143258.png)

### 配置HTTPS访问证书

- 在工作目录下创建证书存放目录

```shell
mkdir ssl 
cd ssl 
```

- 创建证书

```shell
openssl req  -newkey rsa:4096 -nodes -sha256 -keyout ca.key -x509 -days 3650 -out ca.crt

# 一路回车5次直至出现Common Name 输入（因为是CA，可不输入IP或域名）：Harbor Cert Root CA  
注：Harbor为自定义名称
```

![image-20210620141557774](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210620141557.png)

- 生成证书签名请求

```shell
openssl req  -newkey rsa:4096 -nodes -sha256 -keyout harbor.key -out harbor.csr

# 一路回车5次出现Common Name 输入IP或域名：lc.harbor.cn
```

![image-20210620141648882](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210620141648.png)

- 新建extfile.cnf

```shell
--- vim extfile.cnf  --- 

subjectAltName = @alt_names
extendedKeyUsage = serverAuth

[alt_names]
# 域名，如有多个用DNS.2,DNS.3…来增加
DNS.1 = lc.harbor.cn
DNS.2 = *.harbor.cn
# IP地址, 服务器的ip
IP.1 = 172.16.16.104
IP.2 = 127.0.0.1
```

- 生成证书

```shell
openssl x509 -req -days 3650 -in harbor.csr -CA ca.crt -CAkey ca.key -CAcreateserial -extfile extfile.cnf -out harbor.crt
```

![image-20210620141826278](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210620141826.png)

### 为docker login配置证书

- 方式一：配置docker的CA证书，不需要重启docker（墙裂建议）

```shell
- 创建目录
mkdir -p /etc/docker/certs.d/lc.harbor.cn/
- 分发证书
cp ca.crt /etc/docker/certs.d/lc.harbor.cn/
```

- 方式二：配置insecure-registries，跳过https认证，重启docker生效

```shell
vim /etc/docker/daemon.json

"insecure-registries": ["xxx","lc.harbor.cn"],

-----------
- 重启docker
systemctl restart docker
```

![image-20210620141940796](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210620144236.png)

- 方式三：配置系统级CA认证，需重启docker

```shell
- 分发证书
scp ca.crt root@172.16.16.103:/etc/pki/ca-trust/source/anchors/

- 更新配置并重启docker
update-ca-trust extract &&
systemctl restart docker
```

### 附：Ansible一键配置集群证书

- hosts

```
[master]
172.16.16.11 node_name=k8s-master1
172.16.16.12 node_name=k8s-master2
172.16.16.13 node_name=k8s-master3

[node]
172.16.16.101 node_name=k8s-node1
172.16.16.102 node_name=k8s-node2
172.16.16.103 node_name=k8s-node3
172.16.16.104 node_name=k8s-node4
```

- scp-harbor-cert.yml（将ca证书放在同目录下）

```
- hosts:
  - master
  - node
  tasks:
  - name: 删除相关的host解析
    shell: sed -i '/lc.harbor.cn/d' /etc/hosts
  - name: 添加hosts
    lineinfile: dest=/etc/hosts line="172.16.16.104 lc.harbor.cn"
  - name: 创建证书目录
    file: dest=/etc/docker/certs.d/lc.harbor.cn/ state=directory
  - name: 分发证书
    copy: src=./ca.crt dest=/etc/docker/certs.d/lc.harbor.cn/
```

- 执行

```
ansible-playbook -i hosts scp-harbor-cert.yml -uroot -k
```

## docker-compose安装Harbor

### 下载harbor安装包

- [下载harbor](https://github.com/goharbor/harbor/releases)
- 解压

```shell
tar zxvf harbor-offline-installer-v2.2.2.tgz
```

![image-20210620142144226](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210620142144.png)

### 配置harbor

- hostname：本机IP
- 证书地址：见1.3节
- 仓库地址：自定义

```shell
- 编辑harbor.yml.tmpl（按需修改）

vim harbor.yml.tmpl
------------------------

hostname: 172.16.16.104

http:
  port: 80
https:
  port: 443
  certificate: /root/harbor/ssl/harbor.crt
  private_key: /root/harbo/ssl/harbor.key

harbor_admin_password: Harbor12345

------------------------

- 生成harbor.yml文件
cp harbor.yml.tmpl harbor.yml
```

![image-20210620143010495](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210620143010.png)

- 解压镜像

```shell
docker load --input harbor.v2.2.2.tar.gz
```

![image-20210620142508960](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210620142509.png)

- 执行

```shell
./prepare
./install.sh
```

![image-20210620143353484](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210620143353.png)

### 设置开机启动

- 虽然docker-compose.yml中所有服务restart策略为always，但是docker或服务器重启后并不是所有服务都会自动重启

![image-20210621142625055](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210621142625.png)

- 配置开机启动

> [参考官方Issue](https://github.com/goharbor/harbor/issues/7008)

```
vim /usr/lib/systemd/system/harbor.service
------------------------------------------------

[Unit]
Description=Harbor
After=docker.service systemd-networkd.service systemd-resolved.service
Requires=docker.service
Documentation=http://github.com/vmware/harbor

[Service]
Type=simple
Restart=on-failure
RestartSec=5
#注意docker-compose和harbor的安装位置
ExecStart=/usr/local/bin/docker-compose -f  /root/harbor/harbor/docker-compose.yml up
ExecStop=/usr/local/bin/docker-compose -f /root/harbor/harbor/docker-compose.yml down

[Install]
WantedBy=multi-user.target
```

- 启动harbor服务

```
systemctl enable harbor
systemctl restart harbor
```

- 查看harbor服务状态

```
systemctl status harbor
```

## 登录harbor仓库

### 网页登录

- 配置hosts

```shell
win10 hosts路径：C:\Windows\System32\drivers\etc\hosts
------------------

121.89.211.133 lc.harbor.cn 
```

![image-20210620143924267](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210620143924.png)

### docker登录

- 配置hosts

```shell
- vim /etc/hosts（公网/私网按需配置）
------------------------

172.16.16.104 lc.harbor.cn 
```

- docker login

```shell
docker login lc.harbor.cn -uadmin
```

![image-20210620144412989](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210620144413.png)

### 推送镜像

```shell
- 拉取官方镜像（默认tag为latest）
docker pull nginx

- 查看镜像
docker images |grep nginx

- 打tag
docker tag nginx:latest lc.harbor.cn/library/nginx:latest

- 推送（默认推送镜像需要登录）
docker push lc.harbor.cn/library/nginx:latest
```

![image-20210620144722610](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210620144722.png)

- 网页上查看镜像

![image-20210620144905455](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210620144905.png)

### 拉取镜像

- 先移除本地打了tag的镜像

```shell
docker rmi lc.harbor.cn/library/nginx:latest
```

- 拉取镜像

```shell
docker pull lc.harbor.cn/library/nginx:latest
```

![image-20210620145218819](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210620145218.png)

## Harbor后台API2.0

### 准备事项

- API手册链接位于网页左下角
- 标记多个tag并推送（不能使用相同的镜像）

```shell
docker tag nginx:1.17.0 lc.harbor.cn/library/nginx:100
docker tag nginx:1.18.0 lc.harbor.cn/library/nginx:18
docker tag nginx:1.19.0 lc.harbor.cn/library/nginx:19
docker tag nginx:1.20.0 lc.harbor.cn/library/nginx:20
docker tag nginx:latest lc.harbor.cn/library/nginx:latest

推送顺序：latest，100，18，19，20
```

### HarborAPI2.0删除镜像示例

1. 获取项目/仓库下的镜像信息

```shell
image_info=$(curl -s -k -u admin:Harbor12345 -X GET "https://lc.harbor.cn/api/v2.0/projects/library/repositories/nginx/artifacts?page=1&page_size=10&with_tag=true&with_label=false&with_scan_overview=false&with_signature=false&with_immutable_status=false" -H "accept: application/json")

echo $image_info
```

2. 提取出镜像tag

```shell
tags="$(echo "$image_info" | tr , '\n' | grep name | cut -d '"' -f4)"

echo $tags
```

![image-20210620153727861](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210620153727.png)

3. 镜像tag排序，取出最近3个镜像以外的tag

```shell
for tag in `echo ${tags} | awk 'BEGIN{i=1}{gsub(/ /,"\n");i++;print}' | awk -F. '{print $NF}' | sed "1,3d"`;
do
curl -s -k -u admin:Harbor12345 -X DELETE https://lc.harbor.cn/api/v2.0/projects/library/repositories/nginx/artifacts/${tag}
done
```

![image-20210620153807387](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210620153807.png)

![image-20210620153931583](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210620153931.png)

