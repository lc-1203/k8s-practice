![ACK: SRS Edge Cluster for High Concurrency Streaming](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210109145201.png)



**Step 1:** 创建SRS和Nginx源站应用和服务。

- `srs-origin-deploy`: 创建一个无状态应用k8s deployment，运行SRS Origin Server和Nginx，HLS写入共享Volume：
- `srs-origin-service`: 创建一个服务k8s service，基于ClusterIP提供Origin服务，供内部Edge Server调用。
- `srs-http-service`: 创建一个服务k8s service，基于SLB提供HTTP服务，Nginx对外提供HLS服务。



```
# vim srs-origin-3.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: srs-origin-deploy
  labels:
    app: srs-origin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: srs-origin
  template:
    metadata:
      labels:
        app: srs-origin
    spec:
      volumes:
      - name: cache-volume
        emptyDir: {}
      containers:
      - name: srs
        image: ossrs/srs:3
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 1935
        - containerPort: 1985
        - containerPort: 8080
        volumeMounts:
        - name: cache-volume
          mountPath: /usr/local/srs/objs/nginx/html
          readOnly: false
      - name: nginx
        image: nginx
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
        volumeMounts:
        - name: cache-volume
          mountPath: /usr/share/nginx/html
          readOnly: true
      - name: srs-cp-files
        image: ossrs/srs:3
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: cache-volume
          mountPath: /tmp/html
          readOnly: false
        command: ["/bin/sh"]
        args:
        - "-c"
        - >
          if [[ ! -f /tmp/html/index.html ]]; then
            cp -R ./objs/nginx/html/* /tmp/html
          fi &&
          sleep infinity

---

apiVersion: v1
kind: Service
metadata:
  name: srs-origin-service
spec:
  type: ClusterIP
  selector:
    app: srs-origin
  ports:
  - name: srs-origin-service-1935-1935
    port: 1935
    protocol: TCP
    targetPort: 1935

---

apiVersion: v1
kind: Service
metadata:
  name: srs-http-service
spec:
  type: NodePort
  selector:
    app: srs-origin
  ports:
  - name: srs-http-service-80-80
    port: 80
    protocol: TCP
    targetPort: 80
  - name: srs-http-service-1985-1985
    port: 1985
    protocol: TCP
    targetPort: 1985
          
# kubectl apply -f srs-origin-3.yaml
```





**Step 2:** 创建SRS边缘配置、应用和服务。

- `srs-edge-config`: 创建一个配置k8s ConfigMap，存储了SRS Edge Server使用的配置文件。
- `srs-edge-deploy`: 创建一个无状态应用k8s deployment，运行多个SRS Edge Server。
- `srs-edge-service`: 创建一个服务k8s service基于SLB对外提供流媒体服务。

```
# vim srs-edge-3.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: srs-edge-config
data:
  srs.conf: |-
    listen              1935;
    max_connections     1000;
    daemon              off;
    http_api {
        enabled         on;
        listen          1985;
    }
    http_server {
        enabled         on;
        listen          8080;
    }
    vhost __defaultVhost__ {
        cluster {
            mode            remote;
            origin          srs-origin-service;
        }
        http_remux {
            enabled     on;
        }
    }

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: srs-edge-deploy
  labels:
    app: srs-edge
spec:
  replicas: 3
  selector:
    matchLabels:
      app: srs-edge
  template:
    metadata:
      labels:
        app: srs-edge
    spec:
      nodeSelector:
        srs-roles: edge
      volumes:
      - name: config-volume
        configMap:
          name: srs-edge-config
      containers:
      - name: srs
        image: ossrs/srs:3
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 1935
        - containerPort: 1985
        - containerPort: 8080
        volumeMounts:
        - name: config-volume
          mountPath: /usr/local/srs/conf

---

apiVersion: v1
kind: Service
metadata:
  name: srs-edge-service
spec:
  type: NodePort
  selector:
    app: srs-edge
  ports:
  - name: srs-edge-service-1935-1935
    port: 1935
    protocol: TCP
    targetPort: 1935
  - name: srs-edge-service-8080-8080
    port: 8080
    protocol: TCP
    targetPort: 8080
    
    
# kubectl apply -f srs-edge-3.yaml    
```



**Step 2:** 创建一个服务k8s service，使用SLB对外提供流媒体服务：

```
# vim srs-nginx-svc.yaml

apiVersion: v1
kind: Service
metadata:
  name: srs-origin-service
spec:
  type: NodePort
  selector:
    app: srs
  ports:
  - name: srs-origin-service-80-80
    port: 80
    protocol: TCP
    targetPort: 80
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
    
# kubectl apply -f srs-nginx-svc.yaml
```



**Step 3:** Publish Demo Streams to SRS

```
vim srs-demo.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: srs-demo-deploy
  labels:
    app: srs-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: srs-demo
  template:
    metadata:
      labels:
        app: srs-demo
    spec:
      containers:
      - name: encoder
        image: ossrs/srs:encoder
        imagePullPolicy: IfNotPresent
        command: ["/bin/sh"]
        args:
        - "-c"
        - >
          while true; do
            ffmpeg -re -i ./doc/source.200kbps.768x320.flv \
              -c copy -f flv rtmp://srs-origin-service/live/test-demo;
            sleep 3;
          done

# kubectl apply -f srs-demo.yaml
```



**Step 3:** 推流、拉流

```
Publish RTMP to rtmp://8.135.19.91:30600/live/123 
Play RTMP from rtmp://8.135.19.91:30600/live/test-demo
Play HTTP-FLV from http://8.135.19.91:30366/live/test-demo.flv
Play HLS from http://8.135.19.91:30366/live/test-demo.m3u8
Play HLS from http://8.135.19.91:31232/live/test-demo.m3u8
```

