# docker-compose部署Nginx

docker挂载文件时会覆盖掉容器里面的目录，因此需准备一份默认的配置文件

## 准备配置文件

- 临时启动nginx容器，拷出默认配置文件

```shell
# 启动nginx
docker run -d --name tmp-nginx nginx:latest

# 拷出默认配置文件
mkdir -p /root/nginx/{config,data,logs}
docker cp tmp-nginx:/etc/nginx /root/nginx/config/ 
docker cp tmp-nginx:/usr/share/nginx/html /root/nginx/data/
docker cp tmp-nginx:/var/log/nginx /root/nginx/logs/ 
```

- 删除临时容器

```shell
docker rm -f tmp-nginx
```

## docker方式启动

- 持久化目录须为绝对路径

```shell
docker run -d --name nginx -p 80:80 \
-v /root/nginx/config/nginx/:/etc/nginx \
-v /root/nginx/data/html:/usr/share/nginx/html \
-v /root/nginx/logs/:/var/log/nginx \
nginx:latest
```

- 测试，修改欢迎页面

```
cd /root/nginx/data/html
echo "hello nginx" > index.html
```

- 访问

![image-20220311093150528](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20220311093150.png)

## docker-compose方式启动

- 进入工作目录`/root/nginx`，创建docker-compose编排文件（挂载目录一致）

```shell
vim docker-compose.yml
```

```yaml
version: '3'
services:
  nginx:
    image: nginx         # 镜像
    container_name: nginx # 容器名
    restart: always      # 开机自动重启
    ports:               # 端口号绑定（宿主机:容器内）
      - 80:80
    volumes:             # 目录映射（宿主机:容器内）
      - ./config/nginx/:/etc/nginx
      - ./data/html:/usr/share/nginx/html
      - ./logs/:/var/log/nginx
```

- 启动

```shell
docker-compose up -d
```

- 停止

```shell
docker-compose down
```

- 若修改配置，需重新创建容器

```shell
docker-compose up -d --force-recreate
```

