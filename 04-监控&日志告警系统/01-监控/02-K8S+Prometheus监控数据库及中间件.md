# K8S+Prometheus监控数据库及中间件

> **Prometheus部署及监控告警配置参考上文**：[k8s之部署Prometheus监控平台并实现监控告警](https://blog.csdn.net/qq_14999375/article/details/119180043)

[toc]

## 概述

本文采用helm安装数据库&中间件的exporter,并通过配置alertmanager及告警规则监控各组件的状态，并实现邮件报警。其中所采用的helm仓库及chart包如下所示：

- helm仓库：

```shell
prometheus-community: https://prometheus-community.github.io/helm-charts
```

- chart包：

> 下载无反应可尝试重试多次

```shell
prometheus-community/prometheus-mysql-exporter
prometheus-community/prometheus-redis-exporter
prometheus-community/prometheus-kafka-exporter
prometheus-community/prometheus-rabbitmq-exporter
```

## 监控Mysql

### 部署Mysql（单机版示例）

```
kubectl create ns test

vim mysql-deploy.yaml
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: test
spec:
  selector: 
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: root@mysql
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: test
spec:
  type: ClusterIP
  ports:
  - port: 3306
    targetPort: 3306
  selector:
    app: mysql
```

```
kubectl apply -f mysql-deploy.yaml
```

- 最终信息如下

```
host: mysql.test
port: 3306
user: root
pass: root@mysql
```

### 部署mysql-exporter

#### 下载并解压mysql-exporter安装包

```
cd  ~/workspace/prometheus/
helm pull prometheus-community/prometheus-mysql-exporter
tar zvcf [xxx.tgz]
```

#### 配置values.yaml

```shell
cd  ~/workspace/prometheus/prometheus-mysql-exporter
vim values.yaml
```

#### 设置mysql连接

> 参考上节中mysql的连接信息

```
mysql:
  db: ""
  host: "mysql.test"
  param: ""
  pass: "root@mysql"
  port: 3306
  protocol: ""
  user: "root"
```

![image-20210814133022142](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210814134200.png)

#### 部署mysql-exporter

```
helm install prometheus-mysql-exporter -n prometheus .
```

==多实例监控==：部署多个exporter即可（注意区分helm-NAME）

- 在prometheus-server面板中查看Target

![image-20210814133357079](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210814134158.png)

- 查看mysql-exporter采集的信息

![image-20210814133552661](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210814134154.png)

### 配置Grafana-Dashboard

- 导入[MySQL Overview](https://grafana.com/grafana/dashboards/7362)监控面板：7362

![image-20210814134146099](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210814134151.png)

### 告警规则

告警规则可以参考该监控面板配置，示例如下：

#### Mysql状态 

```sql
mysql_up == 0
```

#### 打开文件数量偏高 

```sql
mysql_global_status_innodb_num_open_files / mysql_global_variables_open_files_limit > 0.75
```

#### 当前连接数超过最大限制的75%

```sql
max_over_time(mysql_global_status_threads_connected[5m]) / mysql_global_variables_max_connections > 0.75
```

#### 历史最大连接数超过最大限制的75% 

```sql
mysql_global_status_max_used_connections / mysql_global_variables_max_connections > 0.75
```

#### 慢查询过多 

```sql
rate(mysql_global_status_slow_queries[5m])>3
```

## 监控Redis

### 部署Redis（单机版示例）

> 可参考之前的文章：[k8s之安装单点Redis+NFS持久化+数据迁移](https://blog.csdn.net/qq_14999375/article/details/114841936)

```
vim redis-deploy.yaml
```

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis
  namespace: test
data:
  redis.conf: |+
    requirepass redis@passwd
    maxmemory 268435456
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: test
  labels:
    app: redis
spec:
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
      annotations:
        version/date: "20210814"
        version/author: "lc"
    spec:
      containers:
      - name: redis
        image: redis
        imagePullPolicy: Always
        command: ["redis-server","/etc/redis/redis.conf"]
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: redis-config
          mountPath: /etc/redis/redis.conf
          subPath: redis.conf
      volumes:
      - name: redis-config
        configMap:
          name: redis
          items:
          - key: redis.conf
            path: redis.conf
---
kind: Service
apiVersion: v1
metadata:
  name: redis
  namespace: test
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
```

```
kubectl apply -f redis-deploy.yaml
```

- 最终连接信息如下

```
redisAddress: redis.test:6379
redisPassword: redis@passwd
```

### 部署redis-exporter

#### 下载并解压redis-exporter安装包

```
cd  ~/workspace/prometheus/
helm pull prometheus-community/prometheus-redis-exporter
tar zvcf [xxx.tgz]
```

#### 配置values.yaml

```
oliver006/redis_exporter:v1.27.0
```

```shell
cd  ~/workspace/prometheus/prometheus-redis-exporter
vim values.yaml
```

#### 设置redis连接

- 参考上节中mysql的连接信息，并打开密码认证（在最下方）

```
redisAddress: redis.test:6379
---
auth:
  # Use password authentication
  enabled: true
  # Use existing secret (ignores redisPassword)
  secret:
    name: ""
    key: ""
  # Redis password (when not stored in a secret)
  redisPassword: "redis@passwd"
```

#### 设置exporter的Target

> 此处相比原模版改动较多，请注意差异

```
annotations: 
  prometheus.io/path: "/metrics"
  prometheus.io/port: "9121"
  prometheus.io/scrape: "true"
labels: {}
```

![image-20210814151531262](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210814151531.png)

#### 部署redis-exporter

```
helm install prometheus-redis-exporter -n prometheus .
```

- 在prometheus-server面板中查看Target

![image-20210814151637143](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210814151637.png)

- 查看redis-exporter采集的信息

![image-20210814151714038](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210814151714.png)

### 配置Grafana-Dashboard

- 导入[Redis Dashboard](https://grafana.com/grafana/dashboards/11835)监控面板：11835

![image-20210814152833053](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210814152833.png)

### 告警规则

告警规则可以参考该监控面板配置，示例如下：

#### Redis状态

```
redis_up == 0
```

#### 内存不足

```
redis_memory_used_bytes/redis_memory_max_bytes * 100 > 80
```

#### 连接过多

```
redis_connected_clients > 100
```

#### 连接不足

```
redis_connected_clients < 5
```

#### 连接被拒绝

```
increase(redis_rejected_connections_total[1m]) > 0
```

## 监控Kafka

### kafka集群部署

> 请参考之前的文章部署：[k8s之部署kafka集群+高可用配置](https://blog.csdn.net/qq_14999375/article/details/118145483)

- 依旧部署至test空间，最终连接信息如下

```
kafkaServer: kafka.test:9092
```

### 部署kafka-exporter

#### 下载并解压kafka-exporter安装包

```
cd  ~/workspace/prometheus/
helm pull prometheus-community/prometheus-kafka-exporter
tar zvcf [xxx.tgz]
```

#### 配置values.yaml

```
danielqsj/kafka-exporter:v1.3.1
```

```shell
cd  ~/workspace/prometheus/prometheus-kafka-exporter
vim values.yaml
```

#### 设置kafka连接

```
kafkaServer:
  - kafka.test:9092
```

#### 设置exporter的Target

- 在annotations下添加：

```
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/path: "/metrics"
  prometheus.io/port: "9308"
```

![image-20210815161540033](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210815161540.png)

#### 部署kafka-exporter

```
helm install prometheus-kafka-exporter -n prometheus .
```

- 在prometheus-server面板中查看Target

![image-20210815161629525](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210815161629.png)

- 查看kafka-exporter采集的信息

![image-20210815161652655](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210815161652.png)

#### 采集更多的数据

- 如上图，exporter当前采集的数据较少，后文配置dashborad时将无法显示数据

- 通过指定`--consumer-property`激活消费者配置，使的exporter采集到更多的数据

```
# 进图pod
kubectl exec -it -n test kafka-0 -- bash

# 创建topic
kafka-topics.sh --zookeeper zookeeper:2181 --topic test001  --create --partitions 3 --replication-factor 2

# 生产topic（出现角标后，随意几行数据）
kafka-console-producer.sh --broker-list kafka:9092 --topic test001

# 消费topic（指定--consumer-property）
kafka-console-consumer.sh --bootstrap-server kafka:9092 --from-beginning --topic test001 --consumer-property group.id=test
```

![image-20210815164254348](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210815164254.png)

### 配置Grafana-Dashboard

- 导入[Kafka Exporter Overview](https://grafana.com/grafana/dashboards/7589)监控面板：7589

![image-20210815164904825](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210815164904.png)

### 告警规则

告警规则可以参考该监控面板配置，示例如下：

#### kafka节点状态

```
kafka_brokers < 3
```

#### kafka消息产生数量

```
sum(round(delta(kafka_topic_partition_current_offset[5m])/5)) by (topic) > 100
```

#### kafka消息消费数量

```
sum(round(delta(kafka_consumergroup_current_offset[5m])/5)) by (topic) > 100
```

#### 消费滞后

```
sum(kafka_consumergroup_lag) by (consumergroup, topic) 
```

## 监控RabbitMQ

### RabbitMQ集群部署

> 请参考之前的文章部署：[K8S之部署RabbitMQ集群+镜像模式实现高可用](https://blog.csdn.net/qq_14999375/article/details/119085363)

- 依旧部署至test空间，最终连接信息如下（rabbitmq-management）

```
rabbitmq:
  url: http://rabbitmq.test:15672
  user: admin
  password: admin@mq
```

### 部署rabbitmq-exporter

#### 下载并解压rabbitmq-exporter安装包

```
cd  ~/workspace/prometheus/
helm pull prometheus-community/prometheus-rabbitmq-exporter
tar zvcf [xxx.tgz]
```

#### 配置values.yaml

```
kbudde/rabbitmq-exporter:v0.29.0
```

```shell
cd  ~/workspace/prometheus/prometheus-rabbitmq-exporter
vim values.yaml
```

#### 设置rabbitmq连接

```
rabbitmq:
  url: http://rabbitmq.test:15672
  user: admin
  password: admin@mq
```

#### 设置exporter的Target

- 打开annotations下的注释（端口要加引号）：

```
annotations: 
  prometheus.io/scrape: "true"
  prometheus.io/path: "/metrics"
  prometheus.io/port: "9419"
```

![image-20210815170604958](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210815170605.png)

#### 部署rabbitmq-exporter

```
helm install prometheus-rabbitmq-exporter -n prometheus .
```

- 在prometheus-server面板中查看Target

![image-20210815170652046](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210815170652.png)

- 查看rabbitmq-exporter采集的信息

![image-20210815170745284](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210815170745.png)

### 配置Grafana-Dashboard

- 导入[RabbitMQ Monitoring](https://grafana.com/grafana/dashboards/4279)监控面板：4279
- 导入[RabbitMQ Metrics](https://grafana.com/grafana/dashboards/4371)监控面板：4371

![image-20210815171037876](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210815171038.png)

### 告警规则

告警规则可以参考该监控面板配置，示例如下：

#### 节点状态

```
sum by (node) (rabbitmq_running) == 0
```

#### 内存

超过500M即提示，数值参考历史状态

```
sum by (node) (round(rabbitmq_node_mem_used /1024 /1024 )) > 500
```

#### 文件描述符

数值参考历史状态

```
sum by (node) (rabbitmq_fd_used) > 100
```

#### 网络

```
rabbitmq_sockets_used < 0
```

#### 无队列消费

```
rabbitmq_consumersTotal < 0
```

#### 可消费消息数

```
increase (rabbitmq_queue_messages_ready_total[1m])
```

#### 未确认消息数

```
increase (rabbitmq_queue_messages_unacknowledged_total[1m])
```

