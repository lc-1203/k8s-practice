# Jira+Jenkins+Gitlab自动化交付实践

## jira&confluence 产品的使用

[atlassian官网](https://www.atlassian.com/zh)

### 概述

- Jira 可以帮助团队规划、分配、跟踪、报告和管理工作Jira 
- Software：专为敏捷开发团队中的各个成员以及其他人员所设计，从而规划、跟踪和交付世界一流的软件

### 关键术语

**事物**：指任意类型或规模的单个工作项目，它从创建到完成，全程受到跟踪。

**项目**：指按目的或使用情境划分的事务集合

**板**：项目内用于显示各种事务的组成部分，是项目内团队工作流的可视化表现形式

**工作流**：指事务从创建到完成所遍历的连续路径

**敏捷**：Jira Software 拥有专为实现敏捷开发而设计的主要功能集，其中包括 Scrum 和看板

![image](https://lc-cs.oss-cn-shenzhen.aliyuncs.com/img/202205310944768.png)

## 自动化工具链部署

### 环境

- 服务器：阿里云抢占式实例，张家口区域，
- 规格：通用网络增强型 g5ne，4 核 16G+20G 存储
- 系统：Centos7.9
- docker：docker-ce-20.10.16
- docker-compose：
- jira：8.9.1

###  mysql 驱动

下载 mysql 驱动（jira 容器使用），参照 jira 官方提示：

> [MySQL Community Downloads](https://dev.mysql.com/downloads/)

```sh
# 解压 tar 包，部署 jira 时将解压的 jar 包挂载至容器中 
tar -zxvf mysql-connector-java-8.0.29.tar.gz
```

<img src="https://lc-cs.oss-cn-shenzhen.aliyuncs.com/img/202205310955001.png" alt="image2" width="800" />

### 安装

#### 网络

- 创建网络，供容器间互相联通

```shell
docker network create -d bridge jira
```

#### mysql

安装 mysql，jira 提示最新版本仅支持 mysql57 和 mysql8，因此安装 mysql8，版本：8.0.29

- mysql 配置，按照 jira 要求，创建 my.cnf 并挂载至容器中

```
[client]
default-character-set = utf8mb4
[mysql]
default-character-set = utf8mb4
[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_bin
init_connect='SET NAMES utf8mb4'
```

- docker-compose.yml

```yaml
version: '3'
services:
  mysql:
    restart: always
    image: mysql:latest
    container_name: mysql
    environment:
      - "MYSQL_ROOT_PASSWORD=mysql@lc"
      - "TZ=Asia/Shanghai"
    networks:
      - jira
    ports:
      - 3306:3306
    volumes:
      - ./data/mysql:/var/lib/mysql
      - ./data/conf/my.cnf:/etc/my.cnf
networks:
  jira:
    external: true
```

- 创建用户及数据库，参考 [Connecting Jira applications to a database](https://confluence.atlassian.com/adminjiraserver/connecting-jira-applications-to-a-database-938846850.html) ：

```mysql
-- 创建数据库
CREATE DATABASE jiradb CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
-- 创建用户
CREATE USER 'jira' IDENTIFIED BY 'jira@db';
CREATE USER 'jira'@'localhost' IDENTIFIED BY 'jira@db';
GRANT ALL ON jiradb.* TO 'jira'@'localhost' WITH GRANT option;
GRANT ALL ON  jiradb.* TO 'jira'@'%' WITH GRANT option;
```

#### jira-software

安装 jira-software，版本：8.9.1

- docker-compose.yml

```yaml
version: '3'
services:
  jira:
    restart: always
    image: atlassian/jira-software:latest
    container_name: jira
    environment:
      - "TZ=Asia/Shanghai"
    networks:
      - jira
    ports:
      - 8080:8080
    volumes:
      - ./data:/var/atlassian/application-data/jira
      - ./mysql-connector-java-8.0.29.jar:/opt/atlassian/jira/lib/mysql-connector-java-8.0.29.jar
networks:
  jira:
    external: true
```

- 选择语言，设置：

<img src="https://lc-cs.oss-cn-shenzhen.aliyuncs.com/img/202205310951193.png" alt="image3" style="zoom:50%;" />

- 数据库连接，直接使用数据库的service名（因为容器在同一网络）

<img src="https://lc-cs.oss-cn-shenzhen.aliyuncs.com/img/202205310955002.png" alt="image4" style="zoom:50%;" />

- 生成许可证：选择 data center 版本，会生成一个月有效期的许可证

<img src="https://lc-cs.oss-cn-shenzhen.aliyuncs.com/img/202205310955003.png" alt="image5" style="zoom:50%;" />

#### gitlab

- docker-compose.yml

```yaml
version: '3.6'
services:
  gitlab:
    image: 'gitlab/gitlab-ce:latest'   # 默认最新镜像，亦可指定版本
    container_name: gitlab
    restart: unless-stopped
    privileged: true
    hostname: 'gitlab'
    environment:
      TZ: 'Asia/Shanghai'
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://121.89.208.247'  # 本机IP地址
        gitlab_rails['time_zone'] = 'Asia/Shanghai'
        gitlab_rails['gitlab_shell_ssh_port'] = 2222
    networks:
      - jira
    ports:
      - '80:80'      #默认 80 自定义主机端口时需同步修改 external_url
      - '443:443'
      - '2222:22'    
    volumes:
      - './data/gitlab/config:/etc/gitlab'
      - './data/gitlab/data:/var/opt/gitlab'
      - './data/gitlab/logs:/var/log/gitlab'
networks:
  jira:
    external: true
```

- 获取root用户初始密码（24小时有效期）

```
docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password
```

#### jenkins

- docker-compose.yml

```yaml
version: '3'
services:
  jenkins:
    restart: always
    image: jenkins/jenkins:lts
    container_name: jenkins
    extra_hosts:
      - "chenry.cn:172.16.62.10"
    environment:
      # - JAVA_OPTS:'-Djava.util.logging.config.file=/var/jenkins_home/log.properties'
      - TZ=Asia/Shanghai
    networks:
      - jira
    ports:
      - '8888:8080'
      - '50000:50000'
    privileged: true
    user: root
    restart: always
    volumes:
      - ./data/jenkins_home:/var/jenkins_home
      - ./data/jenkins_home/.m2:/var/jenkins_home/.m2
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/bin/docker:/usr/bin/docker
      - /usr/lib/x86_64-linux-gnu/libltdl.so.7:/usr/lib/x86_64-linux-gnu/libltdl.so.7      
      # - /usr/local/apache-maven-3.6.3:/usr/local/apache-maven-3.6.3
      # - /usr/local/jdk1.8.0_281:/var/local/jdk1.8.0_281
networks:
  jira:
    external: true
```

- 查看密钥

```sh
docker logs -f jenkins
```

- 安装插件：Jenkins --> Manage Jenkins --> Manager Plugin --> Available

```yaml
Localization: Chinese (Simplified)
Pipeline
Config File Provider
Extended Choice Parameter
Git
Git Parameter
Kubernetes
Kubernetes Continuous Deploy
Generic Webhook Trigger		# 生成webhoook，供jira等调用
Rebuilder
Jira plugin								# 集成jira
JIRA Pipeline Steps				# 集成的jira调用函数(如jiraEditIssue)
# SonarQube Scanner for Jenkins
# JIRA Trigger # 触发器，用于触发构建中赋值
```

- 重启jenkins

```
http://jenkins_url:port/restart
```

## Jira自动化交付实践

### jenkins与jira的对接

#### jira触发jenkins

创建流水线时，启用Generic Webhook Trigger触发器，将生成webhook，jira调用该webhook启动流水线，并传递参数

- jenkins启用Generic Webhook Trigger

<img src="https://lc-cs.oss-cn-shenzhen.aliyuncs.com/img/202205310955845.png" alt="image6" style="zoom: 50%;" />

- jira配置webhook，设置--系统--网络钩子。似乎必须使用IP，不能使用service访问（待验证）

<img src="https://lc-cs.oss-cn-shenzhen.aliyuncs.com/img/202205310956819.png" alt="image7" style="zoom: 50%;" />

- 需获取参数：

通过勾选Print contributed variables或者sh 'printenv'打印所有系统变量。

```yaml
# jira问题，系统参数，用于调用jira-api的唯一键
jira_issue_key: webhookData_issue_key

# jira项目名及key，系统参数，将对应于git的项目组，亦可使用自定义参数
jira_project: webhookData_issue_fields_project_name
jira_project_key: webhookData_issue_fields_project_key

# jira模块名，系统参数，将对应于git的模块，亦可使用自定义参数
jira_component: webhookData_issue_fields_components_0_name

# commit，自定义参数，将通过该commit拉取代码，并检查是否为master分支
commit: webhookData_issue_fields_customfield_10112

# 发布状态，自定义参数，执行后将调用jira-api，修改该状态值
status: webhookData_issue_fields_customfield_10114
```

#### jenkins回调jira

- jenkins需安装如下插件：

```yaml
Jira plugin  # 集成jira
JIRA Pipeline Steps  # 集成的jira调用函数(如jiraEditIssue)
```

- 配置jira-site（先添加Credentials），系统管理--系统配置--JIRA Steps。其中，url配置的为jira的service名（因为jira与jenkins容器在同一网络内），亦可使用IP

<img src="https://lc-cs.oss-cn-shenzhen.aliyuncs.com/img/202205310958846.png" alt="img" style="zoom:50%;" />

#### pipeline

```groovy
jira_project_key = env.webhookData_issue_fields_project_key
jira_issue_key = env.webhookData_issue_key

def change_status(key, value){
    stage('修改任务状态') {
    withEnv(['JIRA_SITE=jira']) {
      def lockIssue = [fields: [ project: [key: jira_project_key],
                                 customfield_10114: "${value}"]]
      response = jiraEditIssue idOrKey: "${key}", issue: lockIssue
    }
  }
}

pipeline {
    agent any
    stages{
        stage("Jira"){
            steps{
                println "jira_project: ${webhookData_issue_fields_project_name}"
                println "jira_component: ${webhookData_issue_fields_components_0_name}"
                println "commit: ${webhookData_issue_fields_customfield_10112}"
                println "status: ${webhookData_issue_event_type_name}"
                println "jira_issue_key: ${webhookData_issue_key}"
                change_status(jira_issue_key,"发布完成")
            }
        }
    }
}
```

### jenkins与gitlab的对接

#### jenkins集成git

- jenkins安装相应插件

```
Git
Git Parameter
```

#### pipeline

- 通过checkout检出代码
- 通过commitid获取分支代码，查询该commit是否为main/master分支
- 根据结果修改jira字段

```shell
jira_project_key = env.webhookData_issue_fields_project_key
jira_issue_key = env.webhookData_issue_key
commit = webhookData_issue_fields_customfield_10112

def git_url = "http://gitlab/${webhookData_issue_fields_project_name}/${webhookData_issue_fields_components_0_name}.git"

def change_status(key, value){
    stage('修改任务状态') {
    withEnv(['JIRA_SITE=jira']) {
      def lockIssue = [fields: [ project: [key: jira_project_key],customfield_10114: "${value}"]]
      response = jiraEditIssue idOrKey: "${key}", issue: lockIssue
    }
  }
}

def ref_chk(key, value){
    stage('检查分支') {
    withEnv(['JIRA_SITE=jira']) {
      def lockIssue = [fields: [ project: [key: jira_project_key],customfield_10200: "${value}"]]
      response = jiraEditIssue idOrKey: "${key}", issue: lockIssue
    }
  }
}

pipeline {
    agent any
    
    stages{
        stage("Jira"){
            steps{
                println "jira_project: ${webhookData_issue_fields_project_name}"
                println "jira_component: ${webhookData_issue_fields_components_0_name}"
                println "commit: ${webhookData_issue_fields_customfield_10112}"
                println "status: ${webhookData_issue_event_type_name}"
                println "jira_issue_key: ${webhookData_issue_key}"
                
                change_status(jira_issue_key,"发布完成")
            }
        }
        stage('Gitlab') {
            steps {
                checkout([$class: 'GitSCM', 
                    branches: [[name: commit ]], 
                    userRemoteConfigs: [[
                        credentialsId: "bd3503a8-9a16-4b97-b406-953a5a9340da", 
                        url: "${git_url}"]]])
                script {
                    echo "hello"    
                    sh "git --no-pager show -s --format=%H"
                    commitid = sh(returnStdout: true, script: "git --no-pager show -s --format=%H").trim()
                    ref_branch = sh(returnStdout: true, script: "git branch -r --contains ${commitid}").trim()
                    is_master = ref_branch.contains('main')
                    ref_chk(jira_issue_key, is_master.toString())
                }
            }
        }
    }
}
```

