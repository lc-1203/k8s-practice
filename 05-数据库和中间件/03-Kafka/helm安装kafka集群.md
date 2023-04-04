# k8s之安装kafka集群+高可用配置

## 安装helm

### 安装helm

- helm项目地址：https://github.com/helm/helm

- 安装（可选择最新版本的helm）：

```shell
# 下载
wget https://get.helm.sh/helm-v3.6.1-linux-amd64.tar.gz

# 解压
tar zxvf helm-v3.6.1-linux-amd64.tar.gz

# 安装
mv linux-amd64/helm /usr/local/bin/

# 验证
helm version
```

### Helm基本命令

```shell
# 添加仓库
helm repo add 

# 查询 charts
helm search repo

# 更新repo仓库资源
helm repo update

# 查看当前安装的charts
helm list

# 安装
helm install

# 卸载
helm uninstall

# 更新
helm upgrade
```

## 部署zookeeper与kafka

### 下载chart包

```shell
# 添加bitnami仓库
helm repo add bitnami https://charts.bitnami.com/bitnami

# 查询chart
helm search repo bitnami

# 创建工作目录
mkdir ~/kafka
cd ~/kafka

# 拉取zookeeper和kafka
helm pull bitnami/kafka
helm pull bitnami/zookeeper

# 解压
tar zxvf [kafka/zookeeper] 
```

![image-20210623085007428](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210623085007.png)

### 部署zookeeper

#### 编辑values配置

- 进入zookeeper工作目录：`cd ~/kafka/zookeeper`，配置时区、持久化存储、副本数等

> 建议首次部署时直接修改values中的配置，而不是用--set的方式，这样方便后期upgrade。

```shell
vim values.yaml
```

```shell
# 镜像
bitnami/zookeeper:3.7.0-debian-10-r127
```

```yaml
# 手动添加时区
# 该模版无extraEnvVars字段，直接复制以下三行
# 部署后，可通过查看logs验证时区是否生效
extraEnvVars: 
  - name: TZ
    value: "Asia/Shanghai"


# 允许任意用户连接（默认开启）
allowAnonymousLogin: true
---

# 关闭认证（默认关闭）
auth:
  enable: false   
---

# 修改副本数
replicaCount: 3 
---

# 4. 配置持久化，按需使用
persistence:
  enabled: true
  storageClass: "rook-ceph-block"  # storageClass
  accessModes:
    - ReadWriteOnce
```

#### 安装

```shell
cd ~/zookeeper
kubectl create ns kafka
helm install zookeeper -n kafka .
```

- 查看zookeeper节点状态

```shell
# 进入pod
kubectl exec -it -n kafka zookeeper-0 -- bash

# 查看状态
zkServer.sh status
```

### 安装kafka

#### 编辑values配置

- 进入kafka工作目录：`cd ~/kafka/kafka`，配置zookeeper、持久化存储、副本数、时区、默认分区数、默认副本数、日志过期时间等等

```
vim values.yaml
```

```shell
# 镜像
bitnami/kafka:2.8.0-debian-10-r84
```

#### 基础配置

- 时区、副本数、持久化存储、zookeeper连接

```yaml
# 在extraEnvVars下添加时区
extraEnvVars: 
  - name: TZ
    value: "Asia/Shanghai"
---

# 副本数
replicaCount: 3                    # 副本数
---

# 持久化存储
persistence:
  enabled: true
  storageClass: "rook-ceph-block"  # sc
  accessModes:
    - ReadWriteOnce
  size: 8Gi
---

# 配置zookeeper外部连接
zookeeper:
  enabled: false                   # 不使用内部zookeeper

externalZookeeper:                 # 外部zookeeper
  servers: zookeeper               
```

#### 高可用配置

- 默认分区数、默认副本数、日志过期时间。（需根据kafka节点数设定）

```yaml
## 允许删除topic（按需开启）
deleteTopicEnable: true

## 日志保留时间（默认一周）
logRetentionHours: 168

## 自动创建topic时的默认副本数
defaultReplicationFactor: 2

## 用于配置offset记录的topic的partition的副本个数
offsetsTopicReplicationFactor: 2

## 事务主题的复制因子
transactionStateLogReplicationFactor: 2

## min.insync.replicas
transactionStateLogMinIsr: 2

## 新建Topic时默认的分区数
numPartitions: 3
```

![image-20210815141616563](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210815141616.png)

#### 安装

```
cd ~/kafka
helm install kafka -n kafka .
```

## 调试

### 查询、创建

- 进入pod

```shell
kubectl exec -it -n kafka kafka-0 -- bash
```

- 创建topic（3分区+2副本）

```shell
kafka-topics.sh --zookeeper zookeeper:2181 --topic test001  --create --partitions 3 --replication-factor 2
```

- 列出topic；查询topic详情

```shell
kafka-topics.sh --list --zookeeper zookeeper:2181
kafka-topics.sh --zookeeper zookeeper:2181 --describe --topic test001

4. 查询group
kafka-consumer-groups.sh --bootstrap-server kafka:9092 --list
kafka-consumer-groups.sh --describe --bootstrap-server kafka:9092 --group [group]
```

### 生产、消费

- 生产

```shell
kafka-console-producer.sh --broker-list kafka:9092 --topic test001
```

- 消费（另开一个窗口进入pod）

```shell
kafka-console-consumer.sh --bootstrap-server kafka:9092 --from-beginning --topic event.home.mac  --consumer-property group.id=cmd_test

# 查看消费组
kafka-consumer-groups.sh --bootstrap-server kafka:9092 --list

# 查看组中消费者状态
kafka-consumer-groups.sh --bootstrap-server kafka:9092 --group scene_algorithm --describe 
```

### 修改、删除

- 修改topic配置：增加分区至4个（分区只可增不可减），并调整日志保留时间为10天（需换算为毫秒）

```shell
kafka-topics.sh --zookeeper zookeeper:2181 --alter --config retention.ms=864000000 --partitions 4 --topic test001 
```

- 删除topic（需要设置deleteTopicEnable: true）

```shell
kafka-topics.sh --delete --zookeeper zookeeper:2181 --topic test001 
```

### 维护须知

- 若zookeeper所有节点宕机/卸载后重新部署，zookeeper恢复后还需要重新启动kafka集群。



## Kafka-Eagle

### system-config.properties

```
######################################
# multi zookeeper & kafka cluster list
######################################
kafka.eagle.zk.cluster.alias=cluster
cluster.zk.list=zookeeper:2181

######################################
# broker size online list
######################################
cluster.kafka.eagle.broker.size=20

######################################
# zk client thread limit
######################################
kafka.zk.limit.size=25

######################################
# kafka eagle webui port
######################################
kafka.eagle.webui.port=8048

######################################
# kafka offset storage
######################################
cluster.kafka.eagle.offset.storage=kafka

######################################
# kafka metrics, 30 days by default
######################################
kafka.eagle.metrics.charts=false
kafka.eagle.metrics.retain=30

######################################
# kafka sql topic records max
######################################
kafka.eagle.sql.topic.records.max=5000
kafka.eagle.sql.fix.error=false

######################################
# delete kafka topic token
######################################
kafka.eagle.topic.token=keadmin

######################################
# kafka sasl authenticate
######################################
cluster.kafka.eagle.sasl.enable=false
cluster.kafka.eagle.sasl.protocol=SASL_PLAINTEXT
cluster.kafka.eagle.sasl.mechanism=SCRAM-SHA-256
# 密码长度有限制
cluster.kafka.eagle.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="xxxxx";
cluster.kafka.eagle.sasl.client.id=
cluster.kafka.eagle.sasl.cgroup.enable=false
cluster.kafka.eagle.sasl.cgroup.topics=

######################################
# kafka sqlite jdbc driver address
######################################
#kafka.eagle.driver=org.sqlite.JDBC
#kafka.eagle.url=jdbc:sqlite:/hadoop/kafka-eagle/db/ke.db
#kafka.eagle.username=root
#kafka.eagle.password=www.kafka-eagle.org

######################################
# kafka mysql jdbc driver address
######################################
kafka.eagle.driver=com.mysql.jdbc.Driver
kafka.eagle.url=jdbc:mysql://xxx:3306/ke?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull
kafka.eagle.username=xxx
kafka.eagle.password=xxx
```

```
kubectl create configmap kafka-eagle-config -n kafka --from-file=kafka_client_jaas.conf –-from-file=system-config.properties
```

### kafka_client_jaas.conf

```
KafkaClient {
  org.apache.kafka.common.security.plain.PlainLoginModule required
  username="admin"
  password="admin-secret";
};
```

```
kubectl create configmap kafka-eagle-config -n public --from-file=kafka_client_jaas.conf --from-file=system-config.properties
```

### Deploy

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-eagle
  namespace: public
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      workload.user.cattle.io/workloadselector: deployment-kafka-kafka-eagle
  template:
    metadata:
      labels:
        workload.user.cattle.io/workloadselector: deployment-kafka-kafka-eagle
      annotations:
        version/date: "2021071304"
        version/author: "licong"        
    spec:
      containers:
      - image: buzhiyun/kafka-eagle:latest
        imagePullPolicy: Always
        name: kafka-eagle
        ports:
        - containerPort: 8048
          name: 8048tcp01
          protocol: TCP
        resources: {}
        securityContext:
          allowPrivilegeEscalation: false
          privileged: false
          procMount: Default
          readOnlyRootFilesystem: false
          runAsNonRoot: false
        stdin: true
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        tty: true
        volumeMounts:
        - mountPath: /opt/kafka-eagle/conf
          name: conf
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - configMap:
          defaultMode: 256
          name: kafka-eagle-config
          optional: false
        name: conf
---
apiVersion: v1
kind: Service
metadata:
  name: kafka-eagle-client
  namespace: public
spec:
  type: ClusterIP
  ports:
    - port: 8048
      targetPort: 8048
  selector:
    workload.user.cattle.io/workloadselector: deployment-kafka-kafka-eagle
```

