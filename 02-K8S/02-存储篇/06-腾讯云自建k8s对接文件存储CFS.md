# 腾讯云自建k8s对接文件存储CFS

## 环境

服务器：轻量应用服务器，使用私有网络VPC，需关联云联网实现内网互联

k8s：通过sealyun一键安装1.19.16版本k8s单主集群

csi插件：[kubernetes-csi-tencentcloud](https://github.com/TencentCloud/kubernetes-csi-tencentcloud/blob/master/docs/README_CFS.md)

## 部署步骤

### 创建文件系统及挂载点

- 创建vpc，注意不要与轻量应用服务器自动生成的vpc冲突，否则云联网无法互联
- 创建文件系统及挂载点，记录下文件存储的客户端IP：172.16.0.10

![image-20220314134502850](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20220314134502.png)

### 验证网络通信

- 申请云联网关联，将轻量应用服务实例关联云联网；
- 将文件系统的 VPC 实例关联至云联网
- 在轻量应用服务器上telnet文件存储客户端的nfs端口，验证网络通信

```shell
telnet 172.16.0.10 111
telnet 172.16.0.10 2049 
```

![image-20220314134410000](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20220314134410.png)

### 挂载测试

```shell
mkdir /localfolder
sudo mount -t nfs -o vers=4.0,noresvport 172.16.0.10:/ /localfolder
```

### 安装CSI插件

#### 准备API密钥

创建子用户，并授权文件存储全读写访问权限，记录API密钥

```shell
SecretId AKID4xaw2rRknZ4IFvKSJtCEzUifLksjKfkY 
SecretKey xxxxxxxxxxxxxxxxxxxxxxx
```

#### 安装CSI-CFS插件

插件Github地址：[kubernetes-csi-tencentcloud/cfs](https://github.com/TencentCloud/kubernetes-csi-tencentcloud/tree/master/deploy/cfs/kubernetes)

- rbac

```shell
kubectl apply -f  deploy/cfs/kubernetes/csi-cfs-rbac.yaml
```

- 根据k8s版本安装相应版本的插件，将API密钥填充至`csi-provisioner-cfsplugin.yaml`文件中

```shell
kubectl apply -f  deploy/cfs/kubernetes/csi-cfs-csidriver-old.yaml
kubectl apply -f  deploy/cfs/kubernetes/csi-nodeplugin-cfsplugin-new.yaml
kubectl apply -f  deploy/cfs/kubernetes/csi-provisioner-cfsplugin-new.yaml
```

## 静态挂载测试

- 创建web目录

```shell
mkdir -p /localfolder/web
```

- static-allinone

```shell
vim static-allinone.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: web-static-pv
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Gi
  csi:
    driver: com.tencent.cloud.csi.cfs
    volumeHandle: web-static-pv
    volumeAttributes: 
      host: 172.16.0.10
      path: /web       # 如果不是根目录的话，需提前创建
  storageClassName: ""
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-static-pvc
spec:
  storageClassName: ""
  volumeName: web-static-pv
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx 
spec:
  containers:
  - image: ccr.ccs.tencentyun.com/qcloud/nginx:1.9
    imagePullPolicy: Always
    name: nginx
    ports:
    - containerPort: 80
      protocol: TCP
    volumeMounts:
      - mountPath: /usr/share/nginx/html
        name: data-vol
  volumes:
  - name: data-vol
    persistentVolumeClaim:
      claimName: web-static-pvc
```

- 测试

```shell
echo "hello cfs" > /localfolder/web/index.html
```

![image-20220314142049299](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20220314142049.png)

## 动态挂载

```shell
vim dynamic-provison-allinone.yaml
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cfsauto
parameters:
  vpcid: vpc-klr8fwg5
  subnetid: subnet-63c5x24c
  resourcetags: ""
provisioner: com.tencent.cloud.csi.cfs
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-cfsplugin
spec:
  storageClassName: cfsauto
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: ccr.ccs.tencentyun.com/qcloud/nginx:1.9
    imagePullPolicy: Always
    name: nginx
    ports:
    - containerPort: 80
      protocol: TCP
    volumeMounts:
      - mountPath: /usr/share/nginx/html
        name: data-cfsplugin
  volumes:
  - name: data-cfsplugin
    persistentVolumeClaim:
      claimName: data-cfsplugin
```

```shell
kubectl apply -f dynamic-provison-allinone.yaml
```

- 会自动创建新的cfs，好像这个sc不能直接绑定已有cfs的subpath（待后续研究）

![image-20220314192025136](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203141920253.png)
