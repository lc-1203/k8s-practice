# Local PV配置及使用

[toc]

## 存储卷

### 网络存储 VS 本地存储

Kubernetes 上的数据持久化需要使用 [PersistentVolume (PV)](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)。Kubernetes 提供多种[存储类型](https://kubernetes.io/docs/concepts/storage/volumes/)，主要分为两大类：

- 网络存储

  存储介质不在当前节点，而是通过网络方式挂载到当前节点。一般有多副本冗余提供高可用保证，在节点出现故障时，对应网络存储可以再挂载到其它节点继续使用。

- 本地存储

  存储介质在当前节点，通常能提供比网络存储==更低的延迟==，但没有多副本冗余，一旦节点出故障，数据就有可能丢失。如果是 IDC 服务器，节点故障可以一定程度上对数据进行恢复，但公有云上使用本地盘的虚拟机在节点故障后，数据是**无法找回**的。

### Local PV

> https://segmentfault.com/a/1190000040882124

#### hostPath vs Local PV

- hostPath需配置pod亲和性，选择节点来调度（通过nodeSelector或者nodeAffinity绑定到具体node上）
- hostPath需事先创建好目录，还得注意权限的配置
- hostPath不能指定大小 , 面临磁盘随时被写满的危险 , 且没有I/O隔离机制
- statefulset不能使用hostPath Volume , 写好的Helm不能兼容hostPath Volume

#### Local PV 使用场景

适用于高优先级系统，需要在多个不同节点上存储数据，而且I/O要求较高。

#### 局限性

- 延迟绑定机制：provisioner 字段定义为no-provisioner，这是因为本地卷**不支持动态创建pv**，需手动创建pv。但还是需要创建 StorageClass并以延迟卷绑定（WaitForFirstConsumer），延迟绑定PV与PVC的对应关系，直到 Pod 调度完成。
- 数据安全风险：local pv仍受node节点可用性方面的限制，使用local voluems的应用程序必须能够容忍这种降低的可用性以及潜在的数据丢失。
- 容量限制：Local PV不支持动态的PV空间申请管理，得手动对Local PV进行容量规划，对能够使用的本地资源做一个全局规划，然后划分为各种尺寸的卷后挂载到自动发现目录下。

### 最佳实践

- 为了更好的IO隔离效果，建议将一整块磁盘作为一个存储卷使用；
- 为了得到存储空间的隔离，建议为每个存储卷使用一个独立的磁盘分区；
- 避免重新创建具有相同节点名称的node节点；
- 对于具有文件系统的存储卷，建议在fstab条目和该卷的mount安装点的目录名中使用它们的UUID；
- 对于没有文件系统的原始块存储卷，请使用其唯一ID作为符号链接的名称。

## 手动绑定PV使用

### 创建SC

- 注意声明策略，Delete, Retain；重要数据建议使用Retain

```
vim storageclass.yaml
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
# Supported policies: Delete, Retain
```

### 手动创建PV

- 创建挂载目录

```
mkdir -p /mnt/local-vol/nginx
```

```
vim nginx-local-pv.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-local-pv
  labels:                          # 指定标签，用于volumeClaimTemplates中绑定使用
    pv: nginx
spec:
  capacity:
    storage: 1Gi 
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/local-vol/nginx   # 挂载路径
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k8s-master1                # 指定节点
```

### Deployment挂载使用

- pvc

```
vim nginx-pvc.yaml
```

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-local-pvc
spec:
  storageClassName: "local-storage"
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  selector:
    matchLabels:           # 与pv绑定的标签
      pv: nginx
```

- deployment

```
vim nginx-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:stable-alpine
        name: nginx
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: nginx-local-pvc
```

### StatefulSet中绑定使用

```
vim nginx-sfs.yaml
```

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-sfs
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx"
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:stable-alpine
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: nginx-local-pvc
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "local-storage"
      resources:
        requests:
          storage: 100Mi
      selector:
        matchLabels:           # 与pv绑定的标签
          pv: nginx
```

### 删除pvc后重新绑定pv

删除pvc后，pv处于realease状态，无法再绑定

可以通过删除 claimRef 对 PVC 的引用将pv的状态重新变为Available状态

```shell
kubectl edit [pv]
```

## 通过provisioner动态创建pv

> 参考自：[local-volume-provisioner使用](https://www.jianshu.com/p/436945a25e9f)

### 创建挂载点

挂载盘麻烦或者没有，可以参考mount bind方式生成

```shell
#!/bin/bash
for i in $(seq 1 5); do
  mkdir -p /mnt/fast-disks-bind/vol${i}
  mkdir -p /mnt/fast-disks/vol${i}
  mount --bind /mnt/fast-disks-bind/vol${i} /mnt/fast-disks/vol${i}
done
```

## 部署 local-volume-provisioner 程序

涉及镜像：`quay.io/external_storage/local-volume-provisioner:v2.3.4`

```
vim local-volume-provisioner.yaml
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: "local-storage"
provisioner: "kubernetes.io/no-provisioner"
volumeBindingMode: "WaitForFirstConsumer"

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-provisioner-config
  namespace: kube-system
data:
  setPVOwnerRef: "true"
  nodeLabelsForPV: |
    - kubernetes.io/hostname
  storageClassMap: |
    local-storage:
      hostDir: /mnt/disks
      mountDir: /mnt/disks

---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: local-volume-provisioner
  namespace: kube-system
  labels:
    app: local-volume-provisioner
spec:
  selector:
    matchLabels:
      app: local-volume-provisioner
  template:
    metadata:
      labels:
        app: local-volume-provisioner
    spec:
      serviceAccountName: local-storage-admin
      containers:
        - image: "quay.io/external_storage/local-volume-provisioner:v2.3.4"
          name: provisioner
          securityContext:
            privileged: true
          env:
          - name: MY_NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          - name: MY_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: JOB_CONTAINER_IMAGE
            value: "quay.io/external_storage/local-volume-provisioner:v2.3.4"
          resources:
            requests:
              cpu: 100m
              memory: 100Mi
            limits:
              cpu: 100m
              memory: 100Mi
          volumeMounts:
            - mountPath: /etc/provisioner/config
              name: provisioner-config
              readOnly: true
            # mounting /dev in DinD environment would fail
            # - mountPath: /dev
            #   name: provisioner-dev
            - mountPath: /mnt/disks
              name: local-disks
              mountPropagation: "HostToContainer"
      volumes:
        - name: provisioner-config
          configMap:
            name: local-provisioner-config
        # - name: provisioner-dev
        #   hostPath:
        #     path: /dev
        - name: local-disks
          hostPath:
            path: /mnt/disks

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: local-storage-admin
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: local-storage-provisioner-pv-binding
  namespace: kube-system
subjects:
- kind: ServiceAccount
  name: local-storage-admin
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: system:persistent-volume-provisioner
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: local-storage-provisioner-node-clusterrole
  namespace: kube-system
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: local-storage-provisioner-node-binding
  namespace: kube-system
subjects:
- kind: ServiceAccount
  name: local-storage-admin
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: local-storage-provisioner-node-clusterrole
  apiGroup: rbac.authorization.k8s.io
```



```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-local-pv
spec:
  capacity:
    storage: 5Gi 
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - my-node
```

