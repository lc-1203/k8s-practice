# k8s之安装单点Redis+NFS持久化

## 主机清单

|    角色    |    私网IP    |
| :--------: | :----------: |
| k8s-master | 172.16.62.20 |
| k8s-node1  | 172.16.62.21 |
| k8s-node2  | 172.16.62.22 |

![image-20210315155200913](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210315155201.png)

## 创建NFS持久化目录

### 安装NFS

- 在k8s集群中的每个节点安装nfs-utils

```
yum install nfs-utils -y
```

- 选择一台机器创建共享总目录(为了方便，这里选择主节点)

```
mkdir -p /data/nfs
```

- 编辑配置 `vim /etc/exports`

```
/data/nfs *(rw,no_root_squash)
```

![image-20210315155359262](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210315155359.png)

- 重启服务并验证

```
# 使配置生效
systemctl restart nfs &&
systemctl enable nfs

# 查看
showmount -e
```

![image-20210315155432882](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210315155432.png)

- 为需要持久化的服务创建子目录（必须创建）

```
mkdir -p /data/nfs/redis
```

### 创建PV&PVC

- pv不用指定命名空间
- pvc需要指定命名空间，默认为default
- 若有配置hosts映射，可使用映射名代替

```
# vim 1-pv_pvc.yaml
----------------------------------------------

apiVersion: v1
kind: PersistentVolume
metadata:
  name: redis-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: redis-nfs
  nfs:
    path: /data/nfs/redis
    server: 172.16.62.20
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: redis-nfs
```

- 安装

```
kubectl apply -f 1-pv_pvc.yaml
```

![image-20210315155755833](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210315155755.png)

## K8S部署

### ConfigMap

- 设置密码：`requirepass 123456`

```
vim 2-configmap.yaml
-----------------------------------

apiVersion: v1
kind: ConfigMap
metadata:
  name: redis
data:
  redis.conf: |+
    requirepass 123456
    protected-mode no
    port 6379
    tcp-backlog 511
    timeout 0
    tcp-keepalive 300
    daemonize no
    supervised no
    pidfile /var/run/redis_6379.pid
    loglevel notice
    logfile ""
    databases 16
    always-show-logo yes
    save 900 1
    save 300 10
    save 60 10000
    stop-writes-on-bgsave-error yes
    rdbcompression yes
    rdbchecksum yes
    dbfilename dump.rdb
    dir /data
    slave-serve-stale-data yes
    slave-read-only yes
    repl-diskless-sync no
    repl-diskless-sync-delay 5
    repl-disable-tcp-nodelay no
    slave-priority 100
    lazyfree-lazy-eviction no
    lazyfree-lazy-expire no
    lazyfree-lazy-server-del no
    slave-lazy-flush no
    appendonly no
    appendfilename "appendonly.aof"
    appendfsync everysec
    no-appendfsync-on-rewrite no
    auto-aof-rewrite-percentage 100
    auto-aof-rewrite-min-size 64mb
    aof-load-truncated yes
    aof-use-rdb-preamble no
    lua-time-limit 5000
    slowlog-log-slower-than 10000
    slowlog-max-len 128
    latency-monitor-threshold 0
    notify-keyspace-events Ex
    hash-max-ziplist-entries 512
    hash-max-ziplist-value 64
    list-max-ziplist-size -2
    list-compress-depth 0
    set-max-intset-entries 512
    zset-max-ziplist-entries 128
    zset-max-ziplist-value 64
    hll-sparse-max-bytes 3000
    activerehashing yes
    client-output-buffer-limit normal 0 0 0
    client-output-buffer-limit slave 256mb 64mb 60
    client-output-buffer-limit pubsub 32mb 8mb 60
    hz 10
    aof-rewrite-incremental-fsync yes
```

### Deployment

- ConfigMap生成的配置文件放置于容器内`/etc/redis/redis.conf`

- 使挂载的ConfigMap生效：`command: ["redis-server","/etc/redis/redis.conf"]`
- 将容器的`/data`持久化到`redis-pvc`，即172.16.62.22机器的`/data/nfs/redis`下

```
vim 3-deployment.yaml
---------------------------------

apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  labels:
    app: redis
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
      annotations:
        version/date: "20210310"
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
        - name: redis-persistent-storage
          mountPath: /data
      volumes:
      - name: redis-config
        configMap:
          name: redis
          items:
          - key: redis.conf
            path: redis.conf
      - name: redis-persistent-storage
        persistentVolumeClaim:
          claimName: redis-pvc
```

### Service

- 通过NodePort方式暴露服务，注意是否开启了指定nodeport（30379）的访问权限

```
vim 4-service.yaml
---------------------------------

kind: Service
apiVersion: v1
metadata:
  name: redis
spec:
  type: NodePort
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
    nodePort: 30379
```

### 运行

- 上述yaml文件可写在一个文件内，用`---`分隔

- 一键部署：`kubectl apply -f .`

  ![image-20210315160410810](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210315160410.png)

- 查询运行情况

![](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210315160736.png)

- 测试：Redis Desktop工具登录：

  - 连接地址(集群任一节点的外网IP)：8.142.35.50:30379，密码：123456

  ![image-20210315161010892](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210315161010.png)

### 持久化测试

- 添加新键

![image-20210315164903190](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210315164903.png)

- 查看键值

![image-20210315165442783](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210315165442.png)

- 删除pod

![image-20210315165253685](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210315165253.png)

- 自动新建pod后，查询键值是否存在

![image-20210315170120574](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210315170120.png)

- 查询是持久化目录是否生成文件

![image-20210315170450829](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210315170450.png)



## 数据迁移

### 数据备份

- 若源redis配置有持久化，直接拷贝持久化目录下的dump.rdb

- 若源redis不支持持久化，则进入容器生成dump.rdb并拷出
  - 进入容器：`kubectl exec -it redis-xxx /bin/bash`

  - 进入redis命令台：`redis-cli`

  - 密码认证：`auth password`

  - 保存数据：`save`

  - 退出redis命令台：`quit`
  - 退出容器：`exit`

  - 从容器中取出数据：`kubectl cp -n namespace Pod_Name:/data/dump.rdb ./`
  - 传输至远程主机：`scp dump.rdb root@目标服务器:/目录`

### 数据还原

- 停止redis,直接删除创建的deployment

- 拷贝dump.rdb至目标redis的持久化目录下（将覆盖原文件）

- 重启pod：`kubectl apply -f 3-deployment.yaml`

  

![image-20210315171235281](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210315171235.png)

- 查询键值，可发现源redis的数据已全部重现

![image-20210315171704451](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210315171704.png)