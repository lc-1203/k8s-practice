# k8s集群中部署rook+ceph云原生存储

> 参考自杜老师（github.com/dotbalo）的[基于世界500强的k8s实战课程](https://ke.qq.com/course/2738602)。
[toc]

## 概念

### Ceph

&emsp;&emsp; Ceph是一个开源的分布式存储系统，具有高扩展性、高性能、高可靠性等特点，提供良好的性能、可靠性和可扩展性。支持对象存储、快存储和文件系统，是目前为云平台提供存储的理想方案。

### Rook

&emsp;&emsp; Rook是一个自我管理的分布式存储编排系统，它本身并不是存储系统，在存储和k8s之间搭建了一个桥梁，使存储系统的搭建或者维护变得特别简单，Rook将分布式存储系统转变为自我管理、自我扩展、自我修复的存储服务。它让一些存储的操作，比如部署、配置、扩容、升级、迁移、灾难恢复、监视和资源管理变得自动化，无需人工处理。并且Rook支持cSsl，可以利用CSI做一些Pvc的快照、扩容等操作。

### 架构

![Rook Components on Kubernetes](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232007682.png)

- 各组件说明
  - Operator：Rook控制端，监控存储守护进程，确保存储集群的健康
  - Agent：在每个存储节点创建，配置了FlexVolume插件和Kubernetes 的存储卷控制框架（CSI）进行集成
  - OSD：提供存储，每块硬盘可以看做一个osd
  - Mon：监控ceph集群的存储情况，记录集群的拓扑，数据存储的位置信息
  - MDS：负责跟踪**文件存储**的层次结构
  - RGW：Rest API结构，提供**对象存储**接口
  - MGR：为外界提供统一入口

## 部署rook+ceph

### 准备事项

#### 建议配置

| 环境             | 测试      | 生产环境           |
| ---------------- | --------- | ------------------ |
| 服务器配置（≥3） | 4核8G     | 8核16G，或更高     |
| 硬盘（数据盘）   | 每台1×20G | 每台2×500G，或更高 |
| 网卡             | ——        | 千兆网卡           |

> 注1：学习用建议阿里云--抢占式实例--张家口区域，4核8G仅需5分钱/小时
>
> 注2：osd仅支持裸盘挂载，不支持目录格式

#### 本文环境

| 环境     | 说明                            |
| -------- | ------------------------------- |
| k8s版本  | 1.20.4                          |
| rook版本 | 1.6.5                           |
| ceph节点 | k8s-node2，k8s-node3，k8s-node4 |
| 硬盘     | vdb（20G），vdc（30G）          |

#### 注意事项

- 防火墙
- 文件描述符大小
- **时间同步**
- 检查污点：`kubectl describe node |grep Taint`
- 挂载磁盘：`lsblk -f`，必须为裸盘
- ceph节点最好不要部署其它服务，如harbor，istio等，可能存在冲突

#### 克隆rook项目到本地

```shell
git clone --single-branch --branch v1.6.5 https://github.com/rook/rook.git
```

#### 同步镜像

>  注：所涉及的镜像都比较大，且gcr镜像下载速度极慢，非常有必要存到自建仓库里，加速pod的创建。

- 查看所需镜像

```yaml
# 进入工作目录
cd /root/rook/cluster/examples/kubernetes/ceph

vim operator.yaml
-------------------------------
  # ROOK_CSI_CEPH_IMAGE: "quay.io/cephcsi/cephcsi:v3.3.1"
  # ROOK_CSI_REGISTRAR_IMAGE: "k8s.gcr.io/sig-storage/csi-node-driver-registrar:v2.0.1"
  # ROOK_CSI_RESIZER_IMAGE: "k8s.gcr.io/sig-storage/csi-resizer:v1.0.1"
  # ROOK_CSI_PROVISIONER_IMAGE: "k8s.gcr.io/sig-storage/csi-provisioner:v2.0.4"
  # ROOK_CSI_SNAPSHOTTER_IMAGE: "k8s.gcr.io/sig-storage/csi-snapshotter:v4.0.0"
  # ROOK_CSI_ATTACHER_IMAGE: "k8s.gcr.io/sig-storage/csi-attacher:v3.0.2"

vim cluster.yaml
--------------------------------
    image: ceph/ceph:v15.2.13
```

- 同步方式1：[阿里云+github构建镜像仓库解决 k8s.gcr.io访问](https://blog.csdn.net/sinat_35543900/article/details/103290782)
- 同步方式2（个人）：替换为阿里源仓库，或同步到自建仓库示例

&emsp;&emsp; 通过香港节点拉取镜像，推到阿里云个人仓库，再转到自建的Harbor仓库，也比较麻烦。

> [自建Harbor仓库（上一篇文章）](https://blog.csdn.net/qq_14999375/article/details/118070832)

```shell
# 重新标记tag
docker tag quay.io/cephcsi/cephcsi:v3.3.1 hb.l2c.cn/rook-ceph/cephcsi:v3.3.1

# 推送到私有仓库
docker push hb.l2c.cn/rook-ceph/cephcsi:v3.3.1
```

### 安装rook+ceph集群

#### 创建crd&common

- **直接创建crd&common**

```shell
kubectl create -f crds.yaml -f common.yaml 
```

#### 创建operator（rook）

- 修改配置

```shell
vim operator.yaml
-------------------------------------

# 修改镜像（已同步到私有仓库）
  ROOK_CSI_CEPH_IMAGE: "hb.l2c.cn/rook-ceph/cephcsi:v3.3.1"
  ROOK_CSI_REGISTRAR_IMAGE: "hb.l2c.cn/rook-ceph/csi-node-driver-registrar:v2.0.1"
  ROOK_CSI_RESIZER_IMAGE: "hb.l2c.cn/rook-ceph/csi-resizer:v1.0.1"
  ROOK_CSI_PROVISIONER_IMAGE: "hb.l2c.cn/rook-ceph/csi-provisioner:v2.0.4"
  ROOK_CSI_SNAPSHOTTER_IMAGE: "hb.l2c.cn/rook-ceph/csi-snapshotter:v4.0.0"
  ROOK_CSI_ATTACHER_IMAGE: "hb.l2c.cn/rook-ceph/csi-attacher:v3.0.2"
  
# 开启自动发现磁盘（用于后期扩展）
  ROOK_ENABLE_DISCOVERY_DAEMON: "true"
```

- **部署operator**

```shell
kubectl create -f operator.yaml
```

#### 创建cluster（ceph）

- 修改配置：等待operator容器和discover容器启动，配置osd节点

```yaml
vim cluster.yaml
-------------------------------------

# 修改镜像
    image: hb.l2c.cn/rook-ceph/ceph:v15.2.13

# 改为false，并非使用所有节点所有磁盘作为osd
# 启用deviceFilter
# 按需配置config
# 会自动跳过非裸盘
  storage: # cluster level storage configuration and selection
    useAllNodes: false
    useAllDevices: false
    deviceFilter:
    config:
    nodes:
    - name: "k8s-node2"
      deviceFilter: "vdb"
    - name: "k8s-node3"
      deviceFilter: "vdb"
    - name: "k8s-node4"
      deviceFilter: "^vd."  #自动匹配vd开头的裸盘
```

- **部署cluster**

```shell
kubectl create -f cluster.yaml
```

- 查看集群部署进度

```shell
# 实时查看pod创建进度
kubectl get pod -n rook-ceph -w

# 实时查看集群创建进度
kubectl get cephcluster -n rook-ceph rook-ceph -w

# 详细描述
kubectl describe cephcluster -n rook-ceph rook-ceph
```

- 待osd-x的容器启动，表示安装成功

![image-20210621183546501](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232007652.png)

![image-20210621182916337](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232007665.png)

#### 添加裸盘测试

- 给k8s-node4节点挂载一块裸盘

![image-20210621185209968](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232007701.png)

- 系统会自动检测到裸盘并创建osd：`kubectl get cephcluster -n rook-ceph rook-ceph -w`

![image-20210621185259799](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232007611.png)

#### 安装ceph客户端工具

```shell
# 进入工作目录
cd /root/rook/cluster/examples/kubernetes/ceph

# 创建toolbox
kubectl  create -f toolbox.yaml -n rook-ceph

# 查看pod
kubectl  get pod -n rook-ceph -l app=rook-ceph-tools

# 进入pod
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash

# 查看集群状态
ceph status

# 查看osd状态
ceph osd status

# 集群空间用量
ceph df
```

#### 暴露ceph-dashboard

- 为ceph-dashboard创建一个NodePort（原service无法修改，因此需另建service）

```yaml
vim dashboard-nodeport.yaml
--------------------------------------

apiVersion: v1
kind: Service
metadata:
  labels:
    app: rook-ceph-mgr
    ceph_daemon_id: a
    rook_cluster: rook-ceph
  name: rook-ceph-mgr-dashboard-np
  namespace: rook-ceph
spec:
  ports:
  - name: http-dashboard
    port: 7000
    protocol: TCP
    targetPort: 7000
    nodePort: 30700
  selector:
    app: rook-ceph-mgr
    ceph_daemon_id: a
  sessionAffinity: None
  type: NodePort
  

---------------------------
# 创建
kubectl apply -f dashboard-nodeport.yaml
```

- 查看ceph-dashboard登录密码

```
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo

:6-<5K77\S7dkXjwq^*_
```

- 登录

![image-20210621191239853](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232007599.png)

#### 消除HEALTH_WARN警告

- 查看警告详情
  - AUTH_INSECURE_GLOBAL_ID_RECLAIM_ALLOWED: mons are allowing insecure global_id reclaim
  - MON_DISK_LOW: mons a,b,c are low on available space

![image-20210621191403196](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232007876.png)

> 官方解决方案：https://docs.ceph.com/en/latest/rados/operations/health-checks/

- AUTH_INSECURE_GLOBAL_ID_RECLAIM_ALLOWED

```shell
# 进入toolbox
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash

ceph config set mon auth_allow_insecure_global_id_reclaim false
```

- MON_DISK_LOW：根分区使用率过高，清理即可。

![image-20210621195325516](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232007895.png)

## Ceph存储使用

### 三种存储类型

| 存储类型           | 特征                                                         | 应用场景               | 典型设备       |
| ------------------ | ------------------------------------------------------------ | ---------------------- | -------------- |
| 块存储（RBD）      | 存储速度较快<br />不支持共享存储 [**ReadWriteOnce**]         | 虚拟机硬盘             | 硬盘<br />Raid |
| 文件存储（CephFS） | 存储速度慢（需经操作系统处理再转为块存储）<br />支持共享存储 [**ReadWriteMany**] | 文件共享               | FTP<br />NFS   |
| 对象存储（Object） | 具备块存储的读写性能和文件存储的共享特性<br />操作系统不能直接访问，只能通过应用程序级别的API访问 | 图片存储<br />视频存储 | OSS            |

### 块存储使用

#### 创建CephBlockPool和StorageClass

- 文件路径：`/root/rook/cluster/examples/kubernetes/ceph/csi/rbd/storageclass.yaml`
- CephBlockPool和StorageClass都位于storageclass.yaml 文件

- 配置文件简要解读：

```yaml
vim storageclass.yaml
-------------------------------------------

apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host # host级容灾
  replicated:
    size: 3           # 默认三个副本
    requireSafeReplicaSize: true  # 强制高可用，如果size为1则需改为false

---
apiVersion: storage.k8s.io/v1
kind: StorageClass                       # sc无需指定命名空间
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com   # 存储驱动
parameters:
  clusterID: rook-ceph # namespace:cluster
  pool: replicapool                       # 关联到CephBlockPool
  imageFormat: "2"
  imageFeatures: layering

  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph # namespace:cluster
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph # namespace:cluster
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph # namespace:cluster
  
  csi.storage.k8s.io/fstype: ext4   

allowVolumeExpansion: true               # 是否允许扩容
reclaimPolicy: Delete                    # PV回收策略
mountOptions:                            # 挂载选项
  - discard                              # 自动释放上层应用数据已经删除的块，会影响性能
```

- 创建CephBlockPool和StorageClass

```shell
kubectl create -f storageclass.yaml
```

- 查看

```shell
# 查看sc
kubectl get sc

# 查看CephBlockPool（也可在dashboard中查看）
kubectl get cephblockpools -n rook-ceph
```

#### 块存储使用示例

- **Deployment**单副本+**PersistentVolumeClaim**

```yaml
vim nginx-deploy-rbd.yaml
-------------------------------

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-deploy-rbd
  name: nginx-deploy-rbd
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-deploy-rbd
  template:
    metadata:
      labels:
        app: nginx-deploy-rbd
    spec:
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: nginx-rbd-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-rbd-pvc
spec:
  storageClassName: "rook-ceph-block"
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

- **StatefulSet**多副本+**volumeClaimTemplates**

```yaml
vim nginx-ss-rbd.yaml
-------------------------------

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-ss-rbd
spec:
  selector:
    matchLabels:
      app: nginx-ss-rbd 
  serviceName: "nginx"
  replicas: 3 
  template:
    metadata:
      labels:
        app: nginx-ss-rbd 
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "rook-ceph-block"
      resources:
        requests:
          storage: 2Gi
```

![image-20210622161110029](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232007989.png)

```
/var/lib/kubelet/pods/cba9a872-7c3d-426c-b2ac-1427ec74416e/volumes/kubernetes.io~csi/pvc-07d04670-0275-4b22-ad71-75fe59f31fe7/mount
```

#### 查询块存储挂载情况

1. 定位pv：`kubectl get pv`

![image-20210624150302410](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232007004.png)

2. 查询imageName和pool：`kubectl describe pv pvc-43079df0-80ea-463c-907c-e23b217a2d38`

![image-20210624150545172](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232007978.png)

3. 定位客户端

```shell
kubectl exec -it -n rook-ceph rook-ceph-tools-4lv4d -- bash
rbd status replicapool/csi-vol-bf56d472-d4a2-11eb-becd-e2bb3b8d14dd
```

![image-20210624150648857](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232007019.png)

4. 定位主机：`kubectl get pod -A -o wide |grep 10.244.159.155`

![image-20210624150809527](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232007192.png)

5. 在该主机上查询rbd挂载情况：`lsblk`

![image-20210624150912893](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232007254.png)

6. 进入挂载目录

![image-20210624150952387](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232007305.png)

### 共享文件存储

#### 部署MDS

&emsp;&emsp; 创建Cephfs文件系统需要先部署MDS服务，该服务负责处理文件系统中的元数据。

- 文件路径：`/root/rook/cluster/examples/kubernetes/ceph/filesystem.yaml`

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph 
spec:
  metadataPool:
    replicated:
      size: 3                         # 元数据副本数
      requireSafeReplicaSize: true
    parameters:
      compression_mode:
        none
  dataPools:
    - failureDomain: host
      replicated:
        size: 3                     # 存储数据的副本数
        requireSafeReplicaSize: true
      parameters:
        compression_mode:
          none
  preserveFilesystemOnDelete: true
  metadataServer:
    activeCount: 1                # MDS实例的副本数，默认1，生产环境建议设置为3
```

#### 创建StorageClass

- 文件路径：`/root/rook/cluster/examples/kubernetes/ceph/csi/cephfs/storageclass.yaml`

```shell
kubectl apply -f storageclass.yaml
```

#### 共享文件存储使用示例

- **Deployment**多副本+ **PersistentVolumeClaim**

```yaml
vim nginx-deploy-cephfs.yaml
-------------------------------

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-deploy-cephfs
  name: nginx-deploy-cephfs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-deploy-cephfs
  template:
    metadata:
      labels:
        app: nginx-deploy-cephfs
    spec:
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: nginx-cephfs-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-cephfs-pvc
spec:
  storageClassName: "rook-cephfs"
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

![image-20210622164001844](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202202232007300.png)

## 扩容、快照、克隆

## rook+ceph集群清理

>  参考官网：[rook+ceph集群清理官方指南](https://www.rook.io/docs/rook/v1.6/ceph-teardown.html)

### 删除pvc、sc及对应的存储资源

```yaml
# 按需删除pvc、pv

namespace=kafka && kubectl get pvc -n ${namespace} | awk '{print $1};' | xargs kubectl delete pvc -n ${namespace}

kubectl get pv | grep Released | awk '{print $1};' | xargs kubectl delete pv

# 删除块存储及SC
kubectl delete -n rook-ceph cephblockpool replicapool
kubectl delete storageclass rook-ceph-block

# 删除文件系统存储及SC
kubectl delete -n rook-ceph cephfilesystem myfs
kubectl delete storageclass rook-cephfs

# 删除对象存储
```

### 删除ceph集群

```yaml
kubectl -n rook-ceph patch cephcluster rook-ceph --type merge -p '{"spec":{"cleanupPolicy":{"confirmation":"yes-really-destroy-data"}}}'

kubectl -n rook-ceph delete cephcluster rook-ceph

# 若无法删除，手动终结
for CRD in $(kubectl get crd -n rook-ceph | awk '/ceph.rook.io/ {print $1}'); do
    kubectl get -n rook-ceph "$CRD" -o name | \
    xargs -I {} kubectl patch {} --type merge -p '{"metadata":{"finalizers": [null]}}' -n rook-ceph
done

# 查看ceph资源
kubectl -n rook-ceph get cephcluster
```

### 删除rook集群crd

```shell
# 工作目录
cd /root/rook/cluster/examples/kubernetes/ceph

kubectl delete -f operator.yaml
kubectl delete -f common.yaml
kubectl delete -f crds.yaml
```

### 检查资源是否清除完毕

```shell
kubectl api-resources --verbs=list --namespaced -o name \
  | xargs -n 1 kubectl get --show-kind --ignore-not-found -n rook-ceph
```

### 清楚主机上的数据

#### 准备清除脚本

- 参考官方示例，按需修改

```yaml
vim rook-clean.sh
----------------------------

#!/usr/bin/env bash
DISK="/dev/vdb"
# Zap the disk to a fresh, usable state (zap-all is important, b/c MBR has to be clean)
# You will have to run this step for all disks.
sgdisk --zap-all $DISK
# Clean hdds with dd
dd if=/dev/zero of="$DISK" bs=1M count=100 oflag=direct,dsync
# Clean disks such as ssd with blkdiscard instead of dd
blkdiscard $DISK

# These steps only have to be run once on each node
# If rook sets up osds using ceph-volume, teardown leaves some devices mapped that lock the disks.
ls /dev/mapper/ceph-* | xargs -I% -- dmsetup remove %
# ceph-volume setup can leave ceph-<UUID> directories in /dev (unnecessary clutter)
rm -rf /dev/ceph-*

--------------------------
chmod +x rook-clean.sh
```

#### 编写Ansible剧本

- hosts

```
[master]
172.16.16.11 node_name=k8s-master1
172.16.16.12 node_name=k8s-master2
172.16.16.13 node_name=k8s-master3

[node]
172.16.16.101 node_name=k8s-node1
172.16.16.102 node_name=k8s-node2
172.16.16.103 node_name=k8s-node3
172.16.16.104 node_name=k8s-node4

[osd]
172.16.16.102 node_name=k8s-node2
172.16.16.103 node_name=k8s-node3
172.16.16.104 node_name=k8s-node4
```

- delete-old-rookfile.yml

```yaml
- hosts:
  - master
  - node
  tasks:
  - name: 清除rook目录
    shell: rm -rf /var/lib/rook/*
```

- reset-rook-osd.yml

```yaml
- hosts:
  - osd
  tasks:
  - name: 分发清理脚本
    copy: src=/root/ansible-install-k8s-master-v1.20/rook-clean.sh dest=~/ mode=u+x
  - name: 执行清理脚本
    shell: /bin/bash ~/rook-clean.sh
```

- 执行

```shell
ansible-playbook -i hosts delete-old-rookfile.yml -uroot -k
ansible-playbook -i hosts reset-rook-osd.yml -uroot -k
```

### 问题

#### 无法删除

ns无法删除，显示apiservice错误，删除出错的apiservice即可

## 删除osd

```shell
# 1. 停止Rook operator
kubectl -n rook-ceph scale deployment rook-ceph-operator --replicas=0

# 2. 标记为out（进入tools）
ceph osd out osd.<ID>

# 3. 等待数据迁移，知道pg状态为active+clean
ceph status

# 4. 移除OSD
ceph osd purge <ID> --yes-i-really-mean-it

# 5. 如果创建集群时removeOSDsIfOutAndSafeToRemove不为true，则需手动delete
kubectl delete deployment -n rook-ceph rook-ceph-osd-<ID>

# 6. 再次执行步骤4，并验证osd状态
ceph osd tree

# 7. 卸载该数据盘，然后启动Rook operator
kubectl -n rook-ceph scale deployment rook-ceph-operator --replicas=1
```

