# K8s + SpringBoot实现零宕机发布


## 前言

快速配置请直接跳转至[汇总配置](##汇总配置)

K8s + SpringBoot实现零宕机发布：健康检查+滚动更新+优雅停机+弹性伸缩+Prometheus监控+配置分离（镜像复用）

---
## 配置

### 健康检查

- 健康检查类型：就绪探针（readiness）+ 存活探针（liveness）

- 探针类型：exec（进入容器执行脚本）、tcpSocket（探测端口）、httpGet（调用接口）

#### 业务层面

1. 项目依赖 `pom.xml`

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```

2. 定义访问端口、路径及权限 `application.yaml`

```yaml
management:
  server:
    port: 50000                         # 启用独立运维端口
  endpoint:                             # 开启health端点
    health:
      probes:
        enabled: true
  endpoints:
    web:
      exposure:
        base-path: /actuator            # 指定上下文路径，启用相应端点
        include: health
```

将暴露`/actuator/health/readiness`和`/actuator/health/liveness`两个接口，访问方式如下：

```
http://127.0.0.1:50000/actuator/health/readiness
http://127.0.0.1:50000/actuator/health/liveness
```

#### 运维层面

- k8s部署模版`deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: {APP_NAME}
        image: {IMAGE_URL}
        imagePullPolicy: Always
        ports:
        - containerPort: {APP_PORT}
        - name: management-port
          containerPort: 50000         # 应用管理端口
        readinessProbe:                # 就绪探针
          httpGet:
            path: /actuator/health/readiness
            port: management-port
          initialDelaySeconds: 30      # 延迟加载时间
          periodSeconds: 10            # 重试时间间隔
          timeoutSeconds: 1            # 超时时间设置
          successThreshold: 1          # 健康阈值
          failureThreshold: 6          # 不健康阈值
        livenessProbe:                 # 存活探针
          httpGet:
            path: /actuator/health/liveness
            port: management-port
          initialDelaySeconds: 30      # 延迟加载时间
          periodSeconds: 10            # 重试时间间隔
          timeoutSeconds: 1            # 超时时间设置
          successThreshold: 1          # 健康阈值
          failureThreshold: 6          # 不健康阈值
```

### 滚动更新

k8s资源调度之滚动更新策略，若要实现零宕机发布，需支持健康检查

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {APP_NAME}
  labels:
    app: {APP_NAME}
spec:
  selector:
    matchLabels:
      app: {APP_NAME}
  replicas: {REPLICAS}				# Pod副本数
  strategy:
    type: RollingUpdate				# 滚动更新策略
    rollingUpdate:
      maxSurge: 1                   # 升级过程中最多可以比原先设置的副本数多出的数量
      maxUnavailable: 1             # 升级过程中最多有多少个POD处于无法提供服务的状态
```

### 优雅停机

在K8s中，当我们实现滚动升级之前，务必要实现应用级别的优雅停机。否则滚动升级时，还是会影响到业务。使应用关闭线程、释放连接资源后再停止服务

#### 业务层面

1. 项目依赖 `pom.xml`

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```

2. 定义访问端口、路径及权限 `application.yaml`

```yaml
spring:
  application:
    name: <xxx>
  profiles:
    active: @profileActive@
  lifecycle:
    timeout-per-shutdown-phase: 30s     # 停机过程超时时长设置30s，超过30s，直接停机

server:
  port: 8080
  shutdown: graceful                    # 默认为IMMEDIATE，表示立即关机；GRACEFUL表示优雅关机

management:
  server:
    port: 50000                         # 启用独立运维端口
  endpoint:                             # 开启shutdown和health端点
    shutdown:
      enabled: true
    health:
      probes:
        enabled: true
  endpoints:
    web:
      exposure:
        base-path: /actuator            # 指定上下文路径，启用相应端点
        include: health,shutdown
```

将暴露`/actuator/shutdown`接口，调用方式如下：

```shell
curl -X POST 127.0.0.1:50000/actuator/shutdown
```

#### 运维层面

- 确保dockerfile模版集成curl工具，否则无法使用curl命令

```dockerfile
FROM openjdk:8-jdk-alpine
#构建参数
ARG JAR_FILE
ARG WORK_PATH="/app"
ARG EXPOSE_PORT=8080

#环境变量
ENV JAVA_OPTS=""\
    JAR_FILE=${JAR_FILE}

#设置时区
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo 'Asia/Shanghai' >/etc/timezone
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories  \
    && apk add --no-cache curl
#将maven目录的jar包拷贝到docker中，并命名为for_docker.jar
COPY target/$JAR_FILE $WORK_PATH/


#设置工作目录
WORKDIR $WORK_PATH


# 指定于外界交互的端口
EXPOSE $EXPOSE_PORT
# 配置容器，使其可执行化
ENTRYPOINT exec java $JAVA_OPTS -jar $JAR_FILE
```

- k8s部署模版`deployment.yaml`

注：经验证，java项目可省略结束回调钩子的配置

此外，若需使用回调钩子，需保证镜像中包含curl工具，且需注意应用管理端口（50000）不能暴露到公网

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: {APP_NAME}
        image: {IMAGE_URL}
        imagePullPolicy: Always
        ports:
        - containerPort: {APP_PORT}
        - containerPort: 50000
        lifecycle:
          preStop: 						# 结束回调钩子
            exec:
              command: ["curl", "-XPOST", "127.0.0.1:50000/actuator/shutdown"]
```

### 弹性伸缩

为pod设置资源限制后，创建HPA

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {APP_NAME}
  labels:
    app: {APP_NAME}
spec:
  template:
    spec:
      containers:
      - name: {APP_NAME}
        image: {IMAGE_URL}
        imagePullPolicy: Always
        resources:                     # 容器资源管理
          limits:                      # 资源限制（监控使用情况）
            cpu: 0.5
            memory: 1Gi
          requests:                    # 最小可用资源（灵活调度）
            cpu: 0.15
            memory: 300Mi
---
kind: HorizontalPodAutoscaler            # 弹性伸缩控制器
apiVersion: autoscaling/v2beta2
metadata:
  name: {APP_NAME}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {APP_NAME}
  minReplicas: {REPLICAS}                # 缩放范围
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu                        # 指定资源指标
        target:
          type: Utilization
          averageUtilization: 50
```

### Prometheus集成

#### 业务层面

1. 项目依赖 `pom.xml`

```xml
        <!-- 引入Spring boot的监控机制-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
        </dependency>
```

2. 定义访问端口、路径及权限 `application.yaml`

```yaml
management:
  server:
    port: 50000                         # 启用独立运维端口
  metrics:
    tags:
      application: ${spring.application.name}
  endpoints:
    web:
      exposure:
        base-path: /actuator            # 指定上下文路径，启用相应端点
        include: metrics,prometheus
```

将暴露`/actuator/metric`和`/actuator/prometheus`接口，访问方式如下：

```shell
http://127.0.0.1:50000/actuator/metric
http://127.0.0.1:50000/actuator/prometheus
```

#### 运维层面

- `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        prometheus:io/port: "50000"
        prometheus.io/path: /actuator/prometheus  # 在流水线中赋值
        prometheus.io/scrape: "true"              # 基于pod的服务发现
```

### 配置分离

方案：通过configmap挂载外部配置文件，并指定激活环境运行

作用：配置分离，避免敏感信息泄露；镜像复用，提高交付效率

- 通过文件生成configmap

```shell
# 通过dry-run的方式生成yaml文件
kubectl create cm -n <namespace> <APP_NAME> --from-file=application-test.yaml --dry-run=1 -oyaml > configmap.yaml

# 更新
kubectl apply -f configmap.yaml
```

- 挂载configmap并指定激活环境

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {APP_NAME}
  labels:
    app: {APP_NAME}
spec:
  template:
    spec:
      containers:
      - name: {APP_NAME}
        image: {IMAGE_URL}
        imagePullPolicy: Always
        env:
          - name: SPRING_PROFILES_ACTIVE   # 指定激活环境
            value: test
        volumeMounts:                      # 挂载configmap
        - name: conf
          mountPath: "/app/config"         # 与Dockerfile中工作目录一致
          readOnly: true
      volumes:
      - name: conf
        configMap:
          name: {APP_NAME}
```
---

## 汇总配置

### 业务层面

1. 项目依赖 `pom.xml`

```xml
        <!-- 引入Spring boot的监控机制-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
        </dependency>
```

2. 定义访问端口、路径及权限 `application.yaml`

```yaml
spring:
  application:
    name: project-sample
  profiles:
    active: @profileActive@
  lifecycle:
    timeout-per-shutdown-phase: 30s     # 停机过程超时时长设置30s，超过30s，直接停机

server:
  port: 8080
  shutdown: graceful                    # 默认为IMMEDIATE，表示立即关机；GRACEFUL表示优雅关机

management:
  server:
    port: 50000                         # 启用独立运维端口
  metrics:
    tags:
      application: ${spring.application.name}
  endpoint:                             # 开启shutdown和health端点
    shutdown:
      enabled: true
    health:
      probes:
        enabled: true
  endpoints:
    web:
      exposure:
        base-path: /actuator            # 指定上下文路径，启用相应端点
        include: health,shutdown,metrics,prometheus
```

### 运维层面

- 确保dockerfile模版集成curl工具，否则无法使用curl命令

```dockerfile
FROM openjdk:8-jdk-alpine
#构建参数
ARG JAR_FILE
ARG WORK_PATH="/app"
ARG EXPOSE_PORT=8080

#环境变量
ENV JAVA_OPTS=""\
    JAR_FILE=${JAR_FILE}

#设置时区
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo 'Asia/Shanghai' >/etc/timezone
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories  \
    && apk add --no-cache curl
#将maven目录的jar包拷贝到docker中，并命名为for_docker.jar
COPY target/$JAR_FILE $WORK_PATH/


#设置工作目录
WORKDIR $WORK_PATH


# 指定于外界交互的端口
EXPOSE $EXPOSE_PORT
# 配置容器，使其可执行化
ENTRYPOINT exec java $JAVA_OPTS -jar $JAR_FILE
```

- k8s部署模版`deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {APP_NAME}
  labels:
    app: {APP_NAME}
spec:
  selector:
    matchLabels:
      app: {APP_NAME}
  replicas: {REPLICAS}                            # Pod副本数
  strategy:
    type: RollingUpdate                           # 滚动更新策略
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      name: {APP_NAME}
      labels:
        app: {APP_NAME}
      annotations:
        timestamp: {TIMESTAMP}
        prometheus.io/port: "50000"               # 不能动态赋值
        prometheus.io/path: /actuator/prometheus
        prometheus.io/scrape: "true"              # 基于pod的服务发现
    spec:
      affinity:                                   # 设置调度策略，采取多主机/多可用区部署
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - {APP_NAME}
              topologyKey: "kubernetes.io/hostname" # 多可用区为"topology.kubernetes.io/zone"
      terminationGracePeriodSeconds: 30             # 优雅终止宽限期
      containers:
      - name: {APP_NAME}
        image: {IMAGE_URL}
        imagePullPolicy: Always
        ports:
        - containerPort: {APP_PORT}
        - name: management-port
          containerPort: 50000         # 应用管理端口
        readinessProbe:                # 就绪探针
          httpGet:
            path: /actuator/health/readiness
            port: management-port
          initialDelaySeconds: 30      # 延迟加载时间
          periodSeconds: 10            # 重试时间间隔
          timeoutSeconds: 1            # 超时时间设置
          successThreshold: 1          # 健康阈值
          failureThreshold: 9          # 不健康阈值
        livenessProbe:                 # 存活探针
          httpGet:
            path: /actuator/health/liveness
            port: management-port
          initialDelaySeconds: 30      # 延迟加载时间
          periodSeconds: 10            # 重试时间间隔
          timeoutSeconds: 1            # 超时时间设置
          successThreshold: 1          # 健康阈值
          failureThreshold: 6          # 不健康阈值
        resources:                     # 容器资源管理
          limits:                      # 资源限制（监控使用情况）
            cpu: 0.5
            memory: 1Gi
          requests:                    # 最小可用资源（灵活调度）
            cpu: 0.1
            memory: 200Mi
        env:
          - name: TZ
            value: Asia/Shanghai
---
kind: HorizontalPodAutoscaler            # 弹性伸缩控制器
apiVersion: autoscaling/v2beta2
metadata:
  name: {APP_NAME}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {APP_NAME}
  minReplicas: {REPLICAS}                # 缩放范围
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu                        # 指定资源指标
        target:
          type: Utilization
          averageUtilization: 50
```