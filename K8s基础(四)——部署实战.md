# 服务部署至Kubernetes

作为K8s学习笔记，部署相关，以备后查。

<!-- more -->

将传统服务迁移至K8s集群需要考虑的：

1. 服务类型。有哪几类服务需要部署
2. yaml文件。为不同服务编写yaml文件以便部署
3. 镜像
   - 通用服务，例如MySQL等，通过官方拉取镜像
   - 自定义服务，例如Spring Boot等，通过Dockerfile构建镜像
4. 服务注册与发现
   - Nacos  https://nacos.io/en-us/docs/what-is-nacos.html 

### 部署WordPress+MySQL



### 部署Spring Boot项目



### 部署Nacos