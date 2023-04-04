# 阿里云自建K8S集群使用CSI存储组件

>  [在自建Kubernetes集群中使用阿里云CSI存储组件](https://help.aliyun.com/document_detail/335175.html)

方式一：通过注册集群自动安装CSI插件

方式二：手动安装CSI插件（跳过第一章）

## 自建K8S集群接入阿里ACK

> [创建注册集群并接入本地数据中心集群](https://help.aliyun.com/document_detail/121053.htm?spm=a2c4g.11186623.0.0.1057452dIrIwiT#task-skz-qwk-qfb)

### 创建注册集群

- 负载均衡器（0.1/小时）
- 无需使用EIP（弹性公网IP）

### 将目标集群接入注册集群

## 手动部署CSI存储组件

### 自建Kubernetes集群添加节点标签

> 自动添加：https://developer.aliyun.com/article/786335?spm=a2c4g.11186623.0.0.1057452dIrIwiT

- 手动添加：

```shell
META_EP=http://100.100.100.200/latest/meta-data &&
echo `curl -s $META_EP/region-id`.`curl -s $META_EP/instance-id`


kubectl patch node master1 -p '{"spec":{"providerID": "cn-zhangjiakou.i-8vbhy24ntae8zwo8zudn"}}'
kubectl patch node master2 -p '{"spec":{"providerID": "cn-zhangjiakou.i-8vbhy24ntae8zwo8zudo"}}'
kubectl patch node master3 -p '{"spec":{"providerID": "cn-zhangjiakou.i-8vbhy24ntae8zwo8zudr"}}'
kubectl patch node node01 -p '{"spec":{"providerID": "cn-zhangjiakou.i-8vbhy24ntae8zwo8zudq"}}'
```

### 配置CSI组件的RAM权限

>  参考[在自建Kubernetes集群中使用阿里云CSI存储组件](https://help.aliyun.com/document_detail/335175.html)

### 安装CSI组件

> 已包含块存储、NAS和OSS

- 配置AK

```shell
kubectl -n kube-system create secret generic alibaba-addon-secret --from-literal='access-key-id=xxxxx' --from-literal='access-key-secret=xxxxx'
```

- 下载csi安装文件

```shell
https://github.com/kubernetes-sigs/alibaba-cloud-csi-driver/tree/master/deploy
rbac.yaml
./ack/csi-plugin.yaml
./ack/csi-provisioner.yaml
```

- 编辑配置

```yaml
            - name: MAX_VOLUMES_PERNODE
              value: "15"
            - name: SERVICE_TYPE
              value: "plugin"
            - name: ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  key: access-key-id
                  name: alibaba-addon-secret
            - name: ACCESS_KEY_SECRET
              valueFrom:
                secretKeyRef:
                  key: access-key-secret
                  name: alibaba-addon-secret
```

- 部署

- 经验证无需rbac授权

```shell
kubectl apply -f rbac.yaml
# 查看token名
kubectl get secrets -A |grep csi

csi-admin-token-xt4k8
```

## 使用CSI存储插件

### 块存储

**计费说明：必须是按量付费**

#### 存储规格及价格对比

> [块存储定价](https://www.aliyun.com/price/product?spm=a2c4g.11186623.0.0.655d6ecfNW4RwK#/disk/detail/disk)

> [使用限制](https://help.aliyun.com/document_detail/25412.html)

| 类型        | 容量限制(GB)               | 性能（最大IOPS/最大吞吐量） | 按量价格 | 100G每月费用 |
| ----------- | -------------------------- | --------------------------- | -------- | ------------ |
| 高效云盘    | 20 ~ 32768                 | 5k/ 140MB                   | 0.00049  | 35           |
| SSD云盘     | 20 ~ 32768                 | 25k / 300MB                 | 0.00140  | 101          |
| ESSD云盘PL0 | 20 ~ 32768（==40待验证==） | 10k  / 180MB                | 0.00105  | 76           |
| ESSD云盘PL1 | 20 ~ 32768                 | 50k    / 350MB              | 0.00210  | 151          |

#### 创建

- sc：块存储sc已在csi-provisioner中集中创建

#### 扩容

### NAS/CPFS

| 类型             | 容量限制           | 带宽/IOPS/时延                         | 100G每月费用 |
| ---------------- | ------------------ | -------------------------------------- | ------------ |
| CPFS             | ==3.6TiB== ～ 1PiB | 360MBps 起<br/>54k 起<br />0.8ms       | 83           |
| NAS极速型        | 100GiB~256TiB      | 1200MBps~9600MBps<br />200K<br />0.2ms | 180          |
| NAS通用型-容量型 | 0~10PiB            | 1024MBps<br />15K<br />100ms           | 35           |
| NAS通用型-性能型 | 0~1PiB             | 2048MBps<br />30K<br />2ms             | 151          |

CPFS：3600G起步，扩容步长1200G，4.150/h，不建议使用

NAS-极速型：100G起步，扩容步长1G，0.250/h,180元/100G/月，1.8元/GiB/月

NAS-通用型-容量型：0.35元/GiB/月

#### 创建文件系统并添加挂载点

```
09cceeda-u7rm.cn-zhangjiakou.extreme.nas.aliyuncs.com
```

#### 创建sc

```
vim alicloud-nas-subpath.yaml
```

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: alicloud-nas-subpath
mountOptions:
- nolock,tcp,noresvport
- vers=3
parameters:
  volumeAs: subpath
  server: "09cceeda-u7rm.cn-zhangjiakou.extreme.nas.aliyuncs.com:/k8s/"
provisioner: nasplugin.csi.alibabacloud.com
reclaimPolicy: Retain
```

