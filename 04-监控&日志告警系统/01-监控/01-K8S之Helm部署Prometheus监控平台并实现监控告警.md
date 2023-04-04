#  K8S之Helm部署Prometheus监控平台并实现监控告警

[toc]

## 概述

- 本文采用helm安装Prometheus+Grafana

- 配置alertmanager及告警规则实现邮件报警。
- 其中所采用的helm仓库及chart包如下所示：

```shell
# helm仓库
grafana: https://grafana.github.io/helm-charts
prometheus-community: https://prometheus-community.github.io/helm-charts

# chart包
grafana/grafana
prometheus-community/prometheus
```

## 准备工作

### 安装helm

- 项目地址：https://github.com/helm/helm

- 安装：

```shell
# 下载（自行选择版本）
wget https://get.helm.sh/helm-v3.8.1-linux-amd64.tar.gz

# 解压
tar zxvf helm-v3.8.1-linux-amd64.tar.gz

# 安装
mv linux-amd64/helm /usr/local/bin/

# 验证
helm version
```

- 删除Helm使用时关于kubernetes文件的警告

```shell
chmod g-rw ~/.kube/config
chmod o-r ~/.kube/config
```

### chart包下载

```shell
# 添加grafana和prometheus-community仓库（无响应时多尝试几次）
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# 更新仓库
helm repo update

# 查询chart
helm search repo grafana

# 创建工作目录
mkdir -p ~/workspace/prometheus

# 拉取所有的chart包（请放到相应的目录中）
cd ~/workspace/prometheus

helm pull grafana/grafana
helm pull prometheus-community/prometheus

helm pull prometheus-community/prometheus-mysql-exporter
helm pull prometheus-community/prometheus-redis-exporter
helm pull prometheus-community/prometheus-kafka-exporter
helm pull prometheus-community/prometheus-rabbitmq-exporter

# 分别解压
tar zxvf [压缩包] 
```

### 镜像同步

- prometheus内嵌kube-state-metrics安装包，其使用的是gcr镜像，也是所有chart包中唯一的gcr镜像，可能会导致镜像拉取失败，因此有必要提前同步该镜像

![image-20220314194222118](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203141942167.png)

- 编辑配置文件

> 已同步到个人阿里云镜像仓库

```shell
cd cd ~/workspace/prometheus/prometheus
vim charts/kube-state-metrics/values.yaml
```

```shell
# Default values for kube-state-metrics.
prometheusScrape: true
image:
  repository: registry.cn-zhangjiakou.aliyuncs.com/gcr-sync/kube-state-metrics
  tag: v2.3.0
  pullPolicy: IfNotPresent
```

## 安装Prometheus

- 进入工作目录，按需修改镜像，持久化存储，副本数等配置；
- 建议首次部署时直接修改values中的配置，而不是用--set的方式，这样后期upgrade不必重复设置。

```shell
cd  ~/workspace/promethues/promethues
vim values.yaml
```

### 设置持久化存储

- 若无需持久化，将`enabled`设置为`false`
- 若使用文件存储，需将accessMode改为ReadWriteMany
- **storageClass**的创建请参考之前的文章

```shell
# 搜索持久化设置，VIM界面按Esc后输入(再按n搜索下一个)：
/persistentVolume

# 总共有三处，分别为alertmanager，Prometheus server和pushgateway。
# 参考官方文档建议配置，本文仅开启Prometheus server的持久化，其它的关闭
```

注：可能需要设置`runAsUser`为0（root用户）

![image-20220314195221334](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203141952422.png)

### 多副本

- 设置Prometheus server的副本数为3，并开启statefulset

![image-20220314195530022](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203141955066.png)

### 开启NodePort

- Alertmanager，更改ClusterIP为NodePort，并设置nodeport端口号

![image-20220314201129815](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203142011856.png)

- Prometheus server，更改ClusterIP为NodePort，并新增nodeport字段

![image-20220314201252481](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203142012519.png)

### 部署

```shell
# 创建命名空间
kubectl create ns prometheus

# 确保是在工作目录:~/workspace/prometheus/prometheus，helm部署
helm install prometheus -n prometheus .
```

- 部署完查看service，将会在grafana中配置数据源时用到

![image-20220314203041678](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203142030725.png)

- 访问alertmanager-dashboard：`<Node-IP>:30090`
- 访问server-dashboard：`<Node-IP>:30091`

### 异常处理

若server无法启动，日志提示`err="open /data/queries.active: permission denied"`，则需设置`runAsUser`为0（root用户）

![image-20220314203002665](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203142030707.png)

## 安装Grafana

同样安装在prometheus空间下

### 创建Secret

- 在prometheus命名空间下新建secret，帐号密码：admin / grafana

```shell
cd ~/workspace/prometheus/grafana
echo -n "admin" | base64
echo -n "grafana" | base64
```

```yaml
cat > secret.yaml  <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: grafana
  namespace: prometheus
type: Opaque
data:
  admin-user: YWRtaW4=
  admin-password: Z3JhZmFuYQ==
EOF
```

```shell
kubectl apply -f secret.yaml
```

### chart包参数设置

- 进入工作目录，按需修改镜像，持久化存储，副本数等配置；
- 建议首次部署时直接修改values中的配置，而不是用--set的方式，这样后期upgrade不必重复设置。

```shell
vim values.yaml
```

#### 设置密码

```yaml
admin:
  existingSecret: "grafana"    # 即之前创建的secret
  userKey: admin-user
  passwordKey: admin-password
```

![image-20210805202515948](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210805202515.png)

#### 设置持久化存储

- 若无需持久化，将`enabled`设置为`false`
- 若使用文件存储，需将accessMode改为ReadWriteMany

```yaml
# 搜索持久化设置，VIM界面按Esc后输入：
/persistence

persistence:
  type: pvc
  enabled: true
  storageClassName: nfs-sc
  accessModes:
    - ReadWriteOnce
  size: 2Gi
```

![image-20210805201847152](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210805201847.png)

### 设置NodePort

更改ClusterIP为NodePort，并新增nodeport字段

![image-20220314203958389](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203142039441.png)

### 部署

```shell
helm install grafana -n prometheus .
```

## 配置dashboard

### 登录grafana

访问grafana-dashboard：`<Node-IP>:30092`

> 帐号密码（之前自定义的secret）： admin /grafana

![image-20210805202220603](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210805202220.png)



### 配置Data sources

- 首先，获取prometheus的service地址

```shell
# 查询svc
kubectl get svc -n prometheus

# prometheus-server.prometheus:80
```

- 进入Data sources配置页面

<img src="https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210805202717.png" alt="image-20210805202717816" width="500px" />

- 添加Prometheus，URL填入prometheus的service（80端口可省略）

<img src="https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210805203537.png" alt="image-20210805202858810" width="700px" />

### 导入dashboard模版

- Data sources配置完成后，导入模版

<img src="https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210805203008.png" alt="image-20210805203008361" width="400px" />

- 导入模版：*Node Exporter for Prometheus Dashboard CN 20201010*（8919）

> 更多模版请参考官网网站：https://grafana.com/grafana/dashboards

<img src="https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210805203129.png" alt="image-20210805203129789" width="500px" />

- 数据源选择Prometheus，然后点击import

<img src="https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210805203220.png" alt="image-20210805203220083" width="600px" />

- 最终效果：

![image-20220314205446711](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203142054917.png)

## 监控告警功能

配置方法没变化，有时间再更新

### alertmanager邮箱告警配置

- 首先开通SMTP服务，QQ邮箱：设置--帐号--开通POP3/SMTP服务，记住生成的密码（其它邮箱同理）

![image-20210806151500096](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210806151500.png)

- 编辑prometheus的values.yaml文件，配置邮箱告警

```shell
vim ~/workspace/prometheus/prometheus/values.yaml

# 定位到alertmanager的配置文件，VIM界面按Esc后输入：
/alertmanagerFiles
```

```yaml
alertmanagerFiles:
  alertmanager.yml:
    global:
      resolve_timeout: 5m
      # 邮箱告警配置
      smtp_hello: 'prometheus'
      smtp_from: 'xxx@qq.com'
      smtp_smarthost: 'smtp.qq.com:465'  # 其它邮箱请填写相应的host
      smtp_auth_username: 'xxx@qq.com'
      smtp_auth_password: 'xxxxxxxxxxxxxxxx'
      smtp_require_tls: false            # qq邮箱需要设定
    templates:
    - '/etc/config/*.tmpl'               # 指定告警模板路径
    receivers:
      - name: email
        email_configs:
        - to: 'x@qq.com'
          headers: {"subject":'{{ template "email.header" . }}'}
          html: '{{ template "email.html" . }}'
          send_resolved: true            # 发送报警解除邮件
    route:
      group_wait: 5s                     # 分组等待时间
      group_interval: 5s                 # 上下两组发送告警的间隔时间
      receiver: email
      repeat_interval: 5m                # 重复发送告警时间
    inhibit_rules:                       # 告警抑制：当多级别规则同时生效时，只发送最高级别的告警
    - source_match:
        severity: 'critical'
      target_match:
        severity: 'warning'
      equal: ['alertname']

  template_email.tmpl: |-           # 告警模版
    {{ define "email.header" }}
    {{ if eq .Status "firing"}}[Warning]: {{ range .Alerts }}{{ .Annotations.summary }} {{ end }}{{ end }}
    {{ if eq .Status "resolved"}}[Resolved]: {{ range .Alerts }}{{ .Annotations.resolve_summary }} {{ end }}{{ end }}
    {{ end }}

    {{ define "email.html" }}
    {{ if gt (len .Alerts.Firing) 0 -}}
    <font color="#FF0000"><h3>[Warning]:</h3></font>
    {{ range .Alerts }}
    告警级别：{{ .Labels.severity }}   <br>
    告警类型：{{ .Labels.alertname }}   <br>
    故障主机: {{ .Labels.instance }}   <br>
    告警主题: {{ .Annotations.summary }}   <br>
    告警详情: {{ .Annotations.description }}   <br>
    触发时间: {{ (.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}   <br>
    {{- end }}
    {{- end }}

    {{ if gt (len .Alerts.Resolved) 0 -}}
    <font color="#66CDAA"><h3>[Resolved]:</h3></font>
    {{ range .Alerts }}
    告警级别：{{ .Labels.severity }}   <br>
    告警类型：{{ .Labels.alertname }}   <br>
    故障主机: {{ .Labels.instance }}   <br>
    告警主题: {{ .Annotations.resolve_summary }}   <br>
    告警详情: {{ .Annotations.resolve_description }}   <br>
    触发时间: {{ (.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}   <br>
    恢复时间: {{ (.EndsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}   <br>
    {{- end }}
    {{- end }}
    {{- end }}
```

### prometheus告警规则配置

- 接着配置告警规则，以“物理节点状态”为例，先在prometheus控制台测试此条告警规则，确保输出有效：

```sql
up{component="node-exporter"} 
```

![image-20210806151134924](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210806151135.png)

- 在values中配置告警规则（就在alertmanger的配置文件下方）

```yaml
serverFiles:
  alerting_rules.yml:
    groups:
      - name: 物理节点状态-监控告警
        rules:
          - alert: Node-up
            expr: up {component="node-exporter"} == 0
            for: 2s
            labels:
              severity: critical
            annotations:
              summary: "服务器{{ $labels.kubernetes_node }}已停止运行！"
              description: "检测到服务器{{ $labels.kubernetes_node }}已异常停止，IP: {{ $labels.instance }}，请排查！"
              resolve_summary: "服务器{{ $labels.kubernetes_node }}已恢复运行！"
              resolve_description: "服务器{{ $labels.kubernetes_node }}已恢复运行，IP: {{ $labels.instance }}。"
```

### 更新prometheus

- 编辑完成后，更新prometheus

> 每次增加规则中都需要upgrage，更新后pod中的“configmap-reload”容器会重载配置文件，可能需要等待几分钟

```shell
helm upgrade prometheus -n prometheus .
```

- 查询alertmanger的配置文件是否更新（server同理）：

```shell
kubectl logs -f -n prometheus prometheus-alertmanager-7757db759b-9nq9c prometheus-alertmanager
```

![image-20210806103815005](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210806103815.png)

- 查询server的configmap（alertmanger同理）

```shell
kubectl get configmap -n prometheus
kubectl describe configmaps -n prometheus prometheus-server
```

![image-20210806104124062](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210806104124.png)

- 在alert控制台查看告警规则是否生效：

![image-20210806152001660](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210806152001.png)

### 告警测试

- 停掉某台节点（k8s-node1）

![image-20210806145439890](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210806145439.png)

- 可观察到Alert中，该告警规则状态由Inactive转到Pending再到Firing，而当状态转为Firing，将发送告警邮件

> Pending到Firing的变化默认为1分钟，若想缩短时间，请修改value.yaml中的`server.global.scrape_interval`字段，如`15s`

![image-20210806145525139](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210806145525.png)





![image-20210806145539814](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210806145539.png)

- 告警邮件

![image-20210806150915696](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210806150915.png)

![image-20210806151046098](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210806151046.png)

## 附告警规则

> 部分参考自[阿里云Prometheus监控报警规则](https://help.aliyun.com/document_detail/176180.html?spm=a2c4g.11186623.4.5.717e3eebxbWsZy#title-dcc-mv9-w33)

### K8S组件状态

#### target状态

```sql
sum by (instance,job) (up)
```

#### CPU使用率

```sql
round (sum by (instance,job) (rate(process_cpu_seconds_total[2m]) * 100)) > 80
```

#### 句柄数

```sql
sum by (instance,job) (process_open_fds) > 600
```

#### 虚拟内存

```sql
sum by (instance,job) (round(process_virtual_memory_bytes/1024/1024)) > 4096
```

### 集群资源状态

#### 资源限制：总CPU资源过载（超过80%）

集群 CPU 过度使用，CPU 已经过度使用无法容忍节点故障

```sql
round(sum(kube_pod_container_resource_requests{resource="cpu"}) / sum(kube_node_status_allocatable{resource="cpu"})*100) > 80
```

#### 资源限制：总内存资源过载（超过80%）

集群内存过度使用，内存已经过度使用无法容忍节点故障

```sql
round(sum(kube_pod_container_resource_requests{resource="memory"}) / sum(kube_node_status_allocatable{resource="memory"})*100) > 80
```

#### KubeletTooManyPods（Pod过多）

```sql
max(max(kubelet_running_pods) by(instance) * on(instance) group_left(node) kubelet_node_name) by(node) / max(kube_node_status_capacity{resource="pods"} != 1) by(node) > 0.9
```

### Node资源状态

#### CPU使用率

```sql
round((1 - avg(rate(node_cpu_seconds_total{component="node-exporter",mode="idle"}[5m])) by (instance)) * 100) 
```

#### 内存

```sql
round((1 - (node_memory_MemAvailable_bytes / (node_memory_MemTotal_bytes)))* 100) 
```

#### 剩余容量

```sql
(round((node_filesystem_avail_bytes{fstype=~"ext4|xfs"} / node_filesystem_size_bytes{fstype=~"ext4|xfs"}) * 100 < 30 ) and node_filesystem_readonly{fstype=~"ext4|xfs"} == 0)
```

#### 预测剩余容量

```sql
( round(predict_linear(node_filesystem_avail_bytes{fstype=~"ext4|xfs"}[24h], 7*24*60*60)/1024/1024/1024) < 30   and node_filesystem_readonly{fstype=~"ext4|xfs"} == 0 )
```

#### 节点磁盘的IO使用率

```sql
100-(avg(rate(node_disk_io_time_seconds_total[2m])) by(instance)* 100) < 80
```

#### NodeNetworkReceiveErrs（Node网络接收错误）

```sql
sum (increase(node_network_receive_errs_total[2m])) by (instance) > 10
```

#### NodeNetworkTransmitErrs（Node网络传输错误）

```sql
sum (increase(node_network_transmit_errs_total[2m])) by (instance) > 10
```

#### 入网流量带宽

> 持续5分钟高于100M

```sql
((sum(rate (node_network_receive_bytes_total{device!~'tap.*|veth.*|br.*|docker.*|virbr*|lo*'}[5m])) by (instance)) /102400) > 100
```

#### 出网流量带宽

```sql
((sum(rate (node_network_transmit_bytes_total{device!~'tap.*|veth.*|br.*|docker.*|virbr*|lo*'}[5m])) by (instance))/102400) > 100
```

#### TCP_ESTABLISHED过高

```sql
node_netstat_Tcp_CurrEstab > 1000
```

### Pod

#### PodCpu75（Pod的CPU使用率大于75%）

> container!="POD", container!=""}[2m]
>
> {name=~".+"}：筛选，避免重复指标

```sql
round(100*(sum(rate(container_cpu_usage_seconds_total {name=~".+"}[2m])) by (namespace,pod)  /   sum(kube_pod_container_resource_limits{resource="cpu"}) by (namespace,pod))) > 75
```

#### PodMemory75（Pod的内存使用率大于75%）

```sql
100 * sum(container_memory_working_set_bytes{name=~".+"}) by (namespace,pod) / sum(kube_pod_container_resource_limits {resource="memory"}) by (namespace,pod) > 75
```

#### pod_status_no_running（Pod的状态为未运行）

```sql
sum (kube_pod_status_phase{phase!="Running"}) by (namespace,pod,phase) > 0
```

#### PodMem4GbRestart（Pod的内存大于4096MB）

```sql
(sum (container_memory_working_set_bytes{name=~".+"})by (namespace,pod,container_name) /1024/1024) > 4096
```

#### PodRestart（Pod重启）

> {pod!~"aws-load-balancer-controller.*"}

```sql
sum (round(increase (kube_pod_container_status_restarts_total[5m]))) by (namespace,pod) > 0
```

#### KubePodCrashLooping（Pod出现循环崩溃）

```sql
rate(kube_pod_container_status_restarts_total{app_kubernetes_io_name="kube-state-metrics"}[15m]) * 60 * 5 > 0
```

#### KubePodNotReady（Pod未准备好）

```sql
sum by (namespace, pod) (max by(namespace, pod) (kube_pod_status_phase{app_kubernetes_io_name="kube-state-metrics", phase=~"Pending|Unknown"}) * on(namespace, pod) group_left(owner_kind) max by(namespace, pod, owner_kind) (kube_pod_owner{owner_kind!="Job"}))  > 0
```

#### KubeContainerWaiting（容器等待）

```sql
sum by (namespace, pod, container) (kube_pod_container_status_waiting{app_kubernetes_io_name="kube-state-metrics"}) > 0
```

### Deployment

#### KubeDeploymentGenerationMismatch（出现部署集版本不匹配）

```sql
kube_deployment_status_observed_generation{app_kubernetes_io_name="kube-state-metrics"} != kube_deployment_metadata_generation{app_kubernetes_io_name="kube-state-metrics"}
```

#### KubeDeploymentReplicasMismatch（出现部署集副本不匹配）

```sql
( kube_deployment_spec_replicas{app_kubernetes_io_name="kube-state-metrics"} != kube_deployment_status_replicas_available{app_kubernetes_io_name="kube-state-metrics"} ) and ( changes(kube_deployment_status_replicas_updated{app_kubernetes_io_name="kube-state-metrics"}[5m]) == 0 )
```

#### 检测到部署集有更新

```sql
sum by (namespace, deployment) (changes(kube_deployment_status_observed_generation{app_kubernetes_io_name="kube-state-metrics"}[5m])) > 0
```

### StatefulSet

#### KubeStatefulSetGenerationMismatch（状态集版本不匹配）

```sql
kube_statefulset_status_observed_generation{app_kubernetes_io_name="kube-state-metrics"} != kube_statefulset_metadata_generation{app_kubernetes_io_name="kube-state-metrics"}
```

#### KubeStatefulSetReplicasMismatch（状态集副本不匹配）

```sql
( kube_statefulset_status_replicas_ready{app_kubernetes_io_name="kube-state-metrics"} != kube_statefulset_status_replicas{app_kubernetes_io_name="kube-state-metrics"} ) and ( changes(kube_statefulset_status_replicas_updated{app_kubernetes_io_name="kube-state-metrics"}[5m]) == 0 )
```

#### 检测到状态集有更新

```sql
sum by (namespace, statefulset) (changes(kube_statefulset_status_observed_generation{app_kubernetes_io_name="kube-state-metrics"}[5m]))
```

### PV&PVC

#### KubePersistentVolumeFillingUp（块存储PVC容量即将不足）

```sql
sum by (namespace,persistentvolumeclaim) (round(kubelet_volume_stats_available_bytes / kubelet_volume_stats_capacity_bytes*100)) < 20
```

#### KubePersistentVolumeErrors（PV容量出错）

```sql
sum by (persistentvolume) (kube_persistentvolume_status_phase{phase=~"Failed|Pending",app_kubernetes_io_name="kube-state-metrics"}) > 0
```

#### KubePersistentVolumeFillingUp（PVC空间耗尽预测）

> 通过PVC资源使用6小时变化率预测 接下来4天的磁盘使用率

```sql
(kubelet_volume_stats_available_bytes / kubelet_volume_stats_capacity_bytes ) < 0.4 and predict_linear(kubelet_volume_stats_available_bytes[6h], 4 * 24 * 3600) < 0
```

