# Nginx+Consul全容器化动态负载均衡实践

[toc]

## nginx-upsync镜像制作


### 下载nginx及所需模块

* 版本
|组件|版本|
|:----|:----|
|nginx|1.21.6|
|nginx-upsync-module|weibocom/2.1.3|
|nginx_upstream_check_module|xiaokai-wang/1.12.1+|

* 准备工作目录
```bash
mkdir -p ~/nginx-upsync/module && cd ~/nginx-upsync/module
```
* 下载
```bash
# nginx-1.21.6
wget http://nginx.org/download/nginx-1.21.6.tar.gz

# nginx-upsync-module-2.1.3
wget https://github.com/weibocom/nginx-upsync-module/archive/refs/tags/v2.1.3.tar.gz

# nginx_upstream_check_module
wget https://codeload.github.com/xiaokai-wang/nginx_upstream_check_module/zip/master
```
* 解压
```bash
tar -zxvf nginx-1.21.6.tar.gz
tar -zxvf v2.1.3.tar.gz
yum install -y unzip && unzip master
```
### 导出nginx编译信息及持久化配置

* 启动临时容器，采用同版本的nginx镜像
```bash
docker run -it --rm --name tmp-nginx nginx:1.21.6 bash
```
* 查看configure环境，并拷贝至下阶段的dockerfile中
```bash
nginx -V
```
* 暂时不要退出此临时容器，等下阶段烤出配置文件后再退出
* docker挂载文件时会覆盖掉容器里面的目录，因此需准备一份默认的配置文件
```bash
cd ~/nginx-upsync
docker cp tmp-nginx:/etc/nginx ./config/ 
docker cp tmp-nginx:/usr/share/nginx/html ./data/
docker cp tmp-nginx:/var/log/nginx ./logs/ 
```
* 退出临时容器
### dockerfile多阶段构建

由于采用多阶段构建方式，因此一些指令没有做合并

* Dockerfile
```dockerfile
# 编译阶段，安装相应的编译环境
FROM nginx:1.21.6 as build
# 替换为阿里源
RUN sed -i 's/deb.debian.org/mirrors.aliyun.com/g' /etc/apt/sources.list  \
    && sed -i 's/security.debian.org/mirrors.aliyun.com/g' /etc/apt/sources.list \
    && apt clean  && apt-get update -y
# 安装编译环境
RUN apt -y install gcc make libssl-dev zlib1g-dev patch libpcre3-dev

WORKDIR /module/
# 拷贝所需安装的模块
ADD ./module/*.* ./
COPY ./module/master ./

# 安装upstream_check模块最新补丁
RUN apt install unzip && unzip master

WORKDIR /module/nginx-1.21.6/
RUN patch -p1 < ../nginx_upstream_check_module-master/check_1.12.1+.patch

# 编译：在原始配置上加上upsync和upstream_check模块
RUN ./configure --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-g -O2 -ffile-prefix-map=/data/builder/debuild/nginx-1.21.6/debian/debuild-base/nginx-1.21.6=. -fstack-protector-strong -Wformat -Werror=format-security -Wp,-D_FORTIFY_SOURCE=2 -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -Wl,--as-needed -pie' --add-module=../nginx-upsync-module-2.1.3 --add-module=../nginx_upstream_check_module-master
# 安装
RUN make && make install

# 制作镜像，拷入上阶段产生的编译文件
FROM nginx:1.21.6-alpine
COPY --from=build /usr/sbin/nginx /usr/sbin/nginx构建
```
* 制作镜像
```bash
docker build -t nginx:1.21.6-upsync .
```

## 部署consul

- docker-compose.yml

```yaml
version: '3'
services:
  consul:
    image: consul:1.12.0
    container_name: consul
    command: agent -server -bootstrap-expect=1 -node=consul -bind=0.0.0.0 -client=0.0.0.0 -datacenter=dc1 -ui
    ports:
    - 8500:8500
    volumes:
      - ./data:/consul/data  ##保存数据目录
      - ./conf:/consul/config  ##保存配置目录
```
* 启动
```bash
docker-compose up -d
```

## 部署nginx-upsync


### 配置负载均衡，集成consul

* 进入默认配置目录，新增consul负载均衡配置
```bash
cd ~/nginx-upsync/config/conf.d
mkdir consul && touch ./consul/prj_demo_backend.conf
vim upsync.conf
```
* 监听88端口，转发到后端。如果需要通过80端口负载，删除默认default.conf即可

```yaml
upstream prj_demo_backend {
        # 添加一条负载，负载自身，否则nginx无法启动
        server 10.0.0.10:88;
        # 监听consul的配置中心
        upsync 10.0.0.10:8500/v1/kv/upstreams/prj_demo_backend upsync_timeout=6m upsync_interval=500ms upsync_type=consul strong_dependency=off;
        #根据consul把配置文件生成到的prj_whoami_backend文件中
        upsync_dump_path /etc/nginx/conf.d/consul/prj_demo_backend.conf;
        # 引入配置文件
        include /etc/nginx/conf.d/consul/prj_demo_backend.conf;
        check interval=1000 rise=2 fall=2 timeout=3000 type=http default_down=false;
        check_http_send "HEAD / HTTP/1.0\r\n\r\n";
        check_http_expect_alive http_2xx http_3xx;
}

server {
       listen 88;  # 监听88端口
       index index.html index.php;
       location / {
          proxy_set_header HOST $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header Client-IP $remote_addr;
          proxy_set_header X-Fprwarded-For $proxy_add_x_forwarded_for;
          proxy_pass http://prj_demo_backend;
       }
       # 健康检查 - 查看负载均衡的列表
       location /upstream_list {
           upstream_show;
       }
       # 健康检查 - 查看负载均衡的状态
       location /upstream_status {
           check_status;
           access_log off;
       }
}
```
### docker-compose启动

* docker-compose.yml
```yaml
version: '3'
services:
  nginx-upsync:
    image: nginx:1.21.6-upsync    # 镜像
    container_name: nginx-upsync # 容器名
    restart: always      # 开机自动重启
    ports:               # 端口号绑定（宿主机:容器内）
      - 9080:80
      - 9088:88
    volumes:             # 目录映射（宿主机:容器内）
      - ./config:/etc/nginx
      - ./data:/usr/share/nginx/html
      - ./logs:/var/log/nginx
```
* 启动
```bash
docker-compose up -d 
```

## 动态负载均衡测试


### 启动模拟后端

- 通过whoami模拟后端

```bash
docker run -d -p 9504:80 --name whoami-1 traefik/whoami
docker run -d -p 9505:80 --name whoami-2 traefik/whoami
```
### consul注册服务

```bash
curl -X PUT -d '{"weight":1,"max_fails":2,"fail_timeout":10}' http://10.0.0.10:8500/v1/kv/upstreams/prj_demo_backend/10.0.0.10:9504
curl -X PUT -d '{"weight":1,"max_fails":2,"fail_timeout":10}' http://10.0.0.10:8500/v1/kv/upstreams/prj_demo_backend/10.0.0.10:9505
```
* 访问，两台后端轮流负载
```bash
while true; do curl -s 127.0.0.1:83  | grep Hostname; sleep 1;done
```
<img src="https://lc-cs.oss-cn-shenzhen.aliyuncs.com/img/202205311011241.png" alt="img" style="zoom:50%;" />


### 测试健康检查

* 停掉一台后端
```bash
docker stop whoami-1
```
- 再次访问，仅一台后端负载

<img src="https://lc-cs.oss-cn-shenzhen.aliyuncs.com/img/202205311011043.png" alt="img" style="zoom:50%;" />


* 查看后端列表及状态
```bash
curl 127.0.0.1:9088/upstream_list
curl 127.0.0.1:9088/upstream_status
```
