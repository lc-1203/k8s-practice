- **Step 1:** 创建一个无状态应用[k8s deployment](https://v1-14.docs.kubernetes.io/docs/concepts/workloads/controllers/deployment)，运行SRS源站服务器：

  ```
  vim srs-deploy.yaml
  
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: srs-deploy
    labels:
      app: srs
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: srs
    template:
      metadata:
        labels:
          app: srs
      spec:
        containers:
        - name: srs
          image: ossrs/srs:3
          
  # kubectl create -f srs-deploy.yaml
  ```

- **Step 2:** 创建一个服务[k8s service](https://v1-14.docs.kubernetes.io/docs/concepts/services-networking/service)，自动创建[SLB](https://www.aliyun.com/product/slb)和[EIP](https://www.aliyun.com/product/eip)，对外提供流媒体服务：

  ```
  # vim srs-origin-service.yaml
  
  apiVersion: v1
  kind: Service
  metadata:
    name: srs-origin-service
  spec:
    type: NodePort
    selector:
      app: srs
    ports:
    - name: srs-origin-service-1935-1935
      port: 1935
      protocol: TCP
      targetPort: 1935
    - name: srs-origin-service-1985-1985
      port: 1985
      protocol: TCP
      targetPort: 1985
    - name: srs-origin-service-8080-8080
      port: 8080
      protocol: TCP
      targetPort: 8080
      
  # kubectl create -f srs-origin-service.yaml
  ```

- **Step 3: **查询服务的EIP地址，测试推拉流：

  ```
  kubectl get svc/srs-origin-service
  
  obs推流：rtmp://8.135.19.91:32587/live/12345
  拉流：rtmp://8.135.19.91:32587/live/12345
  ffplay拉流:
  ffplay -i "rtmp://8.135.19.91:32587/live/12345" -fflags nobuffer -vf scale=960:540
  ```

  <img src="https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20201225113028.png" alt="image-20201225113028212" style="zoom: 50%;" />

创建ingress

```
# vim srs-ingress.yaml

apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: srs-svc-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: live-test.com
    http:
      paths:
      - path: /rtmp
        backend:
          serviceName: srs-origin-service
          servicePort: 1935
      - path: /web
        backend:
          serviceName: nginx-work-svc
          servicePort: 80
        
      
          
    
# kubectl create -f srs-ingress.yaml
```

安装nginx

```
kubectl label nodes k8s-node1 deploy.type=nginx-work

vim nginx-work.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-work
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-work
  template:
    metadata:
      labels:
        app: nginx-work
    spec:
      containers:
      - name: nginx-work
        image: nginx:1.18 
    nodeSelectot:
      deploy.type: nginx-work
```

```
# vim nginx-svc.yaml

apiVersion: v1      	# api版本
kind: Service			# 资源类型
metadata:
  name: nginx-work-svc		# 元数据名称
  namespace: default	# 命名空间
spec:
  type: NodePort	  	# 服务类型
  selector:				# 标签选择器
    app: nginx-work	    # 指定关联的标签
  ports:
    - protocol: TCP		# 协议
      port: 80			# Service端口(服务发现)
      targetPort: 80	# 容器端口（负载均衡）
```

