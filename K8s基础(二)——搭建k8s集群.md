# K8s基础(二)——搭建K8s集群

作为K8s学习笔记，多机集群搭建，以备后查。

<!-- more --> 

### 准备工作

- 官方文档 https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- 示例中使用kubeadmin搭建由3台机器组成的K8s集群，1台manager节点，2台worker节点
- 配置要求
  -  https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin 

### 搭建集群

#### 1. 软件版本

```
Docker       18.09.0
---
kubeadm-1.14.0-0 
kubelet-1.14.0-0 
kubectl-1.14.0-0
---
k8s.gcr.io/kube-apiserver:v1.14.0
k8s.gcr.io/kube-controller-manager:v1.14.0
k8s.gcr.io/kube-scheduler:v1.14.0
k8s.gcr.io/kube-proxy:v1.14.0
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.3.10
k8s.gcr.io/coredns:1.3.1
---
calico:v3.9
```

#### 2. 更新并安装依赖

在3台机器中执行

```
yum -y update
yum install -y conntrack ipvsadm ipset jq sysstat curl iptables libseccomp
```

#### 3. 安装Docker&设置阿里镜像仓库

1. 安装必要依赖

   ```
   sudo yum install -y yum-utils \
     device-mapper-persistent-data \
     lvm2
   ```

2. 设置Docker仓库

   ```
   sudo yum-config-manager \
       --add-repo \
       https://download.docker.com/linux/centos/docker-ce.repo
   ```

   为方便后续操作，这里设置下阿里云镜像加速器

   ```
   sudo mkdir -p /etc/docker
   sudo tee /etc/docker/daemon.json <<-'EOF'
   {
     "registry-mirrors": ["此处替换成个人实际地址"]
   }
   EOF
   sudo systemctl daemon-reload
   ```

3. 安装Docker

   ```
   yum install -y docker-ce-18.09.0 docker-ce-cli-18.09.0 containerd.io
   ```

4. 启动Docker

   ```
   sudo systemctl start docker
   ```

#### 4. 修改hosts文件

1. manager节点

   ```
   #设置manager的hostname，并修改hosts文件
   sudo hostnamectl set-hostname m
   vi /etc/hosts
   #添加内容
   45.77.87.94 m
   45.76.68.23 w1
   144.202.117.198 w2
   ```

2. worker节点

   ```
   # 设置worker01/02的hostname，并且修改hosts文件
   sudo hostnamectl set-hostname w1
   sudo hostnamectl set-hostname w2
   
   vi /etc/hosts
   45.77.87.94 m
   45.76.68.23 w1
   144.202.117.198 w2
   ```

#### 5. 系统基础配置

1. 关闭防火墙

   ```
   systemctl stop firewalld && systemctl disable firewalld
   ```

2. 关闭selinux

   ```
   setenforce 0
   sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
   ```

3. 关闭swap

   ```
   swapoff -a
   sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab
   ```

4. 配置iptables的ACCEPT规则

   ```
   iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat && iptables -P FORWARD ACCEPT
   ```

5. 配置系统参数

   ```
   cat <<EOF >  /etc/sysctl.d/k8s.conf
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   EOF
   ```

#### 6. 安装kubeadm、kubelet、kubectl

1. 配置yum源

   ```
   cat <<EOF > /etc/yum.repos.d/kubernetes.repo
   [kubernetes]
   name=Kubernetes
   baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
   enabled=1
   gpgcheck=0
   repo_gpgcheck=0
   gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
          http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
   EOF
   ```

2. 安装kubeadm&kubelet&kubectl

   ```
   yum install -y kubeadm-1.14.0-0 kubelet-1.14.0-0 kubectl-1.14.0-0
   ```

3.  设置docker和k8s同一个cgroup

   ```
   #docker
   vi /etc/docker/daemon.json
   #添加内容
   "exec-opts": ["native.cgroupdriver=systemd"],
     
   systemctl restart docker
       
   #kubelet，输出directory not exist正常，继续进行即可
   sed -i "s/cgroup-driver=systemd/cgroup-driver=cgroupfs/g" /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
   	
   systemctl enable kubelet && systemctl start kubelet
   ```

#### 7. 拉取国外镜像

1. 创建kubeadm.sh脚本 

   ```
   #!/bin/bash
   
   set -e
   
   KUBE_VERSION=v1.14.0
   KUBE_PAUSE_VERSION=3.1
   ETCD_VERSION=3.3.10
   CORE_DNS_VERSION=1.3.1
   
   GCR_URL=k8s.gcr.io
   ALIYUN_URL=registry.cn-hangzhou.aliyuncs.com/google_containers
   
   images=(kube-proxy:${KUBE_VERSION}
   kube-scheduler:${KUBE_VERSION}
   kube-controller-manager:${KUBE_VERSION}
   kube-apiserver:${KUBE_VERSION}
   pause:${KUBE_PAUSE_VERSION}
   etcd:${ETCD_VERSION}
   coredns:${CORE_DNS_VERSION})
   
   for imageName in ${images[@]} ; do
     docker pull $ALIYUN_URL/$imageName
     docker tag  $ALIYUN_URL/$imageName $GCR_URL/$imageName
     docker rmi $ALIYUN_URL/$imageName
   done
   ```

2. 运行脚本 `sh ./kubeadm.sh`

3. 查看镜像 `docker images`

#### 8. 初始化manager节点

1. 初始化manager节点（保存好*kubeadm join*信息）

   ```
   kubeadm init --kubernetes-version=1.14.0 --apiserver-advertise-address= 45.77.87.94 --pod-network-cidr=10.244.0.0/16
   ```

   需要重新初始化集群状态时，先执行`kubeadm reset`再执行初始化命令

2. 根据输出日志执行

   ```
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

3. 查看Pod验证是否为ready状态

   ```
   kubectl get pods -n kube-system
   ```

   除 coredns(网络插件)外，其它组件必须都处于running状态才能进行下一步

4. 健康检查

   ```
   curl -k https://localhost:6443/healthz
   ```

#### 9. 部署Calico网络插件

可选网络插件

- https://kubernetes.io/docs/concepts/cluster-administration/addons/

示例中使用

- https://docs.projectcalico.org/v3.9/getting-started/kubernetes/ 

在k8s中安装Calico

```
kubectl apply -f https://docs.projectcalico.org/v3.9/manifests/calico.yaml
```

确认Calico是否安装成功，必须所有Pod处于running状态才能进行下一步

```
kubectl get pods --all-namespaces -w
```

#### 10. 将worker节点加入manager节点

1. 分别在worker01、worker02节点中执行

   ```
   kubeadm join 45.77.87.94:6443 --token n6bskd.bpkl4gx8yazmhrne     --discovery-token-ca-cert-hash sha256:5f36181ccd5b335d5109ab850d12b225eb5e8f669991fe5629f6f18e54019e00
   ```

2. 在manager节点上检查集群信息

   ```
   kubectl get nodes
   ```

   可以看到输出的集群信息：

   ```
   [root@m ~]# kubectl get nodes
   NAME   STATUS   ROLES    AGE    VERSION
   m      Ready    master   123m   v1.14.0
   w1     Ready    <none>   118m   v1.14.0
   w2     Ready    <none>   118m   v1.14.0
   ```

#### 11. K8s部署Pod

1. 创建pod_nginx.yaml

   ```
   apiVersion: apps/v1
   kind: ReplicaSet
   metadata:
     name: nginx
     labels:
       tier: frontend
   spec:
     replicas: 3
     selector:
       matchLabels:
         tier: frontend
     template:
       metadata:
         name: nginx
         labels:
           tier: frontend
       spec:
         containers:
         - name: nginx
           image: nginx
           ports:
           - containerPort: 80
   ```

2. 根据pod_nginx.yaml创建Pod

   ```
   kubectl apply -f pod_nginx.yaml
   ```

3. 查看Pod

   ```
   kubectl get pods
   kubectl get pods -o wide
   kubectl describe pod nginx
   ```

4. 通过`--replicas`指定数量进行扩缩容

   ```
   kubectl scale rs nginx --replicas=5
   ```

5. 删除Pod

   ```
   kubectl delete -f pod_nginx.yaml
   ```

