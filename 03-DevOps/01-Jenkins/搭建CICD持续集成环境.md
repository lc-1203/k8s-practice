# 搭建CICD持续集成环境

[toc]

## 环境介绍

### 容器环境

|               平台                |  版本   | 安装方式 |
| :-------------------------------: | :-----: | :------: |
| [docker](https://www.docker.com/) | 20.10.5 |   yum    |
|          docker-compose           | 1.28.6  |  二进制  |
|                K8S                |         |  Ansibe  |

### 持续集成环境

|                     平台                     |  版本   |    安装方式    |           端口           |        密码         |
| :------------------------------------------: | :-----: | :------------: | :----------------------: | :-----------------: |
|     [Gitlab](https://about.gitlab.com/)      | 13.7.4  | docker-compose | 9999<br />8443<br />2222 | root<br />Gl@!2021  |
| [Harbor](https://github.com/goharbor/harbor) |  2.1.4  |   离线安装包   |       80<br />443        | admin<br />Hb@!2021 |
|      [Jenkins](https://www.jenkins.io/)      | 2.263.2 | docker-compose |     8888<br />50000      | admin<br />Jk@!2021 |
|                  SonarQube                   |   8.6   | docker-compose |           9000           | admin<br />Sq@!2021 |

## Docker 

### docker

1. 移除电脑上原有的dockers

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

2. 安装docker-ce

```shell
yum install -y yum-utils
 
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
 
yum install docker-ce
```

3. 配置阿里云镜像库

```shell
mkdir -p /etc/docker
 
tee /etc/docker/daemon.json <<-'EOF'
{
"registry-mirrors": ["https://ohdpuoqu.mirror.aliyuncs.com"]
}
EOF
```

4. 设置开启启动

```
systemctl daemon-reload &&
systemctl restart docker &&
systemctl enable docker &&
systemctl status docker
```

### docker-compose

1. 下载[docker-compose](https://github.com/docker/compose/releases)

![image-20210324144910802](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210324144910.png)

2. 拷贝至远程主机，重命名，赋予权限，移动至可执行目录下

![image-20210324145243654](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210324145243.png)

## Gitlab

### docker-compose部署

> [Gitlab安装帮助文档](https://docs.gitlab.com/omnibus/docker/)

```
--- vim docker-compose.yml ---

version: '3'
services:
    web:
      image: 'gitlab/gitlab-ce:latest'
      restart: always
      container_name: gitlab
      hostname: '172.16.62.10'
      environment:
        GITLAB_OMNIBUS_CONFIG: |
          external_url 'http://39.103.171.99:9999' #项目clone地址
      ports:
        - '9999:9999'  #默认80 自定义时主机IP需要容器IP一致
        - '8443:443'
        - '2222:22'
      volumes:
        - '/$PWD/gitlab/config:/etc/gitlab'
        - '/$PWD/gitlab/data:/var/opt/gitlab'
        - '/$PWD/gitlab/logs:/var/log/gitlab'
      logging:
          driver: "json-file"
          options:
              max-size: "200m"
              max-file: "10"
```

### 部署安装

- 后台部署：`docker-compose up -d`
- 状态查看：`docker ps -a`，显示`healthy`即安装完成，约2分钟

- 登录：
  - URL：http://39.103.171.99:9999
  - 首次登录需要修改密码
  - 语言：管理员--设置--偏好设置

![image-20210324155949435](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210324155949.png)

## Harbor

### 准备事项

- 环境：docker、docker-compose、https

> [官方安装包下载地址](https://github.com/goharbor/harbor/releases)

![image-20210324163804178](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210324163804.png)

### 配置HTTPS访问证书

- 创建证书存放目录

```
mkdir -p ./ssl 
cd ./ssl 
```

- 创建证书

```
openssl req  -newkey rsa:4096 -nodes -sha256 -keyout ca.key -x509 -days 3650 -out ca.crt

# 一路回车出现Common Name 输入（因为是CA，可不输入IP或域名）：Chenry Cert Root CA  
# Chenry ：自定义名称
```

- 生成证书签名请求

```
openssl req  -newkey rsa:4096 -nodes -sha256 -keyout harbor.key -out harbor.csr

# 一路回车出现Common Name 输入IP或域名：chenry.cn
```

- 新建extfile.cnf

```
--- vim extfile.cnf  --- 

subjectAltName = @alt_names
extendedKeyUsage = serverAuth

[alt_names]
# 域名，如有多个用DNS.2,DNS.3…来增加
DNS.1 = chenry.cn
DNS.2 = *.chenry.cn
# IP地址, 服务器的ip
IP.1 = 172.16.62.10
IP.2 = 127.0.0.1
```

- 生成证书

```
openssl x509 -req -days 3650 -in harbor.csr -CA ca.crt -CAkey ca.key -CAcreateserial -extfile extfile.cnf -out harbor.crt
```

- 分发证书至其它节点

```
scp ca.crt root@[IP]:/etc/pki/ca-trust/source/anchors/
```

- 更新证书配置，重启docker

```
update-ca-trust extract &&
systemctl restart docker
```

### 安装Harbor

- 解压：`tar zxf [文件名]`
- 配置harbor.yml.tmpl

![image-20210119184959082](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210324170900.png)

- 生成harbor.yml文件：`cp harbor.yml.tmpl harbor.yml`
- 执行：

```
./prepare
./install.sh
```

## SonarQube

### docker-compose.yml

```
=== vim docker-compose.yml ===

version: '3'
services: 
  postgres: 
    image: postgres    #基础镜像名字
    restart: always      #在容器退出时总是重启容器
    container_name: sonarqube_postgres    #自定义sonarqube容器名字
    ports:
      - 5433:5432   #-宿主机端口:容器暴露端口 ； 即将容器暴露端口映射到宿主机上某个端口
    volumes:
      - ./data/postgres/postgresql:/var/lib/postgresql    #将容器目录文件映射到宿主机上
      - ./data/postgres/data:/var/lib/postgresql/data
    environment:
      TZ: Asia/Shanghai    #指定容器时区
      POSTGRES_USER: sonar               #自定义postgres数据库登录用户
      POSTGRES_PASSWORD: sonar    #自定义postgres数据库登录密码
      POSTGRES_DB: sonar                    #自定义postgres数据库 “库”
    networks: 
      - sonar-network  #使用文件末定义的网络模式
  sonarqube:
    image: sonarqube    # 8.6
    restart: always 
    container_name: sonarqube_master
    ports:
      - 9001:9000
    volumes:
      - ./data/sonarqube/extensions:/opt/sonarqube/extensions
      - ./data/sonarqube/logs:/opt/sonarqube/logs
      - ./data/sonarqube/data:/opt/sonarqube/data
      - ./data/sonarqube/conf:/opt/sonarqube/conf
    environment:
      SONARQUBE_JDBC_USERNAME: sonar    #指定postgres数据库连接信息
      SONARQUBE_JDBC_PASSWORD: sonar
      SONARQUBE_JDBC_URL: jdbc:postgresql://postgres:5432/sonar
    depends_on:    #指定依赖某个服务 （本实例sonarqube依赖postgres）
      - postgres
    networks: 
      - sonar-network
networks:
  sonar-network:
    driver: bridge   #定义网络模式为bridge (缺省值bridge; 支持bridge，host，none，container四种类型)
```

### 配置宿主机并运行

```
1、修改宿主机内核参数
echo "vm.max_map_count = 655360"  >> /etc/sysctl.conf
sysctl -p

2、赋权给sonarqube/data目录下所有目录、文件
mkdir -p  ./data/sonarqube/data
chmod  -R 777 ./data/sonarqube/data

3、运行SonarQube(8+) + postgres
docker-compose up -d   #启动容器
docker ps #查看容器状态
```

### 安装中文包

#### 方式1：搜索插件安装

- 管理员帐号登录
- 配置--应用市场--**Chinese Pack**

![image-20210323101240152](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210325141232.png)

- 下载完成后重启

![image-20210323101329695](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210325141232.png)

#### 方式2：手动安装插件

- 去官网下载指定版本的jar包

![image-20210323094659757](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210325141232.png)

- 拷贝至持久化目录
- 重启服务

## Jenkins

### 预装环境

> [Linux下JDK和Maven的安装及配置](https://blog.csdn.net/ula_liu/article/details/80853713)

- [JDK8](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
- [maven](http://maven.apache.org/download.cgi)
- 配置hosts：容器内使用直接读取的宿主机hosts，而jenkins是调用容器自身hosts

### docker-compose部署

- 新建docker-compose.yml

```
--- vim docker-compose.yml ---

version: '3'
services:
  jenkins:
    restart: always
    image: jenkins/jenkins:lts
    container_name: jenkins
    extra_hosts:
      - "chenry.cn:172.16.62.10"
    environment:
      - JAVA_OPTS:'-Djava.util.logging.config.file=/var/jenkins_home/log.properties'
      - TZ=Asia/Shanghai
    ports:
      - '8888:8080'
      - '50000:50000'
    privileged: true
    user: root
    restart: always
    volumes:
      - ./$PWD/data/jenkins_home:/var/jenkins_home
      - ./$PWD/data/jenkins_home/.m2:/var/jenkins_home/.m2
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/bin/docker:/usr/bin/docker
      - /usr/lib/x86_64-linux-gnu/libltdl.so.7:/usr/lib/x86_64-linux-gnu/libltdl.so.7      
      - /usr/local/apache-maven-3.6.3:/usr/local/apache-maven-3.6.3
      - /usr/local/jdk1.8.0_281:/var/local/jdk1.8.0_281
```

- 后台运行：`docker-compose up -d`
- 查看日志获取密钥：`docker logs -f jenkins`

### 安装插件

- 进入插件安装页面：`Jenkins --> Manage Jenkins --> Manager Plugin --> Available`

```
---   选择插件   ---

Localization: Chinese (Simplified)
Pipeline
Config File Provider
Extended Choice Paremeter
Git & Git Parameter
SonarQube Scanner for Jenkins
Kubernetes
Kubernetes Continuous Deploy
```

## CICD环境配置

### 添加平台认证

| 平台      | 类型                         |
| --------- | ---------------------------- |
| Gitlab    | 凭据 - username and password |
| Harbot    | 凭据 - username and password |
| SonarQube | 凭据 - secret text           |
| K8S       | cfg - custom file            |

### 系统配置

### 全局工具配置

### 建立流水线



```
def git_url = "http://39.103.171.99:9999/ai/helloJenkins.git"

// 认证
def harbor_auth = "b1916130-d69a-4f27-8569-89111681e070"
def git_auth = "3ece2fb4-7649-435e-8f56-2304529caf05"

pipeline {
  agent any
  // 参数化构建
  parameters {
      choice(choices: ['dev-platform'], description: '命名空间', name: 'K8S_NAMESPACE')
      choice(choices: ['test'], description: 'Maven打包环境', name: 'MAVEN_RESOURCES')
      choice(choices: ['harbor.chenry.cn'], description: 'Harbor仓库地址', name: 'HARBOR_HOST')
	  choice(choices: ['8080'], description: '容器暴露端口', name: 'EXPOSE_PORT')
      string(name: 'REPLICAS', defaultValue: '1', description: '部署pod个数')
      string(name: 'ROLLBACK', defaultValue: 'FALSE', description: '回滚编号')
    }

  // 生成环境变量

  stages {
    // Step1： 拉取代码 
    stage('GitSCM') {
      steps {
        checkout([$class: 'GitSCM', 
                  branches: [[name: '*/dev']], 
                  doGenerateSubmoduleConfigurations: false, 
                  extensions: [], submoduleCfg: [], 
                  userRemoteConfigs: [[credentialsId: "${git_auth}", url: "${git_url}"]]])
      }                 
    }  
    // Step2： 根据指定的Maven环境打包
    stage('Maven Build') {
      when { expression { params.ROLLBACK == "FALSE" } }
      steps {
          withSonarQubeEnv('sonarqube') {
            sh "mvn clean package -DskipTests=true -U -P ${MAVEN_RESOURCES}"
            }
      }
    }
  }
}

Harbor APIV2.0 删除镜像操作

# 1. 获取项目/仓库下的镜像信息
image_info=$(curl -s -k -u ${username}:${password} -X GET "https://${HARBOR_HOST}/api/v2.0/projects/proj-1/repositories/lc-boot-helloworld-dev-platform/artifacts?page=1&page_size=10&with_tag=true&with_label=false&with_scan_overview=false&with_signature=false&with_immutable_status=false" -H "accept: application/json")

# 2. 提取出镜像tag
tags="$(echo "$image_info" | tr , '\n' | grep name | cut -d "\"" -f4)"

# 3. 镜像tag排序，取出最近10个以外的镜像tag
for tag in `echo ${tags} | awk 'BEGIN{i=1}{gsub(/ /,"\\n");i++;print}' | awk -F. '{print $NF}' | sort -nr | sed "1,10d"`;do
curl -s -k -u 'admin':'Hb@!2021' -X DELETE https://${HARBOR_HOST}/api/v2.0/projects/proj-1/repositories/lc-boot-helloworld-dev-platform/artifacts/${tag}
done
```

- Harbor APIV2.0 删除镜像操作
  - 仓库名不可包含`/`
  - 流水线语法需要+`\`转义

```
# 1. 获取项目/仓库下的镜像信息
image_info=$(curl -s -k -u ${username}:${password} -X GET "https://${HARBOR_HOST}/api/v2.0/projects/proj-1/repositories/lc-boot-helloworld-dev-platform/artifacts?page=1&page_size=10&with_tag=true&with_label=false&with_scan_overview=false&with_signature=false&with_immutable_status=false" -H "accept: application/json")

# 2. 提取出镜像tag
tags="$(echo "$image_info" | tr , '\\n' | grep name | cut -d "\\"" -f4)"

# 3. 镜像tag排序，取出最近10个以外的镜像tag
for tag in `echo ${tags} | awk 'BEGIN{i=1}{gsub(/ /,"\\n");i++;print}' | awk -F. '{print $NF}' | sort -nr | sed "1,3d"`;do
curl -s -k -u 'admin':'Hb@!2021' -X DELETE https://${HARBOR_HOST}/api/v2.0/projects/proj-1/repositories/lc-boot-helloworld-dev-platform/artifacts/${tag}
done
```

```
# 1. 获取项目/仓库下的镜像信息
image_info=$(curl -s -k -u ${username}:${password} -X GET "https://${HARBOR_HOST}/api/v2.0/projects/proj-1/repositories/lc-boot-helloworld-dev-platform/artifacts?page=1&page_size=10&with_tag=true&with_label=false&with_scan_overview=false&with_signature=false&with_immutable_status=false" -H "accept: application/json")

# 2. 提取出镜像tag
tags="$(echo "$image_info" | tr , '\n' | grep name | cut -d '\"' -f4)"

# 3. 镜像tag排序，取出最近10个以外的镜像tag
for tag in `echo ${tags} | awk 'BEGIN{i=1}{gsub(/ /,"\\n");i++;print}' | awk -F. '{print $NF}' | sort -nr | sed "1,3d"`;do
curl -s -k -u 'admin':'Hb@!2021' -X DELETE https://${HARBOR_HOST}/api/v2.0/projects/proj-1/repositories/lc-boot-helloworld-dev-platform/artifacts/${tag}
done

```



Harbor APIV2.0 删除镜像操作

- 仓库名不可包含`/`
- pipeline流水线需要+`\`转义

```
# 1. 获取项目/仓库下的镜像信息
image_info=$(curl -s -k -u ${username}:${password} -X GET "https://${HARBOR_HOST}/api/v2.0/projects/${project_name}/repositories/${repo_name}/artifacts?page=1&page_size=10&with_tag=true&with_label=false&with_scan_overview=false&with_signature=false&with_immutable_status=false" -H "accept: application/json")

# 2. 提取出镜像tag
tags="$(echo "$image_info" | tr , '\n' | grep name | cut -d '\"' -f4)"

# 3. 镜像tag排序，取出最近10个以外的镜像tag
for tag in `echo ${tags} | awk 'BEGIN{i=1}{gsub(/ /,"\\n");i++;print}' | awk -F. '{print $NF}' | sort -nr | sed "1,3d"`;do
curl -s -k -u ${username}:${password} -X DELETE https://${HARBOR_HOST}/api/v2.0/projects/${project_name}/repositories/${repo_name}/artifacts/${tag}
done
```

