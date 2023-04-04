## Nginx代理模块

nginx设置代理分两个模块

- ngx_http_proxy_module的proxy_pass：需要在location段使用，需要提供域名或ip地址和端口外，还需要提供协议，如"http"或"https"
- ngx_stream_proxy_module的proxy_pass：只能在server段使用，可以是tcp端口，也可以是udp端口。

## 安装htpasswd

- 如果服务器上没有htpasswd命令，请安装  

```
yum install httpd
```

- 生成用户名密码


```shell
htpasswd -cm htpasswd admin  #htpasswd为文件名，admin为用户名。之后输入两次密码即可

htpasswd -bc ./webui_htpasswd admin 123456 # 一次性到位
```

- 查看加密密码

```
admin:$apr1$icokp362$hJavNBjPddwYu6oz1P52J1
```

- 随机生成密码fangshi

```
cat /dev/urandom | head -1 | md5sum | head -c 8
```

## nginx主配置

```
  nginx.conf: |+
    user  root;
    worker_processes  1;

    events {
        worker_connections  1024;
    }

    stream {
      include /etc/nginx/conf.d/*.conf;
    }
    http {
      include /etc/nginx/http/*.conf;
    }
```

## http代理+htpasswd

- 生成密码

```
# 随机生成8位字符
cat /dev/urandom | head -1 | md5sum | head -c 8 

# 方式一：生成htpasswd文件
htpasswd -bc ./flink_htpasswd admin 【随机字符】@flink

# 方式二：在线转换工具
https://tool.oschina.net/htpasswd
```

- 配置http代理

```
  http-port.conf: |+
    server {
      listen 82;
      location / {
        auth_basic "Please input password";
        auth_basic_user_file  /etc/nginx/http/flink_htpasswd;
        proxy_connect_timeout   10s;
        client_max_body_size    100m;
        proxy_pass http://flink-jobmanager.flink:8081;
      }
    }
  flink_htpasswd: |+
    admin:$apr1$g.kJ0IkA$.Qy0YyXAWNeH142C9OaS00
```

## tcp代理（无法配置密码）

```
  tcp-port.conf: |+
    upstream flink {
      hash $remote_addr consistent;
      server flink-jobmanager.flink:8081 max_fails=3 fail_timeout=30s;
    }
    server {
      listen 83;
      proxy_connect_timeout   10s;
      proxy_timeout   300s;
      proxy_pass  flink;
    }
```

## 挂载

```
        volumeMounts:
        - mountPath: /etc/nginx/conf.d
          name: tcp-port
        - mountPath: /etc/nginx/http
          name: http-port
        - mountPath: /etc/nginx/nginx.conf
          name: nginx
          subPath: nginx.conf
      volumes:
      - name: tcp-port
        configMap:
          items:
          - key: tcp-port.conf
            path: tcp-port.conf
          name: tcp-proxy
        
      - name: nginx
        configMap:
          items:
          - key: nginx.conf
            path: nginx.conf
          name: tcp-proxy
        
      - name: http-port
        configMap:
          name: tcp-proxy
          items:
          - key: http-port.conf
            path: http-port.conf
          - key: flink_htpasswd
            path: flink_htpasswd
```

