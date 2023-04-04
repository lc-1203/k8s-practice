## docker-compose安装sonar

> 参考自：[jenkins流水线+sonar检查多模块maven项目](https://www.jianshu.com/p/1a4b8bdf12f8)

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
    image: sonarqube    #8.5.1 
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

![image-20210323101240152](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210323101240.png)

- 下载完成后重启

![image-20210323101329695](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210323101329.png)

#### 方式2：手动安装插件

- 去官网下载指定版本的jar包

![image-20210323094659757](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210323094659.png)

- 拷贝至持久化目录
- 重启服务

## 持续集成配置

### sonar后台生成连接密钥

- 我的帐号--安全--生成令牌

![](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210323094911.png)

### Jenkins--凭据

- 添加凭据（Secret text类型）

![image-20210323095134257](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210323095134.png)

### Jenkins--sonar插件安装

- SonarQube Scanner for Jenkins

![image-20210323095217902](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210323095217.png)

### Jenkins--sonar连接属性配置

- 系统管理--系统配置--SonarQube servers

![image-20210323095352041](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210323095352.png)

### Jenkins--流水线实现

#### 方式1：流水线包括所有参数

```
stage('Code Analysis') {
    steps {
        withSonarQubeEnv('sonarqube') {
            sh "mvn clean package -DskipTests=true -U -P ${MAVEN_RESOURCES} sonar:sonar -Dsonar.sorceEncoding=UTF-8 -Dsonar.sources=src/main -Dsonar.java.binaries=target/classes"
        }
    }
}
```

#### 方式2：sonar-project.properties

- 将这些分析参数保存在**项目根目录**下的一个`sonar-project.properties`文件中

```
sonar.sourceEncoding=UTF-8
sonar.sources=src/main
sonar.tests=src/test/java
sonar.java.binaries=target/classes
```

- 再在流水线中传入这个配置文件

```
stage('Code Analysis') {
    steps {
        withSonarQubeEnv('sonarqube') {
            sh "mvn clean package sonar:sonar -Dproject.settings=sonar-project.properties"
        }
    }
}
```

### MAVEN本地实现

```
mvn clean package sonar:sonar -Dsonar.sorceEncoding=UTF-8 -Dsonar.sources=src/main -Dsonar.java.binaries=target/classes 
```

