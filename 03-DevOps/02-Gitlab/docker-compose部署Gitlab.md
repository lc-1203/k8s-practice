# docker-compose部署Gitlab

> 参考官方安装方法：[Install GitLab using Docker Compose](https://docs.gitlab.com/ee/install/docker.html#install-gitlab-using-docker-compose)

## 安装

### 安装docker-compose

> [docker及docker-compose安装](/01-Docker/01-安装/docker及docker-compose安装.md)

### docker-compose安装Gitlab

- 编排文件

```shell
vim docker-compose.yml
```

```yaml
version: '3.6'
services:
  web:
    image: 'gitlab/gitlab-ce:latest'   # 默认最新镜像，亦可指定版本
    container_name: gitlab
    restart: unless-stopped
    privileged: true
    hostname: 'gitlab.l2c.com'
    environment:
      TZ: 'Asia/Shanghai'
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://gitlab.l2c.com'  #项目clone地址,需与hostname一致，默认80端口
        gitlab_rails['time_zone'] = 'Asia/Shanghai'
        gitlab_rails['gitlab_shell_ssh_port'] = 2222
    ports:
      - '80:80'      #默认80 自定义主机端口时需同步修改external_url
      - '443:443'
      - '2222:22'    
    volumes:
      - '/data/gitlab/config:/etc/gitlab'
      - '/data/gitlab/data:/var/opt/gitlab'
      - '/data/gitlab/logs:/var/log/gitlab'
```

- 部署，启动需约2~4分钟

```shell
# 部署
docker-compose up -d

# 查看状态
docker ps -a |grep gitlab

# 停止
docker-compose down
```

- 登录：
  - 通过IP访问
  - 配置hosts后通过域名访问
- 获取root初始密码，24小时后失效，需及时修改密码

```shell
docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password
```

## 更多配置

### IP地址登录

```yaml
version: '3.6'
services:
  web:
    image: 'gitlab/gitlab-ce:latest'   # 默认最新镜像，亦可指定版本
    container_name: gitlab
    restart: unless-stopped
    privileged: true
    hostname: 'gitlab'
    environment:
      TZ: 'Asia/Shanghai'
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://192.168.17.200'  #项 本机IP地址
        gitlab_rails['time_zone'] = 'Asia/Shanghai'
        gitlab_rails['gitlab_shell_ssh_port'] = 2222
    ports:
      - '80:80'      #默认80 自定义主机端口时需不同修改external_url
      - '443:443'
      - '2222:22'    
    volumes:
      - '/data/gitlab/config:/etc/gitlab'
      - '/data/gitlab/data:/var/opt/gitlab'
      - '/data/gitlab/logs:/var/log/gitlab'
```

### 自定义端口

```yaml
version: '3.6'
services:
  web:
    image: 'gitlab/gitlab-ce:latest'   # 默认最新镜像，亦可指定版本
    container_name: gitlab
    restart: unless-stopped
    privileged: true
    hostname: 'gitlab.l2c.com'
    environment:
      TZ: 'Asia/Shanghai'
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://gitlab.l2c.com:8888'  #项目clone地址,需与hostname一致，默认80端口
        gitlab_rails['time_zone'] = 'Asia/Shanghai'
        gitlab_rails['gitlab_shell_ssh_port'] = 2222
    ports:
      - '8888:8888'      #自定义端口
      - '443:443'
      - '2222:22'    
    volumes:
      - '/data/gitlab/config:/etc/gitlab'
      - '/data/gitlab/data:/var/opt/gitlab'
      - '/data/gitlab/logs:/var/log/gitlab'
```

### 配置邮箱+logging

> 参考自：[gitlab配置邮件服务（QQ邮箱）图解教程](https://blog.csdn.net/qq_38966361/article/details/90543377)

启用邮箱后需手动修改管理员默认邮箱

#### 在docker-compose编排文件中配置

```yaml
version: '3.6'
services:
  web:
    image: 'gitlab/gitlab-ce:latest'   # 默认最新镜像，亦可指定版本
    container_name: gitlab
    restart: unless-stopped
    privileged: true
    hostname: 'gitlab.l2c.com'
    environment:
      TZ: 'Asia/Shanghai'
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://gitlab.l2c.com'  #项目clone地址,需与hostname一致，默认80端口
        gitlab_rails['time_zone'] = 'Asia/Shanghai'
        gitlab_rails['gitlab_shell_ssh_port'] = 2222
        gitlab_rails['smtp_enable'] = true
        gitlab_rails['smtp_address'] = "smtp.qq.com"
        gitlab_rails['smtp_port'] = 465
        gitlab_rails['smtp_user_name'] = "xxxxxxxx@qq.com"  # 用自己的邮箱
        gitlab_rails['smtp_password'] = "xxxxxxxx"   # 开启smtp后生成的授权码
        gitlab_rails['smtp_domain'] = "smtp.qq.com"
        gitlab_rails['smtp_authentication'] = "login"
        gitlab_rails['smtp_enable_starttls_auto'] = true
        gitlab_rails['smtp_tls'] = true
        gitlab_rails['gitlab_email_from'] = 'xxxxxxxx@qq.com'
        alertmanager['admin_email'] = 'admin@example.com'   # 设置管理员邮箱
    ports:
      - '80:80'      #默认80 自定义主机端口时需不同修改external_url
      - '443:443'
      - '2222:22'
    volumes:
      - '/data/gitlab/config:/etc/gitlab'
      - '/data/gitlab/data:/var/opt/gitlab'
      - '/data/gitlab/logs:/var/log/gitlab'
    logging:
      driver: "json-file"
      options:
        max-size: "200m"
        max-file: "10"
```

```shell
docker-compose up -d
```

#### 在持久化文件中配置

```shell
vim /data/gitlab/config/gitlab.rb
# 编辑邮箱配置
```

- 进入容器，重载配置

```shell
# 进入gitlab容器
docker exec -it gitlab /bin/bash

# 重载配置
gitlab-ctl reconfigure
```

- 测试

```shell
# 进入控制台(需要一点时间)
gitlab-rails console

# 发送测试邮件
Notify.test_email('xxxxxxxx@qq.com','gitlab邮箱测试','gitlab邮箱测试').deliver_now
```

### 配置SSL

#### 生成HTTPS访问证书

- 创建证书存放目录

```shell
mkdir -p ./ssl 
cd ./ssl 
```

- 创建证书

```shell
openssl req  -newkey rsa:4096 -nodes -sha256 -keyout ca.key -x509 -days 3650 -out ca.crt

# 一路回车出现Common Name 输入（因为是CA，可不输入IP或域名）：L2C Cert Root CA  
# L2C为自定义名称
```

- 生成证书签名请求

```shell
openssl req  -newkey rsa:4096 -nodes -sha256 -keyout gitlab.key -out gitlab.csr

# 一路回车出现Common Name 输入IP或域名：l2c.com
```

- 新建extfile.cnf

```shell
vim extfile.cnf
```

```ruby
subjectAltName = @alt_names
extendedKeyUsage = serverAuth

[alt_names]
# 域名，如有多个用DNS.2,DNS.3…来增加
DNS.1 = l2c.com
DNS.2 = *.l2c.com
# IP地址, 服务器的ip
IP.1 = 192.168.17.200
IP.2 = 127.0.0.1
```

- 生成证书

```shell
openssl x509 -req -days 3650 -in gitlab.csr -CA ca.crt -CAkey ca.key -CAcreateserial -extfile extfile.cnf -out gitlab.crt
```

- 分发证书至其它节点

```shell
scp ca.crt root@[IP]:/etc/pki/ca-trust/source/anchors/
```

- 更新证书配置

```shell
update-ca-trust extract
```

#### 配置HTTPS

- 将证书拷贝至持久化目录

```shell
mkdir -p /data/gitlab/config/trusted-certs
cp gitlab.crt gitlab.key /data/gitlab/config/trusted-certs
```

- 配置

```shell
version: '3.6'
services:
  web:
    image: 'gitlab/gitlab-ce:latest'   # 默认最新镜像，亦可指定版本
    container_name: gitlab
    restart: unless-stopped
    privileged: true
    hostname: 'gitlab.l2c.com'
    environment:
      TZ: 'Asia/Shanghai'
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://gitlab.l2c.com'  #项目clone地址,需与hostname一致，默认80端口
        nginx['redirect_http_to_https']= true   #取消注释改为 true
        nginx['ssl_certificate'] = "/etc/gitlab/trusted-certs/gitlab.crt"    #放置对应的证书密钥
        nginx['ssl_certificate_key'] = "/etc/gitlab/trusted-certs/gitlab.key" #放置对应的证书密钥
        gitlab_rails['time_zone'] = 'Asia/Shanghai'
        gitlab_rails['gitlab_shell_ssh_port'] = 2222
        gitlab_rails['smtp_enable'] = true
        gitlab_rails['smtp_address'] = "smtp.qq.com"
        gitlab_rails['smtp_port'] = 465
        gitlab_rails['smtp_user_name'] = "xxxxxxxx@qq.com"  # 用自己的邮箱
        gitlab_rails['smtp_password'] = "xxxxxxxx"   # 开启smtp后生成的授权码
        gitlab_rails['smtp_domain'] = "qq.com"
        gitlab_rails['smtp_authentication'] = "login"
        gitlab_rails['smtp_enable_starttls_auto'] = true
        gitlab_rails['smtp_tls'] = true
        gitlab_rails['gitlab_email_from'] = 'xxxxxxxx@qq.com'
    ports:
      - '80:80'      #默认80 自定义主机端口时需不同修改external_url
      - '443:443'
      - '2222:22'
    volumes:
      - '/data/gitlab/config:/etc/gitlab'
      - '/data/gitlab/data:/var/opt/gitlab'
      - '/data/gitlab/logs:/var/log/gitlab'
    logging:
      driver: "json-file"
      options:
        max-size: "200m"
        max-file: "10"
```

## 重置root密码

## git使用

### 配置全局用户名和邮箱

```shell
# 查看配置
git config --list

git config --global user.name "xxx"
git config --global user.email xxxxxxxx@xxx.com
```

### 配置用户名和密码

```shell
# 启用store模式，输入密码后，会以明文形式默认保存于~/.git-credentials
git config --global credential.helper store
```

