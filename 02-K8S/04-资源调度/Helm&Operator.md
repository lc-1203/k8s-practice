

## Helm

### 为什么需要Helm

由于Kubernetes缺少对发布的应用版本管理和控制，使得部署的应用维护和更新等面临诸多的挑战，主要面临以下问题:

- 如何将这些服务作为一个整体管理?
- 这些资源文件如何高效复用?
- 不支持应用级别的版本管理

### helm介绍

Helm是一个Kubernetes的包管理工具，就像Linux下的包管理器，如yum/apt等，可以很方便的将之前打包好的yaml文件部署到kubernetes上。
Helm有3个重要概念:

- helm:一个命令行客户端工具，主要用于Kubernetes应用chart的创建、打包、发布和管理。
- Chart:应用描述，一系列用于描述k8s资源相关文件的集合。
- Release:基于Chart的部署实体，一个chart被 Helm运行后将会生成对应的一个release;将在k8s中创建出真实运行的资源对象。

### 创建chart示例

![image-20210530100110088](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210530100110.png)

![image-20210530100148354](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/20210530100148.png)

## Helm&Operator

### 部署前提

- 工作原理：架构、配置（环境变量、配置文件）、端口号、启动命令
- 镜像：dockerfile自建、官方、开源第三方
- 部署方式：有无状态、配置分离、部署文件来源、如何部署
- 如何被使用：协议、内部还是外部

### 部署方式

- Helm：安装Helm客户端-->添加helm仓库-->编辑chart模版-->Install
- Operator：创建Operator控制机-->创建自定义资源（crd）-->执行相关逻辑

### 中间件&生产环境

- 非生产环境：较推荐使用k8s管理
- 生产环境：需考虑性能、持久化、稳定性等问题

如mysql对磁盘IO要求比较高

