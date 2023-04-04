# k8s之helm部署mariadb-galera集群+maxscale读写分离+压测

[toc]

## 安装helm

### 安装helm

- 项目地址：https://github.com/helm/helm

- 安装：

```
- 下载
wget https://get.helm.sh/helm-v3.6.1-linux-amd64.tar.gz

- 解压
tar zxvf helm-v3.6.1-linux-amd64.tar.gz

- 安装
mv linux-amd64/helm /usr/local/bin/

- 验证
helm version
```

### 基本命令

```
- 添加仓库
helm repo add 

- 查询 charts
helm search repo

- 更新repo仓库资源
helm repo update

- 查看当前安装的charts
helm list

- 安装
helm install

- 卸载
helm uninstall

- 更新
helm upgrade
```

## 安装mariadb-galera集群

### 下载chart包

```
- 添加bitnami仓库
helm repo add bitnami https://charts.bitnami.com/bitnami

- 查询chart
helm search repo bitnami

- 创建工作目录 
mkdir ~/mariadb
cd ~/mariadb

- 拉取chart包
helm pull bitnami/mariadb-galera

- 解压
tar zxvf [xxx.t]
```

### 安装mariadb-galera

#### 修改配置

- 进入工作目录：`cd ~/mariadb/mariadb-galera`，配置持久化存储、副本数等

```
vim values.yaml （找到相应的键值修改）
```

```
# 镜像
bitnami/mariadb-galera:10.5.12-debian-10-r1
```

```
grant all ON *.* TO dev_rw@'localhost' identified by 'e0f2b9746a38a76e' with grant option;
grant all ON *.* TO dev_rw@'%' identified by 'e0f2b9746a38a76e' with grant option;
grant all ON *.* TO maxuser@'localhost' identified by 'maxscale@Db' with grant option;
grant all ON *.* TO maxuser@'%' identified by 'maxscale@Db' with grant option;
grant all ON *.* TO exporter@'localhost' identified by 'exporter@mariadb' with grant option;
grant all ON *.* TO exporter@'%' identified by 'exporter@mariadb' with grant option;


GRANT ALL ON phpmyadmin.* TO 'read'@'%' WITH GRANT OPTION;
```



```yaml
# 设置时区
extraEnvVars:                  
  - name: TZ
    value: "Asia/Shanghai"
---

# 设置root帐号密码
rootUser:
  user: root
  password: "root@Db"           
---
  
# 添加初始化sql脚本
initdbScripts:
  init.sql: |
    grant all ON *.* TO maxuser@'localhost' identified by 'maxscale@Db' with grant option;
    grant all ON *.* TO maxuser@'%' identified by 'maxscale@Db' with grant option;
    grant all ON *.* TO exporter@'localhost' identified by 'exporter@mariadb' with grant option;
    grant all ON *.* TO exporter@'%' identified by 'exporter@mariadb' with grant option;
    grant all ON test.* TO testrw@'localhost' identified by 'testrw' with grant option;
    grant all ON test.* TO testrw@'%' identified by 'testrw' with grant option;
    grant all ON phpmyadmin.* TO 'testrw'@'%' with grant option;
    flush privileges;

  
# 设置副本数  
replicaCount: 3                 
---

# 资源限制，按需配置
resources:                     
  limits: 
     cpu: 1
     memory: 1024Mi
  requests: 
     cpu: 0.5
     memory: 512Mi
---

# 持久化存储，本文使用ceph，亦可使用nfs-client-provisioner
persistence:
  enabled: true
  mountPath: /bitnami/mariadb
  selector: {}
  storageClass: "rook-ceph-block"    
  annotations:
  accessModes:
    - ReadWriteOnce
  size: 20Gi
```

#### 安装

> 安装时，建议取名为mariadb-galera，否则会命名为[xxx]-mariadb-galera，想自定义需修改Chart.yaml中name字段。

```
cd ~/mariadb/mariadb-galera
kubectl create ns mariadb
helm install mariadb-galera -n mariadb .
```

#### 查看集群状态

```
- 进入pod
kubectl exec -it -n mariadb mariadb-galera-0 -- bash

- 登录mysql
mysql -uroot -p

- 列出集群所有参数
show status like '%wsrep%';

- 集群节点个数
show status like "wsrep_cluster_size";

- 集群的目前状态：PRIMARY（正常）/NON_PRIMARY（不一致）
show status like "wsrep_cluster_status";

- 集群已经提交事务数目，所有节点应该相等
show status like '%wsrep_last_committed%';

- 读节点执行写集的线程个数
show global variables like 'wsrep_slave_threads';

- 这个值高于0.0，说明发生同步延迟，将会引起限流
show global status like 'wsrep_local_recv_queue_avg';
```

## 维护须知

**请参考最新官网文档**

### 集群宕机重启

- 当mariadb集群全宕机时，集群重启时无法选举出主节点，因此需将手动第一个节点设置为主节点再启动

![image-20210623201016483](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210623201016.png)

1. 设置主节点

```
vim values.yaml
# 配置galera
galera:
  name: galera
  bootstrap:     
    bootstrapFromNode: mariadb-galera-0     # 需与pod名称一致
    forceSafeToBootstrap: true              # 集群全部宕机无法正常重启时开启
```

2. 更新项目

```
helm upgrade mariadb-galera -n mariadb .
```

3. 手动删除所有pod

```
kubectl delete pod -n mariadb mariadb-galera-0 mariadb-galera-1 mariadb-galera-2
```

4. 查看集群是否恢复

```
kubectl logs -f -n mariadb mariadb-galera-0
```

![image-20210623201331806](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210623201331.png)

5. 集群恢复后，关闭forceSafeToBootstrap，否则helm upgrade将首先重启最后一个节点，若还是手动指定第一个节点为主节点，那么第一个节点将无法加入集群

```
vim values.yaml
# 配置galera
galera:
  name: galera
  bootstrap:     
    bootstrapFromNode: 
    forceSafeToBootstrap: false              
```

6. 更新项目

```
helm upgrade mariadb-galera -n mariadb .
```

> 若仍无法重启集群，可能是因为数据不同步的问题，设置wsrep_sst_method=rsync后再次重启集群

### pod关闭异常

### 部署maxscale中间件（读写分离）

#### 部署maxscale

- 进入工作目录：`cd ~/mariadb `

```
vim maxscale.yaml
```

```
apiVersion: v1
kind: Service
metadata:
  name: maxscale
  namespace: data
  labels:
    app: maxscale
spec:
  type: ClusterIP
  ports:
  - name: maxscale-readwrite
    port: 4006
    targetPort: 4006
  selector:
    app: maxscale
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: maxscale-config
  namespace: data
  labels:
    app: maxscale
data:
  maxscale.cnf: |
    [maxscale]
    threads=auto
    admin_host=0.0.0.0

    [dbserv1]
    type=server
    address=mariadb-galera-0.mariadb-galera
    port=3306
    protocol=mariadbbackend

    [dbserv2]
    type=server
    address=mariadb-galera-1.mariadb-galera
    port=3306
    protocol=mariadbbackend

    [dbserv3]
    type=server
    address=mariadb-galera-2.mariadb-galera
    port=3306
    protocol=mariadbbackend

    [Splitter-Service]
    type=service
    router=readwritesplit
    servers=dbserv1,dbserv2,dbserv3
    user=maxuser
    password=max@Db
    max_slave_connections=100%
    max_slave_replication_lag=3600
    use_sql_variables_in=all
    
    

    [Splitter-Listener]
    type=listener
    service=Splitter-Service
    protocol=mariadbclient
    port=4006

    [Galera-Monitor]
    type=monitor
    module=galeramon
    servers=dbserv1,dbserv2,dbserv3
    user=maxuser
    password=max@Db
    disable_master_failback=true
    available_when_donor=true

  start-maxscale-instance.sh: |
    cp /etc/config-template/maxscale.cnf /etc/config-map/maxscale.cnf
    maxscale -d -U maxscale --configdir=/etc/config-map -lstdout
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: maxscale
  namespace: data
  labels:
    app: maxscale
spec:
  serviceName: maxscale
  selector:
    matchLabels:
      app: maxscale
  replicas: 3
  template:
    metadata:
      labels:
        app: maxscale
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - maxscale
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: maxscale
        image: mariadb/maxscale:2.4.4    #2.5.14 2.4.12
        command:
        - bash
        - /etc/config-template/start-maxscale-instance.sh 
        ports:
        - containerPort: 4006
        resources:
          requests:
            cpu: 0.5
            memory: 512Mi
          limits:
            cpu: 1
            memory: 1024Mi
        volumeMounts:
        - name: maxscale-config-vol
          mountPath: /etc/config-map
        - name: maxscale-config-template-vol
          mountPath: /etc/config-template
      restartPolicy: Always
      volumes:
      - name: maxscale-config-vol
        emptyDir: {}
      - name: maxscale-config-template-vol
        configMap:
          name: maxscale-config     
```

- 2.5版本会报错

![image-20210819105311711](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210819105311.png)

#### 查看状态

- 查看maxscale主机是否可用

```
# 查看状态
kubectl exec -it -n data maxscale-0 -- maxctrl list servers
```

![image-20210623143043288](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210623143043.png)

## 测试

### 部署sysbench

```
kubectl run sysbench-client --tty -i --restart='Always' --image  docker.io/severalnines/sysbench:latest
```

### 开启压测

- 准备数据

```
- 进入pod
kubectl exec -it sysbench-client -- bash

- 准备数据
sysbench \
    --db-driver=mysql \
    --time=300 \
    --threads=10 \
    --report-interval=10 \
    --mysql-host=maxscale.data \
    --mysql-port=4006 \
    --mysql-user=testrw \
    --mysql-password=testrw \
    --mysql-db=test \
    --tables=10  \
    --table_size=1000 \
    oltp_read_write --db-ps-mode=disable prepare
```

- 方案选择

```
mariadb-galera集群+maxscale：maxscale.data
mariadb-galera单节点：mariadb-galera-0.mariadb-galera.data
```

- 模式选择：

```
综合读写TPS：oltp_read_write
只读：oltp_read_only
删除：oltp_delete
更新索引字段：oltp_update_index
插入性能：oltp_insert
写入性能：oltp_write_only
```

- 运行：将prepare改为run，再次执行
- 调整资源限制，待`maxctrl list servers`就绪后再开启测试
- 清理数据：将run改为cleanup

| 方案（综合读写oltp_read_write）          | **TPS** | **QPS** | **95th** | errs |
| ---------------------------------------- | ------- | ------- | -------- | ---- |
| mariadb-galera单节点（0.5核1G）          | 99      | 1972    | 1803     | 0    |
| mariadb-galera集群 + maxscale（0.5核1G） | 109     | 2175    | 1509     | 0    |
| mariadb-galera集群 + maxscale（1核1G）   | 232     | 4642    | 720      | 0    |
| mariadb-galera集群 + maxscale（2核1G）   | 475     | 9510    | 370      | 0    |

| 方案（综合读写oltp_read_write,2核4G）    | **TPS** | **QPS** | **95th** | errs |
| ---------------------------------------- | ------- | ------- | -------- | ---- |
| mariadb-galera集群 + maxscale            | 413     | 8267    | 376      | 0    |
| mariadb-galera集群 + maxscale（discard） | 467     | 9345    | 363      | 0    |

![image-20210625192330184](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210625192330.png)

| 方案（prepare,1核2G，机械硬盘）               | 时间（秒） |
| --------------------------------------------- | ---------- |
| mariadb-galera集群 + maxscale（hdd）          | 300        |
| mariadb-galera集群 + maxscale（hdd, discard） | 325        |
| mariadb-galera集群 + maxscale（ssd）          | 270        |

| 方案（oltp_read_write,1核2G，机械硬盘）       | **TPS** | **QPS** | **95th** | errs |
| --------------------------------------------- | ------- | ------- | -------- | ---- |
| mariadb-galera集群 + maxscale（hdd）          | 153     | 3063    | 1109     | 0    |
| mariadb-galera集群 + maxscale（hdd, discard） | 140     | 2806    | 1192     | 0    |
| mariadb-galera集群 + maxscale（ssd）          | 204     | 4080    | 817      | 0    |



| 方案（oltp_read_only,1核2G）               | **TPS** | **QPS** | **95th** | errs |
| ------------------------------------------ | ------- | ------- | -------- | ---- |
| mariadb-galera集群 + maxscale              | 206     | 3290    | 694      | 0    |
| mariadb-galera集群 + maxscale（discard）20 | 207     | 3304    | 694      | 0    |
| mariadb-galera集群 + maxscale（ssd）       | 213     | 3409    | 682      | 0    |



| 方案（oltp_write_only,1核2G）            | **TPS** | **QPS** | **95th** | errs |
| ---------------------------------------- | ------- | ------- | -------- | ---- |
| mariadb-galera集群 + maxscale            | 459     | 2753    | 443      | 1    |
| mariadb-galera集群 + maxscale（discard） | 451     | 2705    | 405      | 0    |
| mariadb-galera集群 + maxscale（ssd）     | 597     | 3583    | 263      | 2    |



综合读写接oltp_read_only后会出现OOMKILLED，待节点重启后再测

![image-20210703174633498](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210703174633.png)





![image-20210719143051072](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210719143051.png)







- dev:10 1000 300

![image-20210817163343133](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210817163343.png)





```
kubectl run --generator=run-pod/v1 -i --rm --tty volpod --overrides='
{
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
        "name": "volpod"
    },
    "spec": {
        "containers": [{
            "command": [
                "cat",
                "/mnt/data/grastate.dat"
            ],
            "image": "bitnami/minideb",
            "name": "mycontainer",
            "volumeMounts": [{
                "mountPath": "/mnt",
                "name": "galeradata"
            }]
        }],
        "restartPolicy": "Never",
        "volumes": [{
            "name": "galeradata",
            "persistentVolumeClaim": {
                "claimName": "data-mariadb-galera-2"
            }
        }]
    }
}' --image="bitnami/minideb"
```

