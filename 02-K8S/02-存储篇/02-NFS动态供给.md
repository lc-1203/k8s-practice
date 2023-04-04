

# NFS动态供给

[toc]

## 安装nfs客户端

- 在k8s集群中的每个节点安装nfs-utils

```shell
yum install nfs-utils -y &&
systemctl restart nfs && systemctl enable nfs
```

- 启动服务

```shell
systemctl restart nfs && systemctl enable nfs
```

- 选择一台机器创建共享总目录 (为了方便，这里选择主节点[172.16.16.11])

```shell
mkdir -p /data/nfs
```

- 编辑配置 

> 注：*表示内网所有机器都可访问并挂载目录，这里最好添加白名单限制，仅对K8S集群内节点开放

```shell
# 打开配置文件
vim /etc/exports

# 未设置权限
/data/nfs *(rw,no_root_squash)

# 添加挂载IP限制
/data/nfs 172.16.16.0/16(rw,async,no_root_squash)
```

```
/data/nfs 10.7.20.200(rw,async,no_root_squash) 10.7.20.201(rw,async,no_root_squash) 10.7.20.202(rw,async,no_root_squash) 10.7.20.203(rw,async,no_root_squash) \
10.7.20.204(rw,async,no_root_squash) 10.7.20.205(rw,async,no_root_squash) 10.7.20.206(rw,async,no_root_squash) \
10.7.20.207(rw,async,no_root_squash) 10.7.20.208(rw,async,no_root_squash) 10.7.16.204(rw,async,no_root_squash) 10.7.21.10(rw,async,no_root_squash) \
10.7.17.14(rw,async,no_root_squash) 10.7.17.15(rw,async,no_root_squash) 10.7.16.180(rw,async,no_root_squash)

10.7.17.14
```

![image-20210804172945369](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232006330.png)

- 使挂载配置生效并验证

```shell
# 重新挂载并显示（无需重启服务）
exportfs -rv

# 本机查看挂载情况
showmount -e

# 在其它节点查看挂载情况
showmount -e 172.16.16.11
```

![image-20210804181022842](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232006340.png)

- 为需要持久化的服务创建子目录（必须提前手动创建）

> 后文演示动态供给功能，因此添加子目录为“dynamic”，亦可根据需求创建其它子目录

```shell
mkdir -p /data/nfs/dynamic
```

- 挂载测试

```yaml
# 登录其它节点
mkdir test
mount -t nfs 172.16.16.11:/data/nfs test

# 查看挂载情况
mount |grep 172.16.16.11

# 卸载挂载目录
umount test
```

![image-20210804182158710](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232006338.png)

## 部署nfs-client-provisioner插件

### 配置授权（RBAC）

```shell
vim rbac.yaml
```

```yaml
kind: ServiceAccount
apiVersion: v1
metadata:
  name: nfs-client-provisioner
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

```shell
kubectl apply -f rbac.yaml
```

### Deployment

> 需配置挂载目录172.16.16.11: /data/nfs/dynamic

```shell
vim nfs-client-provisioner.yaml
```

```yaml
kind: Deployment
apiVersion: apps/v1 
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: nfs.provisioner
            - name: NFS_SERVER
              value: 172.16.16.11
            - name: NFS_PATH
              value: /data/nfs/dynamic
      volumes:
        - name: nfs-client-root
          nfs:
            server: 172.16.16.11
            path: /data/nfs/dynamic
```

```shell
kubectl apply -f nfs-client-provisioner.yaml
```

### 创建StorageClass

```shell
vim storageclass.yaml
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-sc
provisioner: nfs.provisioner
parameters:
  archiveOnDelete: "true"
allowVolumeExpansion: true
```

```
kubectl apply -f storageclass.yaml
```

![image-20210804201759761](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232006361.png)

- archiveOnDelete: "true"，PVC被删除后，挂载的文件夹将会被标记为“archived”

![image-20210804203706418](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232006357.png)

## 使用示例

 NFS动态供给可以根据访问模式主要分为ReadWriteOnce和ReadWriteMany两种方式，类似于块存储和文件存储的使用：

- ReadWriteOnce（RWO）：该卷能够以读写模式被加载到一个节点上
- ReadWriteMany（RWX）：该卷能够以读写模式被多个节点同时加载

### Deployment+RWO

> 一对一挂载

```shell
vim 1-deployment-rwo.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-deploy-rwo
spec:
  storageClassName: "nfs-sc"
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy-rwo
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
          claimName: nginx-deploy-rwo
```

```shell
kubectl apply -f 1-deployment-rwo.yaml
```

![image-20210804195745446](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232006351.png)

- 文件挂载测试

```yaml
# 进入pod
kubectl exec -it nginx-deploy-rwo-6c7cf9ccdf-mkbcx -- sh

# 在持久化目录下生成文件
echo "hello,1-deployment-rwo" > /usr/share/nginx/html/1-deployment-rwo.html
```

- 到本地挂载目录查看文件

![image-20210804201029272](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232006695.png)

### Statefulset+RWO

> Statefulset多副本一对一挂载使用volumeClaimTemplates，并指定storageClass，将自动创建pvc、pv

```shell
vim 2-nginx-sfs-rwo.yaml
```

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-sfs-rwo
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx"
  replicas: 3 
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
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "nfs-sc"
      resources:
        requests:
          storage: 1Gi
```

```shell
kubectl apply -f 2-nginx-sfs-rwo.yaml
```

![image-20210804201733423](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232006697.png)

- 文件挂载测试

```shell
# 进入nginx-sfs-rwo-0
kubectl exec -it nginx-sfs-rwo-0 -- sh
echo "hello,this is nginx-sfs-rwo-0" > /usr/share/nginx/html/index.html

# 进入nginx-sfs-rwo-1
kubectl exec -it nginx-sfs-rwo-1 -- sh
echo "hello,this is nginx-sfs-rwo-1" > /usr/share/nginx/html/index.html
```

- 然后分别访问nginx-sfs-rwo-0和nginx-sfs-rwo-1

![image-20210804201642905](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232006705.png)

### Deployment/Statefulset+RWX

> 多pod挂载同一个卷，示例中使用Deployment或Statefulset部署皆可

```
vim 3-nginx-rwx.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-rwx
spec:
  storageClassName: "nfs-sc"
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-rwx
  labels:
    app: nginx
spec:
  replicas: 3
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
          claimName: nginx-rwx
```

```shell
kubectl apply -f 3-nginx-rwx.yaml
```

- 文件挂载测试

```shell
# 进入第一个pod
kubectl exec -it nginx-rwx-64598fd68d-49hxv -- sh
echo "hello,this is nginx-rwx-0" > /usr/share/nginx/html/index.html
```

- 访问第二个pod

![image-20210804202633006](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232006692.png)

