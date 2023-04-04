# K8S中部署分布式Jmeter压测环境

> 参考：[jmeter-on-k8s](https://github.com/qqian1991/jmeter-on-k8s)，基于此项目设计的精简版

## 架构说明

jmeter-maste与jmeter-slave使用相同的镜像，区别是启动命令不同。

- slave：启动jmeter-server进程，并开启远程端口
- master：通过-R将任务分配至所有slave节点。

## 部署jmeter-slave

副本数可任意设置，注意资源限制大小，依主机资源自行调节

**注：并发数 = num_threads × slave副本数 / ramp_time**

```shell
vim slave.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jmeter-slaves
  labels:
    jmeter_mode: slave
spec:
  replicas: 2
  selector:
    matchLabels:
      jmeter_mode: slave
  template:
    metadata:
      labels:
        jmeter_mode: slave
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: jmeter_mode
                  operator: In
                  values:
                  - slave
              topologyKey: "kubernetes.io/hostname"
      containers:
      - name: jmslave
        image: justb4/jmeter
        imagePullPolicy: IfNotPresent
        env:
          - name: TZ
            value: Asia/Shanghai       
        lifecycle:
          postStart:
            exec:
              command: ["/bin/sh","-c","mkdir -p /jmeter/jmx"]
        command: 
          - "jmeter-server"
          - "-Dserver.rmi.localport=50000"
          - "-Dserver_port=1099"
          - "-Jserver.rmi.ssl.disable=true"
        ports:
        - containerPort: 1099
        - containerPort: 50000
        resources:
          limits:
            cpu: 1
          requests:
            cpu: 0.5
            memory: 0.5Gi
---
apiVersion: v1
kind: Service
metadata:
  name: jmeter-slaves-svc
  labels:
    jmeter_mode: slave
spec:
  clusterIP: None
  ports:
    - port: 1099
      name: first
      targetPort: 1099
    - port: 50000
      name: second
      targetPort: 50000
  selector:
    jmeter_mode: slave
```

```shell
kubectl apply -f slave.yaml
```

## 部署master

将jmeter-master与nginx部署在同一pod中，使用emptyDir共享web目录，并通过nodeport暴露，方便查询压测结果。

```shell
vim master.yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: jmeter-run
  labels:
    app: jmeter-run
data:
  run.sh: |
    #!/bin/bash
    rm -rf /jemter/jtl/test.jtl /jmeter/web/*
    jmeter -Dserver.rmi.ssl.disable=true -n -t /jmeter/jmx/test.jmx -l /jemter/jtl/test.jtl -e -o /jmeter/web/ \
      -R `getent ahostsv4 jmeter-slaves-svc | cut -d' ' -f1 | sort -u | awk -v ORS=, '{print $1}' | sed 's/,$//'`
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jmeter-master
  labels:
    jmeter_mode: master
spec:
  replicas: 1
  selector:
    matchLabels:
      jmeter_mode: master
  template:
    metadata:
      labels:
        jmeter_mode: master
    spec:
      containers:
      - name: jmmaster
        image: justb4/jmeter
        imagePullPolicy: IfNotPresent
        env:
          - name: TZ
            value: Asia/Shanghai       
        lifecycle:
          postStart:
            exec:
              command: ["/bin/bash","-c","mkdir -p /jmeter/{jmx,jtl,web}"]
        command: [ "/bin/bash", "-c", "--" ]
        args: [ "while true; do sleep 30; done;" ]
        ports:
        - containerPort: 60000
        resources:
          limits:
            cpu: 2
          requests:
            cpu: 0.5
            memory: 0.5Gi
        volumeMounts:
          - name: jmeter-cm
            mountPath: /run.sh
            subPath: run.sh
          - name: shared-data
            mountPath: /jmeter/web
      - name: nginx
        image: nginx:stable-alpine
        readinessProbe:
          tcpSocket:
            port: 80
          initialDelaySeconds: 5      # 延迟加载时间
          periodSeconds: 10             # 重试时间间隔
          failureThreshold: 6          # 不健康阈值       
        volumeMounts:
          - name: shared-data
            mountPath: /usr/share/nginx/html
      volumes:
      - name: jmeter-cm
        configMap:
         name: jmeter-run
      - name: shared-data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: jmeter-nginx
  namespace: default
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  selector:
    jmeter_mode: master 
```

```shell
kubectl apply -f master.yaml
```

## 压测执行脚本

### start_test.sh

```shell
vim start_test.sh
```

```shell
#!/bin/bash

jmx="api_list.jmx"  # jmx压测文件
csv="data_list.csv" # csv文件，在jmx中路径为/jmeter/jmx/data_list；若无，同时注释掉Step2

echo "Step1 拷贝jmx至master"
master_pod=`kubectl get po | grep jmeter-master | grep Running | awk '{print $1}'`
kubectl cp "$jmx" -c jmmaster  "$master_pod:/jmeter/jmx/test.jmx"

echo "Step2 拷贝csv至slave"
slave_pod=`kubectl get po | grep jmeter-slave | grep Running | awk '{print $1}'`
for i in $slave_pod;do
  kubectl cp "$csv"  "$i:/jmeter/jmx/$csv"
done;

echo "Step3 开启压测"
kubectl exec -ti  $master_pod -c jmmaster -- bash /run.sh

echo "Step4 查询压测结果："
echo "http://<外网IP>:30080/test/"
```

```shell
chmod +x start_test.sh
```

### restart.sh

如果压测异常，执行此脚本重启jmeter POD

```shell
vim restart.sh
```

```shell
#!/bin/bash
kubectl delete pod -l jmeter_mode=master --force --grace-period=0
kubectl delete pod -l jmeter_mode=slave --force --grace-period=0
```

```shell
chmod +x restart.sh
```

## 测试

- 准备好jmx和csv压测文件后

**注：并发数 = num_threads × slave副本数 / ramp_time**

![image-20220313210402894](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203132104933.png)

- 执行：

```shell
./start_test.sh
```

![image-20220313210349724](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203132103783.png)

- 查看结果

![image-20220313210634850](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203132106036.png)

