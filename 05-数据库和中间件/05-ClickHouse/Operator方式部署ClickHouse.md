# 部署clickhouse operator

## 下载源码

- 项目地址

```shell
git clone https://github.com/Altinity/clickhouse-operator.git
```

- 镜像

```
# CRD镜像
vim ./clickhouse-operator-master/deploy/operator/clickhouse-operator-install-bundle.yaml
altinity/clickhouse-operator:0.17.0
altinity/metrics-exporter:0.17.0

# ClickHouse镜像
yandex/clickhouse-server:20.7

registry-vpc.cn-zhangjiakou.aliyuncs.com/lc-sc/clickhouse-operator:0.17.0
registry-vpc.cn-zhangjiakou.aliyuncs.com/lc-sc/metrics-exporter:0.17.0
registry-vpc.cn-zhangjiakou.aliyuncs.com/lc-sc/clickhouse-server:20.7
```

## 安装CRD

> 注：默认安装于kube-system空间

### 安装

```shell
# 1.进入工作目录
cd ./clickhouse-operator-master/deploy/operator

# 2.编辑配置文件，可自定义命名空间（卸载时需保持空间一致）
vim clickhouse-operator-install.sh

# 3.执行安装脚本
./clickhouse-operator-install.sh
```

### 卸载

- 指定OPERATOR_NAMESPACE后卸载，若不是kube-system，将会直接删除NAMESPACE，

```shell
./clickhouse-operator-delete.sh
```

## 安装ClickHouse集群

### 简单示例

- 进入工作目录

```shell
cd /clickhouse-operator-master/docs/chi-examples
```

- 部署安装

```shell
kubectl apply -f 01-simple-layout-02-1shard-2repl.yaml -n ck
```

- 更改Service

```shell
kubectl edit svc -n ck clickhouse-simple-01

# 去掉本地连接限制
# 将LoadBalancer改为Nodeport（非必须），并修改nodePort端口
```

- 查看集群状态

```shell
# 查看CRD
kubectl get ClickHouseInstallation -n ck
```

- 连接

```shell
# 连接地址：IP:NodePort
# 默认帐号密码：clickhouse_operator/clickhouse_operator_password
```

- 卸载

```shell
kubectl delete ClickHouseInstallation -n ck <NAME>
```

### 自定义安装

自定义存储，Service，集群规模，密码等

#### 自定义Service

```
spec:
  defaults:
    templates:
      serviceTemplate: svc-template
  templates:
    serviceTemplates:
      - name: svc-template
        spec:
          ports:
            - name: http
              port: 8123
              nodePort: 31023
            - name: tcp
              port: 9000
              nodePort: 31090
          type: NodePort
```

#### 自定义密码

- 生成 sha256sum的Hash值

```shell
# da14f14a91a3eea9@Dev
PASSWORD=1f197c8508eb2da2@Dev; echo "$PASSWORD"; 
echo -n "$PASSWORD" | sha256sum | tr -d '-'



# 开发帐号
clickhouse_operator: da14f14a91a3eea9@Dev
002670a5d9dcf17207b916619ab2b2c4d9b6f0aee6c7a2018cc6fb3f70138415

# 管理员帐号
admin: 1f197c8508eb2da2@Dev
f321c64f4ac462975cf96fb9b89d70d8c10ebc6f29a80b3f97d39c271e1e5d1c
```

```
apiVersion: clickhouse.altinity.com/v1
kind: ClickHouseInstallation
metadata:
  name: "settings-03"
spec:
  configuration:
    users:
      clickhouse_operator/password_sha256_hex: 002670a5d9dcf17207b916619ab2b2c4d9b6f0aee6c7a2018cc6fb3f70138415
      admin/password_sha256_hex: f321c64f4ac462975cf96fb9b89d70d8c10ebc6f29a80b3f97d39c271e1e5d1c
      admin/networks/ip: "127.0.0.1/32"
      admin/profile: default
      admin/quota: default
```

#### 自定义ClickHouseInstallation

- 参考`04-replication-zookeeper-04-medium-AWS-persistent-volume.yaml`

```
vim 00-replication-zookeeper-04-medium-AWS-persistent-volume.yaml
```

```yaml
apiVersion: "clickhouse.altinity.com/v1"
kind: "ClickHouseInstallation"
metadata:
  name: "repl-04"
spec:
  defaults:
    templates:
      serviceTemplate: svc-template
  configuration:
    zookeeper:
      nodes:
        - host: zookeeper.ck
          port: 2181
    clusters:
      - name: replcluster
        templates:
          podTemplate: clickhouse-with-volume-template
        layout:
          shardsCount: 3
          replicasCount: 2
  templates:
    serviceTemplates:
      - name: svc-template
        spec:
          ports:
            - name: http
              port: 8123
              nodePort: 31023
            - name: tcp
              port: 9000
              nodePort: 31090
          type: NodePort
    podTemplates:
      - name: clickhouse-with-volume-template
        spec:
          containers:
            - name: clickhouse-pod
              image: yandex/clickhouse-server:20.7
              volumeMounts:
                - name: data
                  mountPath: /var/lib/clickhouse
              resources:
                limits:
                  memory: '4Gi'
                  cpu: '2'
                requests:
                  memory: '1Gi'
                  cpu: '0.5'
    volumeClaimTemplates:
      - name: data
        reclaimPolicy: Retain
        spec:
          # no storageClassName - means use default storageClassName
          storageClassName: alicloud-nas
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 10Gi
```

```shell
kubectl apply -f 00-replication-zookeeper-04-medium-AWS-persistent-volume.yaml
```
