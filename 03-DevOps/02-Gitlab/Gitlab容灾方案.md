# Gitlab容灾方案

## 概述

gitlab 是一个基于git实现的在线代码仓库软件，提供web可视化管理界面，通常用于企业团队内部协作开发。Gitlab不仅保存了科室内的重要代码，并且也是项目自动化构建的重要一环，为DevOps提供源代码控制。因此为Gitlab建设容灾方案、保证Gitlab业务的连续性是当前的关键事宜。

## 容灾方案设计

### 容灾目标

容灾能力建设的主要目的，就是在灾难发生的时候，能够保证生产业务系统的不间断运行。评估容灾的目标要求，需要从恢复时间目标(RTO)和恢复点目标(RPO)的选定范围出发：

① RTO：企业可容许服务中断的时间长度，简言之业务可以恢复的最快时间。

② RPO：企业可容许数据丢失的数量级，简言之数据可以恢复到最新的时刻点。

因此，制定容灾建设方案前，须综合评估RTO&RPO的限定范围并权衡企业可以付诸的投入，最终才能确定合理的容灾建设方案。

### RTO和RPO限制

Gitlab服务器信息：

| 服务器           | 系统版本  | Gitlab版本 | 部署方式 | 数据备份/恢复说明                |
| ---------------- | --------- | ---------- | -------- | -------------------------------- |
| 主服务器         | CentOS6.9 | 11.10.4-ce | 裸机部署 | 定时每天全量备份，备份大小约200G |
| 备服务器（计划） | CentOS7.9 | 11.10.4-ce | Docker   | 定时将主服务器备份恢复           |

结合Gitlab主服务器的状态分析，限制因素如下：

| 分类 | 限制因素                                     | 说明                             |
| ---- | -------------------------------------------- | -------------------------------- |
| RTO  | 访问方式，主备切换需要更换IP访问             | 恢复时间取决于切换IP访问的速度。 |
| RPO  | 备份方式，数据的备份传输时间影响数据的时效性 | 恢复点取决于数据备份的间隔时间   |

详细说明：

- 访问方式：主服务器通过固有IP访问，没有设定负载均衡LB。并由于目前Gitlab服务已被广泛使用（七所，四所，及jenkins平台），切换为LB访问的成本较高。

- 备份方式：每天定时全量备份，备份文件大小约200G，影响备份时间、传输时间和恢复时间，从而拉长了数据备份的间隔时间。此外，全量备份方式转增量备份的工序比较复杂，需要花时间攻关，建议整理gitlab空间占用，清理/转移文件，以缩短备份间隔时间。

### 方案设计

通过综合分析RTO&RPO的限定范围和容灾成本，评估当前可达到的容灾能力为：**数据定时异地备份，异地热备接管业务**。

首先在备用服务器上以docker方式部署与主服务器版本一致的Gitlab，形成两套功能相同的Gitlab系统。然后将系统运行环境作为一个整体进行定时异机备份及恢复，组成主备架构。当主服务器发生故障时，可以将Gitlab的运行环境在备服务器上整体恢复，以保证业务的连续性。具体方案如下：

![image-20210826090726659](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210826091233.png)

步骤说明：

1. Gitlab主服务器定时自动备份（02:00至07:00）：全量备份方式，备份时间耗时约5小时。
2. Gitlab主服务器定时自动将备份传输到备服务器（18:00至24:00）：
3. Gitlab备服务器定时自动恢复备份（02:00开始）
4. Gitlab备服务器定时删除过期备份（12:00开始）

## 具体实现

### 备服务器安装docker及docker-compose

#### docker

- 移除电脑上原有的dockers

```shell
yum remove docker \
           docker-client \
           docker-client-latest \
           docker-common \
           docker-latest \
           docker-latest-logrotate \
           docker-logrotate \
           docker-engine
```

- 安装docker-ce

```shell
yum install -y yum-utils
 
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
 
yum install docker-ce
```

- 设置开启启动

```
systemctl daemon-reload &&
systemctl restart docker &&
systemctl enable docker &&
systemctl status docker
```

#### docker-compose

- 下载[docker-compose](https://github.com/docker/compose/releases)

![image-20210324144910802](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210324144910.png)

- 拷贝至远程主机，重命名，赋予权限，移动至可执行目录下

![image-20210324145243654](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210324145243.png)

### 备服务器安装Gitlab

```
vim docker-compose.yml
```

```
version: '3'
services:
  web:
    image: 'gitlab/gitlab-ce:11.10.4-ce.0'
    container_name: gitlab
    restart: unless-stopped
    privileged: true
    hostname: 'gitlab'
    environment:
      TZ: 'Asia/Shanghai'
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://xxx' #项目clone地址
        gitlab_rails['time_zone'] = 'Asia/Shanghai'
        gitlab_rails['backup_archive_permissions'] = 0644  # 只读
        gitlab_rails['backup_keep_time'] = 86400 # 过期时间2天
    ports:
      - '80:80'  #默认80 自定义时主机IP需要容器IP一致
      - '443:443'
      - '22:22'
    volumes:
      - '/$PWD/gitlab/config:/etc/gitlab'
      - '/$PWD/gitlab/data:/var/opt/gitlab'
      - '/$PWD/gitlab/logs:/var/log/gitlab'
```

```
version: '3'
services:
  web:
    image: 'gitlab/gitlab-ce:11.10.4-ce.0'
    container_name: gitlab
    restart: unless-stopped
    privileged: true
    hostname: 'gitlab'
    environment:
      TZ: 'Asia/Shanghai'
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://10.2.3' 
        gitlab_rails['time_zone'] = 'Asia/Shanghai'
        gitlab_rails['backup_archive_permissions'] = 0644  
        gitlab_rails['backup_keep_time'] = 86400 
    ports:
      - '80:80'  
      - '443:443'
      - '22:22'
    volumes:
      - '/data/gitlab/config:/etc/gitlab'
      - '/data/gitlab/data:/var/opt/gitlab'
      - '/data/gitlab/logs:/var/log/gitlab'
```



```
# 启动
docker-compose up -d

# 查看状态
docker ps -a |grep gitlab

# 容器运行后，生效的配置文件位于：
/gitlab/gitlab/data/gitlab-rails/etc
```

### 主服务器配置免密登录备服务器

### 主服务器定时自动备份

```
vim auto_backup.sh
```

```
docker exec -t gitlab gitlab-rake gitlab:backup:create CRON=1
```

```
# 添加定时任务
crontab -e  

# 每天凌晨两点创建备份
* 2 * * * ~/gitlab/auto_backup.sh -D 1 
```

### 主服务器定时自动将备份传输到备服务器

```
vim auto_scp_backupfile_to_remote.sh
```

```
#!/bin/bash

# gitlab 机房备份路径
LocalBackDir=/data/gitlab/backups

# 远程备份服务器 gitlab备份文件存放路径
RemoteBackDir=/data/gitlab/backups

# 远程备份服务器 登录账户
RemoteUser=root

# 远程备份服务器 IP地址
RemoteIP=10.2.30.193

#当前系统日期
DATE=`date +"%Y-%m-%d"`

#Log存放路径
LogFilePath=$LocalBackDir/log
LogFile=$LocalBackDir/log/$DATE.log

# 查找 本地备份目录下查找指定时间范围内（1天）最新的，并且后缀为.tar的gitlab备份文件
RESTORE_BACKUPFILE=$(find $LocalBackDir -type f -mtime -1  -name '*.tar*'|sort -nr |head -n 1)

#新建日志文件
mkdir -p $LogFilePath
touch $LogFile

#追加日志到日志文件
echo "---------------------------------------------------------------------------" >> $LogFile
echo "Gitlab auto backup to remote server, start at  $(date +"%Y-%m-%d %H:%M:%S")" >> $LogFile
echo "The file to scp to remote server is: $BACKUPFILE_SEND_TO_REMOTE" >> $LogFile

#备份到远程服务器
scp $BACKUPFILE_SEND_TO_REMOTE $RemoteUser@$RemoteIP:$RemoteBackDir

#追加日志到日志文件
echo "---------------------------------------------------------------------------" >> $LogFile
```

```
# 添加定时任务
crontab -e  

# 每天下午6点开始传输
* 18 * * * ~/gitlab/auto_scp_backupfile_to_remote.sh -D 1 
```

### 备服务器定时自动恢复备份

```
vim auto_restore_backup.sh
```

```
#!/bin/bash

# gitlab 机房备份路径
GitlabBackDir=~/gitlab/gitlab/data/backups

#当前系统日期
DATE=`date +"%Y-%m-%d"`

#Log存放路径
LogFilePath=$GitlabBackDir/restore_log
LogFile=$GitlabBackDir/restore_log/$DATE.log

# 查找 本地备份目录下查找指定时间范围内最新的，并且后缀为.tar的gitlab备份文件
RESTORE_BACKUPFILE=$(find $GitlabBackDir -type f -mmin -10  -name '*.tar*'|sort -nr |head -n 1)

#新建日志文件
mkdir -p $LogFilePath
touch $LogFile

#追加日志到日志文件
echo "---------------------------------------------------------------------------" >> $LogFile
echo "Gitlab auto restore backup, start at  $(date +"%Y-%m-%d %H:%M:%S")" >>  $LogFile
echo "The backup file to restore is: $RESTORE_BACKUPFILE" >> $LogFile

#恢复备份
BACKUPFILE_VERSION=$(echo ${RESTORE_BACKUPFILE//_gitlab_backup.tar/}| tr '/' '\n' | tail -n1)
docker exec -t gitlab gitlab-rake gitlab:backup:restore force=yes BACKUP=BACKUPFILE_VERSION

#追加日志到日志文件
echo "---------------------------------------------------------------------------" >> $LogFile
```

```
# 添加定时任务
crontab -e  

# 每天凌晨两点恢复备份
* 2 * * * ~/gitlab/auto_scp_backupfile_to_remote.sh -D 1 
```

### 备服务器定时自动删除过期备份

```
vim auto_remove_old_backup.sh
```

```
#!/bin/bash

# 远程备份服务器 gitlab备份文件存放路径
GitlabBackDir=~/gitlab/gitlab/data/backups

# 保留最近指定数量的备份，删除早期备份
for FILE in `ls -ac $GitlabBackDir/*.tar | sed "1,2d"`; do
rm $FILE
done
```

```
crontab -e  # 编辑定时任务

# 每天12点清除早期备份
* 12 * * * ~/gitlab/auto_remove_old_backup.sh -D 1   
```

```
#!/bin/bash
GitlabBackDir=~/gitlab/gitlab/data/backups
for FILE in `ls -ac $GitlabBackDir/*.tar | sed "1,2d"`; do
rm $FILE
done
```

