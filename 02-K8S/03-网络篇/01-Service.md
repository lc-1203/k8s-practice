## Service的定义

&emsp;&emsp; Service是Kubernetes的核心概念，通过创建Service，可以为一组具有相同功能的容器应用提供一个统一的入口地址，并且将请求负载分发到后端的各个容器应用上。

## Service存在的意义

&emsp;&emsp; 问题描述： 如果一组 Pod（称为“后端”）为集群内的其他 Pod（称为“前端”）提供功能， 那么前端如何找出并跟踪要连接的 IP 地址，以便前端可以使用工作量的后端部分？

&emsp;&emsp; 直接通过Pod的IP地址和端口号可以访问到容器应用内的服务，但是Pod的IP地址是不可靠的，例如当Pod所在的Node发生故障时，Pod将被Kubernetes重新调度到另一个Node，Pod的IP地址将发生变化。更重要的是，如果容器应用本身是分布式的部署方式，通过多个实例共同提供服务，就需要在这些实例的前端设置一个负载均衡器来实现请求的分发。

可见，k8s中Service的引入主要是解决Pod的动态变化，提供统一访问入口：

- 防止Pod失联（服务发现）
- 定义一组Pod的访问策略（负载均衡）

## Pod与Service的关系

- Service通过标签关联一组Pod

- Service使用iptables或者ipvs为一组Pod提供负载均衡能力

  <img src="https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061107928.png" alt="image-20201226141804803" style="zoom: 80%;" />

## Service的定义与创建

### 前提：创建Deployment

```
# vim nginx-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment   # deployment的名称
  namespace: default
spec:
  replicas: 1				# Pod副本预期数量
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx			# Pod标签
    spec:
      containers:
      - name: nginx			# 容器名称
        image: nginx:1.15
```

### 创建Deployment并查看关联的pod标签

```
# 创建
kubectl create -f nginx-deployment.yaml

# 查看关联的pod标签
kubectl get pods --show-labels
```

![image-20201226160656175](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061107055.png)

### 服务暴露

#### 方式一：快速暴露

```
kubectl expose deployment nginx-deployment --port=80 --target-port=80 --type=NodePort
```

#### 方式二：创建Service对象

```
# vim nginx-svc.yaml

apiVersion: v1      	# api版本
kind: Service			# 资源类型
metadata:
  name: nginx-svc		# 元数据名称
  namespace: default	# 命名空间
spec:
  type: NodePort	  	# 服务类型
  selector:				# 标签选择器
    app: nginx			# 指定关联的标签
  ports:
    - protocol: TCP		# 协议
      port: 80			# Service端口(服务发现)
      targetPort: 80	# 容器端口（负载均衡）

# 需要注意的是，Service 能够将一个接收 port 映射到任意的 targetPort。 
# 默认情况下，targetPort 将被设置为与 port 字段相同的值。
```

&emsp;&emsp; 上述配置创建一个名称为 "nginx-svc" 的 Service 对象，指定了Service所需的虚拟端口号为80（port）, 它会将请求代理到使用 TCP 端口80（targetPort），并且具有标签 `"app=nginx"` 的 Pod 上。

### 创建该Service并查看IP地址和端口号

```
# 创建
kubectl create -f nginx-svc.yaml

# 查看Service
kubectl get svc

# 查看关联的pod
kubectl get endpoints 
```

![image-20201226160923695](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061107969.png)

### 通过Service的IP地址和端口号进行访问

&emsp;&emsp; 创建Service时指定了服务类型为NodePort，可将Service的端口号映射到映射到物理机，因此可通过物理机的IP地址和生成的NodePort访问Service。无论内部访问还是外部访问，对该Service的访问都将被负载分发到后端关联的多个Pod上。

#### 集群内部访问

```
# 内部访问：curl CLUSTER-IP:port
curl 10.0.0.82:80
```

![image-20201226161221640](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061107023.png)

#### 外部访问

```
# 外部访问：NodeIP（公网）:NodePort
浏览器输入：http://8.135.19.91:30304/
```

![image-20201226161439804](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061107027.png)

## Service的三种常用类型

- ClusterIP：集群内部使用
- NodePort：对外暴露引用（集群外）
- LoadBalancer：对外暴露应用，使用公有云

###  ClusterIP

&emsp;&emsp; 默认类型，分配一个稳定的IP地址，即VIP，只能在集群内部访问。

![image-20201231142238512](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061107934.png)

### NodePort

&emsp;&emsp; 在每个节点上启用一个端口暴露服务，可以在集群外部访问。也会默认分配一个稳定的内部集群IP地址。

![image-20201231142256663](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061107375.png)

&emsp;&emsp; 访问地址：任意NodeIP：NodePort

&emsp;&emsp; 端口范围：30000-32767

- 查询端口监听信息

```
ss -anpt |grep <Port>
```

&emsp;&emsp; NodePort会在每台Node上监听端口接受用户流量，在实际情况下，对用户暴露的只会有一个IP和端口，因此，需要配置公网负载均衡器为项目提供统一访问入口。

![image-20201231142307998](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061107355.png)

### LoadBalancer

&emsp;&emsp; 与NodePort类似，在每个节点上启用一个端口来暴露服务。除此之外，Kubernetes会请求底层云平台（例如阿里云、腾讯云）上的负载均衡器，将每个Node（NodeIP：NodePort）作为后端添加进去。

![image-20201231142319638](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061107362.png)

### kube-proxy运行机制解析

&emsp;&emsp; 在Kubernetes集群的每个Node上都会运行一个kube-proxy服务进程，我们可以把这个进程看作Service的透明代理兼负载均衡器，其核心功能是将到某个Service的访问请求转发到后端的多个Pod实例上。此外，Service的Cluster IP与NodePort等概念是kube-proxy服务通过iptables的NAT转换实现的，kube-proxy在运行过程中动态创建与Service相关的iptables规则，这些规则实现了将访问服务（Cluster IP或NodePort）的请求负载分发到后端Pod的功能。由于iptables机制针对的是本地的kube-proxy端口，所以在每个Node上都要运行kube-proxy组件，这样一来，在Kubernetes集群内部，我们可以在任意Node上发起对Service的访问请求。综上所述，由于kube-proxy的作用，在Service的调用过程中客户端无须关心后端有几个Pod，中间过程的通信、负载均衡及故障恢复都是透明的。

## Service代理模式

### iptables与IPVS（实战代码待补充）

&emsp;&emsp; Service的底层实现主要有iptables和ipvs两种网络模式，决定了如何转发流量

&emsp;&emsp; 流程：客户端-->NodePort/ClusterIP(iptables/IPVS负载均衡规则)-->分发在各节点的Pod

![image-20210104091331635](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203061107455.png)

- Iptables
  - 数据包经过iptables规则匹配，重定向到另一个链路
  - 该链关联到一组规则，有几个pod就有几条规则（负载均衡），以概率的方式转发到不同的链
  - 使用DNAT转发到具体的pod中
- IPVS



- 对比：

  iptables：

  - 灵活，功能强大
  - 规则遍历匹配和更新，呈线性时延

  IPVS

  - 工作在内核态，有更好的性能
  - 调度算法丰富：rr，wrr，lc，wlc，ip hash...

### Service DNS名称解析

&emsp;&emsp; IP地址是动态的，pod如果重启后，地址会变化

&emsp;&emsp; CoreDNS：是一个DNS服务器，Kubernetes默认采用，以Pod部署在集群中，CoreDNS服务监视Kubernetes API，为每一个Service创建DNS记录用于域名解析。