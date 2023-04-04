# k8s集群外主机通过kubectl控制集群

kubectl是Kubernetes集群的命令行工具，通过kubectl能够对集群本身进行管理，并能够在集群上进行容器化应用的安装部署。

## 问题描述

`kubectl`默认从`~/.kube/config`配置文件中获取访问kube-apiserver 地址、证书、用户名等信息，需要正确配置该文件才能正常使用`kubectl`命令。然而我通过Ansible部署k8s集群后，发现没有生成config配置文件，因此现在需要手动生成config配置文件并拷贝到需要使用kubectl 命令的机器上。

## 环境说明

|    角色    |     私网IP     |    公网IP    |                             组件                             |
| :--------: | :------------: | :----------: | :----------------------------------------------------------: |
|   工作站   | 172.18.248.135 | 8.135.50.213 |                         配置kubectl                          |
| k8s-master |  172.18.25.81  | 8.135.19.91  | Ansible，kube-apiserver  kube-controller-manager  kube-scheduler  etcd |
| k8s-node1  |  172.18.25.82  |              |              kubelet  kube-proxy  docker  etcd               |
| k8s-node2  |  172.18.25.83  |              |              kubelet  kube-proxy  docker  etcd               |

## 配置kubectl

### 安装kubectl
在需要使用kubectl 命令的机器上（工作站）安装kubectl

- 方式1：进入ansible/master主机，将kubectl二进制文件拷贝至工作站：

  ```
  # 从master主机拷贝kubectl到工作站
  ssh root@172.18.25.200
  scp /usr/bin/kubectl 172.18.25.100:/usr/bin/
  ```

- 方式2：[直接在工作站下载kubectl](https://www.cnblogs.com/hackyo/p/10670046.html)

  ```
  # 首先在master节点上查看kubectl版本
  kubectl version 
  #在工作站下载对应版本的kubectl
  curl -O https://dl.k8s.io/v1.18.6/kubernetes-client-linux-amd64.tar.gz
  # 解压安装
  tar zxf kubernetes-client-linux-amd64.tar.gz
  chmod +x kubectl
  mv kubectl /usr/bin/
  ```

### 生成config配置文件

- 进入master/ansible主机的证书目录，即搭建集群是为etcd及k8s生成自签证书所在的目录

  ```
  cd /workspace/ansible-install-k8s-master/ssl/k8s/
  ```

  ![image-20201225150219182](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20201225150219.png)

- [生成config文件，执行下面几个命令即可](https://www.cnblogs.com/yeyu1314/p/13430973.html)

  ```
  # 1 设置集群参数(注意：单master集群为master节点私网IP，高可用集群为虚拟IP)
  kubectl config set-cluster kubernetes \
    --server=https://8.142.145.207:6443 \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --kubeconfig=config
  
  
  # 2 设置客户端认证参数
  kubectl config set-credentials cluster-admin \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --client-key=admin-key.pem \
    --client-certificate=admin.pem \
    --kubeconfig=config
    
  # 3 设置上下文参数
  kubectl config set-context default \
    --cluster=kubernetes \
    --user=cluster-admin \
    --kubeconfig=config
    
  # 4 设置默认上下文
  kubectl config use-context default --kubeconfig=config
  ```

### 测试kubectl

- 执行完上述命令，会生成config文件，将其拷贝至工作站：

  ```
  scp config 172.18.25.200:~/
  
  # 进入工作站，创建默认目录
  mkdir ~/.kube
  # 将config文件移植默认目录
  mv config ~/.kube
  ```

![image-20201225150712341](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20201225150712.png)

- 在工作站测试kubectl

  ```
  kubectl get nodes
  ```

  ![image-20201225152009428](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20201225152009.png)
