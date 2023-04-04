# K8S集群中部署Squid正向代理

## Squid简介

 Squid Cache（简称为Squid）是HTTP代理服务器软件。Squid用途广泛，可以作为缓存服务器，可以过滤流量帮助网络安全，也可以作为代理服务器链中的一环，向上级代理转发数据或直接连接互联网。Squid的主要作用和应用场景有：

- 用来做前置的Web缓存，加快用户访问Web的速度
- 代理内网用户访问互联网资源
- 设置访问控制策略，控制用户的上网行为
- 主要支持http、ftp等应用协议

参考： <https://hub.docker.com/r/sameersbn/squid>，<https://www.cnblogs.com/ssgeek/p/12302135.html>

## 问题描述

通常公司内部服务器只有指定的节点具备出网权限，当其他节点需要访问公网时，需要通过该节点代理上网。步骤如下：

- 在公网节点上容器化部署Squid，提供HTTP正向代理
- 在其它节点上配置docker代理

## Docker方式部署

### 环境说明

本文中涉及三台服务器，仅有master节点具有公网访问能力（也可指定为其它节点）。

|    角色    |    私网IP    |   公网IP    |
| :--------: | :----------: | :---------: |
| k8s-master | 172.18.25.81 | 8.135.19.91 |
| k8s-node1  | 172.18.25.82 |             |
| k8s-node2  | 172.18.25.83 |             |

### 在公网节点[使用docker安装squid](https://github.com/sameersbn/docker-squid)

- 从docker hub下载容器

  ```
  docker pull sameersbn/squid:3.5.27-2
  ```

- 在docker中创建容器

  ```
  docker run --name squid -d --restart=always \
    --publish 3128:3128 \
    --volume /srv/docker/squid/cache:/var/spool/squid \
    sameersbn/squid:3.5.27-2
  ```

### 配置受信

- 安装htpasswd

  ```
  yum install httpd-tools
  ```

- 生成认证文件

  ```
  # squid_passwd为认证文件名，squid为账号名
  
  htpasswd -c squid_passwd squid
  # 这里输入两次密码：123456
  ```

- 将认证文件拷贝至容器

  ```
  docker cp squid_passwd squid:/etc/squid/
  ```

### [修改Squid配置文件](https://blog.csdn.net/weixin_34075551/article/details/88837934)

- 从Squid容器中导出默认配置文件

  ```
  docker cp squid:/etc/squid/squid.conf ./
  ```

- 去掉注释

  ```
  awk '/^[^#]/' squid.conf > squid-simple.conf
  ```

- 编辑配置文件

  ```
  vim squid-simple.conf
  ```

- 添加以下三行至“http_access deny all”上面（因为过滤规则是从上至下的）

  ```
  auth_param basic program /usr/lib/squid/basic_ncsa_auth /etc/squid/squid_passwd
  acl ncsa_users proxy_auth REQUIRED
  http_access allow ncsa_users
  ```

- 将配置文件导入Squid容器

  ```
  docker cp squid-simple.conf squid:/etc/squid/squid.conf
  ```

- 重启docker

  ```
  docker restart squid
  ```

- 附上完整配置文件：squid.conf

  ```
  acl localnet src 172.18.25.0/16
  acl localnet src 10.0.0.0/16
  acl localnet src 10.224.0.0/16
  
  acl SSL_ports port 443
  acl Safe_ports port 80          # http
  acl Safe_ports port 21          # ftp
  acl Safe_ports port 443         # https
  acl Safe_ports port 70          # gopher
  acl Safe_ports port 210         # wais
  acl Safe_ports port 1025-65535  # unregistered ports
  acl Safe_ports port 280         # http-mgmt
  acl Safe_ports port 488         # gss-http
  acl Safe_ports port 591         # filemaker
  acl Safe_ports port 777         # multiling http
  acl CONNECT method CONNECT
  auth_param basic program /usr/lib/squid/basic_ncsa_auth /etc/squid/squid_passwd
  acl ncsa_users proxy_auth REQUIRED
  http_access allow ncsa_users
  http_access deny !Safe_ports
  http_access deny CONNECT !SSL_ports
  http_access allow localhost manager
  http_access deny manager
  http_access allow localhost
  http_access deny all
  http_port 3128
  coredump_dir /var/spool/squid
  refresh_pattern ^ftp:           1440    20%     10080
  refresh_pattern ^gopher:        1440    0%      1440
  refresh_pattern -i (/cgi-bin/|\?) 0     0%      0
  refresh_pattern (Release|Packages(.gz)*)$      0       20%     2880
  refresh_pattern .               0       20%     4320
  ```



## K8S Deployment方式

1. 通过daemonset+nodeselector的方式将squid部署在k8s的两个外网节点上
2. 为其它节点的docker和pod配置代理

```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: proxy
  name: squid
data:
  config: |+
    acl localnet src 172.16.16.0/16
    acl localnet src 10.0.0.0/24
    acl localnet src 10.244.0.0/16
    
    acl SSL_ports port 443
    acl Safe_ports port 80
    acl Safe_ports port 443
    acl Safe_ports port 1025-65535
    acl CONNECT method CONNECT
    
    auth_param basic program /usr/lib/squid/basic_ncsa_auth /etc/squid/squid_passwd
    acl ncsa_users proxy_auth REQUIRED
    http_access allow ncsa_users  
    
    http_access deny !Safe_ports
    http_access deny CONNECT !SSL_ports
    http_access allow localhost manager
    http_access deny manager
    http_access deny to_localhost
    http_access allow localnet
    http_access allow localhost
    http_access deny all
    http_port 3128
    
    strip_query_terms off
    cache_dir ufs /var/spool/squid 100 16 256
    coredump_dir /var/spool/squid

    refresh_pattern ^ftp:        1440    20%    10080
    refresh_pattern ^gopher:    1440    0%    1440
    refresh_pattern -i (/cgi-bin/|\?) 0    0%    0
    refresh_pattern (Release|Packages(.gz)*)$      0       20%     2880
    refresh_pattern .        0    20%    4320
  squid_passwd: |+
    squid:$apr1$obHp1gH0$4x5SbGGCUuDAhiKWSZwrb/ 
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: squid
  namespace: proxy
  labels:
    app: squid
spec:
  selector:
    matchLabels:
      app: squid
  template:
    metadata:
      labels:
        app: squid
    spec:
      nodeSelector:
        deploy.type: ingress      
      containers:
      - name: squid
        image: sameersbn/squid:3.5.27-2
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3128
          hostPort: 3128
          
        volumeMounts:
        - name: squid-config
          mountPath: /etc/squid/squid.conf
          subPath: squid.conf
        - name: squid-passwd
          mountPath: /etc/squid/squid_passwd
          subPath: squid_passwd        
      volumes:
      - name: squid-config
        configMap:
          name: squid
          items:
          - key: config
            path: squid.conf
      - name: squid_passwd
        configMap:
          name: squid
          items:
          - key: squid_passwd
            path: squid_passwd      

---
kind: Service
apiVersion: v1
metadata:
  name: squid
  namespace: proxy
spec:
  selector:
    app: squid
  ports:
  - name: proxy
    port: 3128
```

## 代理上网

### 为其它节点上的docker配置代理

> [配置docker代理](https://www.cnblogs.com/ssgeek/p/12302135.html)

- 环境配置，格式：http://{帐号名}:{密码}@{公网节点的私网IP}:3128

  ```
  # mkdir -p /etc/systemd/system/docker.service.d
  # vim /etc/systemd/system/docker.service.d/http-proxy.conf
  
  [Service]
  Environment="HTTP_PROXY=http://squid:123456@172.18.25.81:3128" "HTTPS_PROXY=http://squid:123456@172.18.25.81:3128" "NO_PROXY="
  
  [Service]
  Environment="HTTP_PROXY=http://squid:squid1ot@10.0.0.252:3128" "HTTPS_PROXY=http://squid:squid1ot@10.0.0.252:3128" "NO_PROXY="
  ```

- 使配置生效并重启docker

  ```
  systemctl daemon-reload &&
  systemctl restart docker
  ```

- 拉取镜像验证

  ```
  docker pull busybox:latest
  ```

### 为k8s集群中的pod配置代理

- 在Deployment中加上env配置

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: 
  labels:
    app: 
spec:
  selector:
    matchLabels:
      app: 
  replicas: 
  template:
    metadata:
      name: 
      labels:
        app: 
    spec:
      containers:
        - name: 
          image: 
          imagePullPolicy: 
          ports:
            - containerPort: 
          env:
          - name: http_proxy
            value: http://squid:123456@172.18.25.81:3128
          - name: https_proxy
            value: http://squid:123456@172.18.25.81:3128
```



## 总结

水平有限，一些细节、问题没有考虑到；此外，部署方式还可以加以改进：

- 公网节点的squid部署可采用声明式部署的方式
- 其它节点的代理可在搭建集群时统一预先配置

