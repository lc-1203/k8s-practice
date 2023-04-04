## Stress（K8S部署）

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: stress
  labels:
    app: stress
spec:
  selector:
    matchLabels:
      app: stress
  template:
    metadata:
      name: stress
      labels:
        app: stress
      annotations:
        timestamp: 17:49
    spec:
      nodeSelector:
        deploy.type: stress
      containers:
      - name: stress
        image: polinux/stress
        imagePullPolicy: IfNotPresent
        command:
        - /bin/sh
        - -c
        - |
          sleeptime=$((RANDOM%10+1))
          cpu=$((RANDOM%3+2))
          vm=$((RANDOM%9+8))
          time=$((RANDOM%60+11))
          sleep $((sleeptime))
          stress -c $((cpu)) --vm $((vm)) --vm-bytes 1024M --vm-hang $((time)) --timeout $((time))s --verbose
          stress -c 1 --vm 1 --vm-bytes 1024M --vm-hang 30 --timeout 30 --verbose
        resources:
          limits:
            cpu: 6
            memory: 24Gi
          requests:
            cpu: 2
            memory: 8Gi
      restartPolicy: Always
```



## 加压程序（Linux）

```
vim cpu_task.py
```

```
i = 0
while True:
    i = i+1
```

- 执行并显示进程号

```
python cpu_task.py /dev/null &
```

### 创建CPU任务

- 创建cgroup子系统

```
mkdir /sys/fs/cgroup/cpu/cpu_test
```

- 设置子系统cpu风分配（除以1000为百分率）

```
echo 1000 > /sys/fs/cgroup/cpu/cpu_test/cpu.cfs_quota_us
```

- 将进程号写进tasks文件中

```
echo 10331 >> /sys/fs/cgroup/cpu/cpu_test/tasks
```

### 随机加压程序

- 创建脚本：每隔[20,60]s设定单核CPU占用率为[40,60]%

```
vim random_cfs_quota_us.sh
```

```
#!/bin/bash
while true
do
cpu=$(((RANDOM%21+40)*1000));
time=$((RANDOM%41+20));
echo $((cpu)) > /sys/fs/cgroup/cpu/cpu_test/cpu.cfs_quota_us
sleep $((time))
done
```

- 授权

```
chmod +x random_cfs_quota_us.sh
```

- 后台运行

```
./random_cfs_quota_us.sh &
```

- 查询后台运行的脚本任务

```
jobs
```

- 中断后台脚本

```
kill %number
```

> [用什么手段可以把linux服务器的CPU跑在50%左右？](https://blog.csdn.net/l714417743/article/details/118309445)

> [shell基础之后台运行脚本](https://www.cnblogs.com/renyz/p/11364771.html) 