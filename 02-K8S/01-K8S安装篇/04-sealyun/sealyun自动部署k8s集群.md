# sealyun自动部署k8s集群

## 环境

腾讯云主机一台，内网地址`10.0.8.17`，通过sealyun安装k8s1.19.16（docker）版本一主集群。

## 安装k8s集群

- 下载1.19.16安装包，[下载地址](https://www.sealyun.com/goodsDetail?type=cloud_kernel&name=kubernetes)
- 下载并安装sealos工具

```shell
# sealos是个golang的二进制工具，直接下载拷贝到bin目录即可, release页面也可下载
wget -c https://sealyun-home.oss-cn-beijing.aliyuncs.com/sealos/latest/sealos && \
    chmod +x sealos && mv sealos /usr/bin
```

- 一键部署k8s集群

```shell
sealos init --passwd 'Aa666666' \
  --master 10.0.8.17 \
  --pkg-url /root/kube1.19.16.tar.gz \
  --podcidr 172.16.0.0/12 \
  --svccidr 192.168.0.0/16 \
  --version v1.19.16 \
  --cert-sans apiserver.l2c.cn
```

| 配置        | 说明           |
| ----------- | -------------- |
| 系统版本    | CentOS7.6      |
| 内核        | 3.10.0         |
| K8S版本     | 1.19.16        |
| Docker版本  | 20.10.5        |
| 主机网段    | 10.0.8.0/24    |
| Pod网段     | 172.16.0.0/12  |
| Service网段 | 192.168.0.0/16 |

## 允许master节点调度

```shell
# 允许调度
kubectl taint node master01 node-role.kubernetes.io/master-
# 禁止调度
kubectl taint node master01 node-role.kubernetes.io/master="":NoSchedule
```

## kubectl自动补全

```shell
yum install -y bash-completion

source /usr/share/bash-completion/bash_completion &&
source <(kubectl completion bash) &&
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

## Metrics部署

在新版的Kubernetes中系统资源的采集均使用Metrics-server，可以通过Metrics采集节点和Pod的内存、磁盘、CPU和网络的使用率

> 官方地址：[Kubernetes Metrics Server](https://github.com/kubernetes-sigs/metrics-server)

```shell
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

- 修改参数及镜像

```shell
vim components.yaml
```

```yaml
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls
        - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt 
        - --requestheader-username-headers=X-Remote-User
        - --requestheader-group-headers=X-Remote-Group
        - --requestheader-extra-headers-prefix=X-Remote-Extra-
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/metrics-server:v0.6.1
```

- 挂载证书

```yaml
        volumeMounts:
        - mountPath: /tmp
          name: tmp-dir
        - name: ca-ssl
          mountPath: /etc/kubernetes/pki
      nodeSelector:
        kubernetes.io/os: linux
      priorityClassName: system-cluster-critical
      serviceAccountName: metrics-server
      volumes:
      - emptyDir: {}
        name: tmp-dir
      - name: ca-ssl
        hostPath:
          path: /etc/kubernetes/pki
```

## Dashboard部署

Dashboard用于展示集群中的各类资源，同时也可以通过Dashboard实时查看Pod的日志和在容器中执行一些命令等。

> 官方地址： [K8S官方Dashboard最新部署模板](https://github.com/kubernetes/dashboard/blob/master/aio/deploy/recommended.yaml)

- 下载最新部署模板

```
wget https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended.yaml
```

- 修改service

```shell
vim recommended.yaml
```

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort
  ports:
    - port: 443
      nodePort: 31443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
```

```shell
kubectl apply -f recommended.yaml
```

- 创建用户

```shell
vim admin.yaml
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding 
metadata: 
  name: admin-user
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```

```
kubectl apply -f admin.yaml
```

- 查看Token

```shell
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

- 配置安全组策略，开放该NodePort即可访问

## 验证Kube-Proxy模式是否为ipvs

```shell
curl 127.0.0.1:10249/proxyMode
```

## 集群可用性验证

- 部署busybox

```yaml
cat > busybox-deploy.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox
spec:
  selector:
    matchLabels:
      app: busybox
  template:
    metadata:
      labels:
        app: busybox
    spec:
      containers:
        - name: busybox
          image: busybox:1.28        # 1.29以上nslookup命令有问题
          args:
          - /bin/sh
          - -c
          - sleep 10; touch /tmp/healthy; sleep 30000
          readinessProbe:           #就绪探针
            exec:
              command:
              - cat
              - /tmp/healthy
            initialDelaySeconds: 10         #10s之后开始第一次探测
            periodSeconds: 5                #第一次探测之后每隔5s探测一次
EOF
```

- 验证

```shell
# Pod解析Service
export busyboxpod=$(kubectl get pods -l app=busybox -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $busyboxpod -- nslookup kubernetes

# Pod能解析跨namespace的Service
kubectl exec -it $busyboxpod -- nslookup kube-dns.kube-system

# 每个节点都必须要能访问Kubernetes的kubernetes svc 443和kube-dns的service 53
# 安装telnet
yum install telnet -y
telnet 192.168.0.1 443
telnet 192.168.0.10 53

# Pod和Pod之间跨namespace跨机器都能通信

# 另一种方法测试coredns解析，其中10.249.92.65为coredns的POD_IP
yum -y install bind-utils
dig -t A kubernetes.default.svc.cluster.local. @10.249.92.65 +short
```
