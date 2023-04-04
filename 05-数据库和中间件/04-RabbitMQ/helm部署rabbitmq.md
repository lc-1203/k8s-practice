参考说明：
> 安装思路： [Kubernetes全栈架构师：基于世界500强的k8s实战课程](https://ke.qq.com/course/2738602)
> 镜像模式： [RabbitMQ高可用-镜像模式部署使用](https://www.cnblogs.com/huligong1234/p/13549450.html)

## 安装helm

### 安装helm

- 项目地址：https://github.com/helm/helm

- 安装：
```shell
# 下载（自行选择版本）
wget https://get.helm.sh/helm-v3.6.1-linux-amd64.tar.gz

# 解压
tar zxvf helm-v3.6.1-linux-amd64.tar.gz

# 安装
mv linux-amd64/helm /usr/local/bin/

# 验证
helm version
```

### 基本命令参考

```shell
# 添加仓库
helm repo add 
# 查询 charts
helm search repo
# 更新repo仓库资源
helm repo update
# 查看当前安装的charts
helm list -A
# 安装
helm install 
# 卸载
helm uninstall
# 更新
helm upgrade
```

## 安装RabbitMQ

### 下载chart包

```shell
# 添加bitnami仓库
helm repo add bitnami https://charts.bitnami.com/bitnami

# 查询chart
helm search repo bitnami

# 创建工作目录
mkdir -p ~/test/rabbitmq
cd ~/test/rabbitmq

# 拉取rabbitmq
helm pull bitnami/rabbitmq

# 解压
tar zxvf [rabbitmq] 
```

![image-20210725143655097](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061343211.png)

### 配置参数

#### 编辑配置文件

> 官方配置参考：https://github.com/bitnami/charts/tree/master/bitnami/rabbitmq

- 进入工作目录，配置持久化存储、副本数等
- 建议首次部署时直接修改values中的配置，而不是用--set的方式，这样后期upgrade不必重复设置。

```shell
cd  ~/test/rabbitmq/rabbitmq
vim values.yaml
```

#### 设置管理员密码

- 方式一：在配置中指定

```yaml
auth:
  username: admin
  password: "admin@mq"
  existingPasswordSecret: ""
  erlangCookie: secretcookie
```

![image-20210725144921881](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061343223.png)

- 方式二：在安装时通过set方式指定（避免密码泄露）

```shell
--set auth.username=admin,auth.password=admin@mq,auth.erlangCookie=secretcookie
```

#### rabbitmq集群意外宕机强制启动

- 当rabbitmq启用持久化存储时，若rabbitmq所有pod同时宕机，将无法重新启动，因此有必要提前开启`clustering.forceBoot`

```yaml
clustering:
  enabled: true
  addressType: hostname
  rebalance: false
  forceBoot: true
```

![image-20210725153657694](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061343260.png)

#### 模拟rabbitmq集群宕机（可跳过）

- 未设置`clustering.forceBoot`时，如下图，通过删除rabbitmq集群所有pod模拟宕机，可见集群重新启动时第一个节点迟迟未就绪

![image-20210725154535139](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061343250.png)

- 报错信息如下：

![image-20210725154742499](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061343246.png)

- 启用`clustering.forceBoot`，并更新rabbitmq，可见集群重启正常

```shell
helm upgrade rabbitmq -n test .
kubectl delete pod -n test rabbitmq-0
get pod -n test -w
```

![image-20210725155446725](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061343272.png)

#### 指定时区

```yaml
extraEnvVars: 
  - name: TZ
    value: "Asia/Shanghai"
```

![image-20210725145746565](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061343591.png)

#### 指定副本数

```yaml
replicaCount: 3
```

![image-20210725153715021](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061343593.png)

#### 设置持久化存储

- 若无需持久化，将`enabled`设置为`false`

- 持久化需使用块存储，本文通过aws的ebs-csi创建storageClass，亦可使用自建块存储storageClass

> 注：sc最好具备扩容属性

```yaml
persistence:
  enabled: true
  storageClass: "ebs-sc"
  selector: {}
  accessMode: ReadWriteOnce
  existingClaim: ""
  size: 8Gi
```

![image-20210725153802203](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061343623.png)

#### 设置service

- 默认通过ClusterIP暴露5672（amqp）和15672（web管理界面）等端口供集群内部使用，外部访问方式将在第三章中详细说明
- 不建议在values中直接配置nodeport，不方便后期灵活配置

### 部署RabbitMQ

#### 创建命名空间

```shell
cd  ~/test/rabbitmq/rabbitmq
kubectl create ns test 
```

#### 安装

- 方式一 ：在配置文件中指定管理员帐号密码

```shell
helm install rabbitmq -n test .
```

![image-20210725151045382](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061343633.png)

- 方式二：通过set方式指定密码

```shell
helm install rabbitmq -n test . \
  --set auth.username=admin,auth.password=admin@mq,auth.erlangCookie=secretcookie
```

> 后期**upgrade**时亦须指定上述参数

#### 查看rabbitmq安装状态

- 查看rabbitmq安装进度

```shell
kubectl get pod -n test -w
```

- 待各节点都正常启动后，查看svc

```shell
kubectl get svc -n test
```

![image-20210725151643559](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061343597.png)

> 当前rabbitmq通过ClusterIP方式暴露，供集群内部访问；外部访问方式将在下章介绍。

- 查看集群状态

```shell
# 进入pod
kubectl exec -it -n test rabbitmq-0 -- bash

# 查看集群状态
rabbitmqctl cluster_status

# 列出策略（尚未设置镜像模式）
rabbitmqctl list_policies

#设置集群名称
rabbitmqctl set_cluster_name [cluster_name]
```

## 配置RabbitMQ集群外部访问方式

### 建议方式

- 不建议在默认安装方式中指定nodeport，而是另外创建

- 5672：建议通过`service-私网负载均衡器`暴露给私网其它应用使用
- 15672：建议通过`ingress`或`service-公网负载均衡器`暴露给外界访问

| 端口  | 暴露方式（见下文方式三）                     | 访问方式                                                     |
| ----- | -------------------------------------------- | ------------------------------------------------------------ |
| 5672  | Service-LoadBalancer（配置为私网负载均衡器） | k8s集群内：rabbitmq.test:5672<br />私网：私网负载均衡IP:5672 |
| 15672 | ingress-ALB（配置为公网负载均衡器）          | 公网负载均衡URL                                              |

> 注：本文使用亚马逊托管版k8s集群，已配置`aws-load-balancer-controller`

### 方式一：Service-Nodeport（5672，15672）

- 获取原始Service-ClusterIP的yaml文件：

```shell
cd /test/rabbitmq
kubectl get svc -n test rabbitmq -o yaml > service-clusterip.yaml
```

- 参考service-clusterip，创建service-nodeport.yaml

```shell
cp service-clusterip.yaml service-nodeport.yaml
```

- 配置service-nodeport（去除多余信息）

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq-nodeport
  namespace: test
spec:
  ports:
  - name: amqp
    port: 5672
    protocol: TCP
    targetPort: amqp
    nodePort: 32672
  - name: http-stats
    port: 15672
    protocol: TCP
    targetPort: stats
    nodePort: 32673
  selector:
    app.kubernetes.io/instance: rabbitmq
    app.kubernetes.io/name: rabbitmq
  type: NodePort
```

- 创建service

```shell
kubectl apply -f service-nodeport.yaml
kubectl get svc -n test
```

![image-20210725165223728](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061343610.png)

- 即可通过NodeIP:Port访问服务。

### 方式二：Service-公网LoadBalancer（5672，15672）

- 创建`service-loadbalancer.yaml`：

```shell
vim service-loadbalancer.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq-loadbalance
  namespace: test
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
spec:
  ports:
  - name: amqp
    port: 5672
    protocol: TCP
    targetPort: amqp
  - name: http-stats
    port: 15672
    protocol: TCP
    targetPort: stats
  selector:
    app.kubernetes.io/instance: rabbitmq
    app.kubernetes.io/name: rabbitmq
  type: LoadBalancer
```

- 创建service：

```shell
kubectl apply -f service-loadbalancer.yaml
kubectl get svc -n test
```

![image-20210725171840468](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061343875.png)

> **浏览器登录控制台**: http://k8s-test-rabbitmq-fbff138068-d79b31bdcb2f6f2b.elb.cn-northwest-1.amazonaws.com.cn:15672：

![image-20210725172013248](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061343937.png)

### 方式三：Service-私网LoadBalancer（5672）+Ingress-公网ALB（15672）

#### 创建Service-私网LoadBalancer

```shell
vim service-lb-internal.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq-lb-internal
  namespace: test
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    # service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing #注释后即为私网
spec:
  ports:
  - name: amqp
    port: 5672
    protocol: TCP
    targetPort: amqp
  selector:
    app.kubernetes.io/instance: rabbitmq
    app.kubernetes.io/name: rabbitmq
  type: LoadBalancer
```

```shell
kubectl apply -f service-lb-internal.yaml
```

#### 创建Ingress-ALB

```shell
vim ingress-alb.yaml
```

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: rabbitmq
  namespace: test
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
  labels:
    app: rabbitmq
spec:
  rules:
    - http:
        paths:
          - path: /*
            backend:
              serviceName: "rabbitmq"
              servicePort: 15672
```

```shell
kubectl apply -f ingress-alb.yaml
```

![image-20210725172536738](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061343895.png)

> **浏览器登录控制台**: k8s-test-rabbitmq-4623cb772f-1674334573.cn-northwest-1.elb.amazonaws.com.cn

![image-20210725172921696](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061343957.png)

## 配置镜像模式实现高可用

### 镜像模式介绍

&emsp;&emsp; 镜像模式：将需要消费的队列变为镜像队列，存在于多个节点，这样就可以实现 RabbitMQ 的 HA 高可用性。作用就是消息实体会主动在镜像节点之间实现同步，而不是像普通模式那样，在 consumer 消费数据时临时读取。缺点就是，集群内部的同步通讯会占用大量的网络带宽。

### rabbitmqctl设置镜像模式

```shell
# 进入pod
kubectl exec -it -n test rabbitmq-0 -- bash

# 列出策略（尚未设置镜像模式）
rabbitmqctl list_policies

# 设置镜像模式（仅“/”vhost）
rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all" , "ha-sync-mode":"automatic"}'

# 指定vhost
rabbitmqctl set_policy -p <自定义vhost> ha-all "^" '{"ha-mode":"all" , "ha-sync-mode":"automatic"}'

# 再次列出策略
rabbitmqctl list_policies
```

![image-20210725173209193](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061343943.png)

> 控制台查看

![image-20210725173257368](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061343944.png)

## 清理RabbitMQ集群

### 卸载RabbitMQ

```shell
helm uninstall rabbitmq -n test
```

### 删除pvc

```shell
kubectl delete pvc -n test data-rabbitmq-0 data-rabbitmq-1 data-rabbitmq-2
```

### 清理手动创建的service，ingress

```shell
kubectl delete -f service-nodeport.yaml
kubectl delete -f service-loadbalancer.yaml
kubectl delete -f ingress-alb.yaml
```