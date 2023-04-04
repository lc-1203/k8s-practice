# Ansible学习笔记（K8S部署）

## 为什么编写K8S自动化部署工具

- 快速部署
- 配置统一
- 便于后期维护
- 易于故障排查
- 自动化更容易掌控
- 熟悉二进制详细部署流程及架构

## Ansible介绍

- 一种IT自动化工具，用于配置系统、部署软件，以及协调更高级的IT任务，如持续部署，滚动更新等；

- 适用于管理企业IT基础设施，小规模到数千个实例的企业环境

- Ansible也是一种简单的自动化语言，可以完美的描述IT应用程序基础结构

  

特点：简单（学习成本低），强大（应用生命周期的管理），无代理（基于sh通信）



安装（默认纳入red hat）：

```
yum install ansible -y
```



查看命令行工具：ansible+Tab

![image-20201210140100991](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20201210140101.png)



ansible原理：只要能通过sh连接集群，就可以管理

![image-20201210140633382](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20201210140633.png)

Inventory：资产清单，被管理端有哪些（ip，用户名密码等）

module：任务均由模块完成，可定制性开发（如常用的脚本）

Plugins：插件

Playbooks：剧本，模块化定义一系列任务，供外部统一调用

## 主机清单

### 打开主机清单的host，定义被管理的机器

```
vi /etc/ansible/hosts

[webservers]
172.18.248.134
```

### 连接

#### 直接连接 -m 方式 -a 命令

```
ansible webservers -m shell -a "ls"
#无权限
```

#### 在资产清单中定义用户名密码

```
vi /etc/ansible/hosts

[webservers]
172.18.248.134 ansible_ssh_user=root ansible_ssh_pass="Aa666666"

ansible webservers -m shell -a "ls"
```

#### 命令行交互式输入

```
ansible webservers -m shell -a "ls" -k -uroot

ansible w -m shell -a "ls" -k -uroot
```

#### SSH密钥对认证（免密登录）

#### 运行方式

- 命令行：简单，主要是完成一次性任务

  ```
  # 模块文档(如ping shell等) 
  ansible-doc -l
  ansible-doc -s ping
  
  
  # 检查资产清单能否连上
  ansible all -m ping
  
  #查看利用率等
  ansible all -m shell -a "free -m" 
  ```

  

- palybook：文本复用，方便管理任务，主要是完成一些复杂的任务

## 常用模块

- shell
- copy
- file
- yum
- service/systemd
- unarchive
- debug

## Playbook

```
- hosts: master
  tasks:
  - name: Ansible delete file glob
    find:
      paths: /opt/kubernetes/ssl
      patterns: kubelet-client-2021.*
    register: files_to_delete
```

## ansible批量传文件及解压

```
ansible worknodes -m copy -a "src=node-exporter_v0_16.tar.gz dest=/root"
ansible worknodes -m shell -a "docker load -i node-exporter_v0_16.tar.gz"


ansible worknodes -m copy -a "src=prometheus_v2_2_1.tar.gz dest=/root"
ansible worknodes -m shell -a "docker load -i prometheus_v2_2_1.tar.gz"

ansible worknodes -m shell -a "mkdir /data && chmod 777 /data/"

ansible worknodes -m copy -a "src=heapster-grafana-amd64_v5_0_4.tar.gz dest=/root"
ansible worknodes -m shell -a "docker load -i heapster-grafana-amd64_v5_0_4.tar.gz"
```

