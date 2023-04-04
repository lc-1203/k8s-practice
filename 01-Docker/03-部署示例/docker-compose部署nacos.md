## Nacos Docker

项目地址：[nacos-docker](https://github.com/nacos-group/nacos-docker)

部署模式：**standalone-mysql-8**

## 端口排查

```shell
# NACOS: 8848 9848 9555 未占用
# MYSQL: 3306 妥善起见，不使用默认端口
netstat -tunlp |grep 3306
```

- 更改mysql默认端口（编辑docker-compose.yml文件对外暴露端口即可）

> 无需修改env/nacos-standlone-mysql.env中配置端口

## 部署

### 项目目录

- 工作目录：/root/lcSpace/nacos

- ./init.d/custom.properties：nacos初始化参数
- ./env/{mysql.env,nacos-standlone-mysql.env}: docker-compose 环境变量文件
- ./docker-compose.yml：编排文件

```shell
# 工作目录：/root/lcSpace/nacos
./init.d/custom.properties：nacos初始化参数
./env/{mysql.env,nacos-standlone-mysql.env}: docker-compose 环境变量文件
./docker-compose.yml：编排文件
```

### docker-compose部署

- docker-compose.yml

```yaml
version: "2"
services:
  nacos:
    image: nacos/nacos-server
    container_name: nacos-standalone-mysql
    env_file:
      - ./env/nacos-standlone-mysql.env
    volumes:
      - ./standalone-logs/:/home/nacos/logs
      - ./init.d/custom.properties:/home/nacos/init.d/custom.properties
    ports:
      - "8848:8848"
      - "9848:9848"
      - "9555:9555"
    depends_on:
      - mysql
    restart: always
  mysql:
    container_name: mysql
    image: nacos/nacos-mysql:8.0.16
    env_file:
      - ./env/mysql.env
    volumes:
      - ./mysql:/var/lib/mysql
    ports:
      - "30306:3306"
```

```shell
docker-compose up -d
```

## 使用

### 管理界面

- 连接地址

```
http://10.10.10.100:8848/nacos
```

- 帐号

```
nacos:nacos
```

### 数据库

- 连接地址：

```
10.10.10.1008:30306
```

- 帐号

```
root:root
nacos:nacos
```

