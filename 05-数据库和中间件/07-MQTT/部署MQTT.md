> [从零开始建立 EMQ X MQTT 服务器 的 K8S 集群](https://blog.csdn.net/emqx_broker/article/details/106790687)



## 准备镜像

```
1. 查看镜像
docker pull emqx/emqx:v4.1-rc.1

2. 改名（为了改仓库路径）
docker tag emqx/emqx:v4.1-rc.1 l2c.harbor.cn/library/emqx:v4.1-rc.1

3. 推送
docker push l2c.harbor.cn/library/emqx:v4.1-rc.1
```

## NFS

### 方式1：动态PV

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mqtt-pvc
spec:
  storageClassName: "managed-nfs-storage"
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
```

### 方式2：静态PV

```
# vim 1-pv_pvc.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: dev-mqtt-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: dev-mqtt-nfs
  nfs:
    path: /data/nfs/mqtt/dev
    server: master01
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: dev-platform
  name: dev-mqtt-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: dev-mqtt-nfs
```



## Deoloyment 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: emqx-deployment
  namespace: dev-platform
  labels:
    app: emqx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: emqx
  template:
    metadata:
      labels:
        app: emqx
      annotations:
        version/date: "20210508"
        version/author: "lc"
    spec:
      containers:
      - name: emqx
        image: l2c.harbor.cn/library/emqx:v4.1-rc.1
        ports:
        - name: mqtt
          containerPort: 1883
        - name: mqttssl
          containerPort: 8883
        - name: mgmt
          containerPort: 8081
        - name: ws
          containerPort: 8083
        - name: wss
          containerPort: 8084
        - name: dashboard
          containerPort: 18083
        volumeMounts:
        - name: emqx-data
          mountPath: /opt/emqx/data/mnesia
      volumes:
      - name: emqx-data
        persistentVolumeClaim: 
          claimName: dev-mqtt-pvc
```

## Services

```
apiVersion: v1
kind: Service
metadata:
  name: emqx
  namespace: dev-platform
spec:
  selector:
    app: emqx
  ports:
    - name: mqtt
      port: 1883
      protocol: TCP
      targetPort: mqtt
    - name: mqttssl
      port: 8883
      protocol: TCP
      targetPort: mqttssl
    - name: mgmt
      port: 8081
      protocol: TCP
      targetPort: mgmt
    - name: ws
      port: 8083
      protocol: TCP
      targetPort: ws
    - name: wss
      port: 8084
      protocol: TCP
      targetPort: wss
    - name: dashboard
      port: 18083
      protocol: TCP
      targetPort: dashboard
```

## 连接方式

```
emqx.dev-platform:1883
```

