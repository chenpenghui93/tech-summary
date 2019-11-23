### YAML

#### 简介

YAML是专门用来写配置文件的语言，其设计目标就是方便读写，本质上是一种通用的数据串行化格式。

基本语法规则：

- 大小写敏感
- 使用缩进表示层级关系
- 缩进时只能使用空格，不能使用Tab键
- 缩进的空格数目不重要，只要同层级的元素左侧对其即可(一般空两格)

支持的数据结构：

- 对象——键值对的集合
- 数组——一组按序列排列的值
- 纯量——单个的、不可再分的值

#### 参考

-  https://yaml.org/spec/1.2/spec.html 
-  https://en.wikipedia.org/wiki/YAML 

### Container

在Docker中通过`docker run`或者定义yaml文件来运行一个容器，单机环境中可以使用docker-compose，多机环境中可以使用docker swarm创建。在K8s中同样以一个yaml文件维护，Container运行在Pod中。

#### Pod

<https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/>

> A Pod is the basic execution unit of a Kubernetes application
> A Pod encapsulates an application’s container (or, in some cases, multiple containers), storage resources, a unique network IP, and options that govern how the container(s) should run

#### 体验Pod

1. 创建一个Pod的yaml文件，命名为nginx_pod.yaml

   ```
   apiVersion: v1
   kind: Pod
   metadata:
     name: nginx-pod
     labels:
       app: nginx
   spec:
     containers:
     - name: nginx-container
       image: nginx
       ports:
       - containerPort: 80
   ```

2. 根据yaml创建Pod

   ```
   kubectl apply -f nginx_pod.yaml
   ```

3. 查看Pod

   ```
   kubectl get pods
   kubectl get pods -o wide
   kubectl describe pod nginx-pod
   ```

4. 删除Pod

   ```
   kubectl delete -f nginx_pod.yaml
   ```

#### Storage&Networking

##### Stroage

> A Pod can specify a set of shared storage Volumes. All containers in the Pod can access the shared volumes, allowing those containers to share data.

- 官网 https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/#storage

##### Networking

> Each Pod is assigned a unique IP address. Every container in a Pod shares the network namespace, including the IP address and network ports. 

- 官网 https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/#networking

### Controllers

#### ReplicationController(RC)

ReplicationController声明某种Pod的数量在任意时刻都符合某个预期的数值，因此，RC的定义中包含以下内容：

- Pod的副本数(replicas)
- 用于筛选目标Pod的Label Selector
- 当Pod数量少于目标值时，用于创建新Pod的模板(template)

##### 体验RC

1. 创建名为nginx_replication.yaml的文件

   ```
   apiVersion: v1
   kind: ReplicationController
   metadata:
     name: nginx
   spec:
     replicas: 3
     selector:
       app: nginx
     template:
       metadata:
         name: nginx
         labels:
           app: nginx
       spec:
         containers:
         - name: nginx
           image: nginx
           ports:
           - containerPort: 80
   ```

2. 根据nginx_replication.yaml创建Pod

   ```
   kubectl apply -f nginx_replication.yaml
   ```

3. 查看Pod

   ```
   kubectl get pods -o wide
   kubectl get rc
   ```

4. 删除某个Pod再进行查看

   ```
   kubectl delete pods nginx-vz22h
   kubectl get pods
   ```

5. 对Pod进行扩缩容

   ```
   kubectl scale rc nginx --replicas=5
   ```

6. 删除Pod

   ```
   kubectl delete -f nginx_replication.yaml
   ```

#### ReplicaSet(RS)

> A ReplicaSet’s purpose is to maintain a stable set of replica Pods running at any given time. As such, it is often used to guarantee the availability of a specified number of identical Pods.

- 官网 https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/

- RS可以理解为RC的升级版，二者区别在于RS支持基于集合的Label Selector（Set-based selector），而RC只支持基于等式的Label Selector（equality-based selector），这使得ReplicaSet的功能更强。
- 一般情况下，我们很少单独使用Replica Set，它主要是被Deployment这个更高的资源对象所使用，从而形成一整套Pod创建、删除、更新的编排机制

#### Deployment

> A Deployment provides declarative updates for Pods and ReplicaSets.
>
> You describe a desired state in a Deployment, and the Deployment Controller changes the actual state to the desired state at a controlled rate. You can define Deployments to create new ReplicaSets, or to remove existing Deployments and adopt all their resources with new Deployments.

- 官网 https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- 使用Deployment可以随时知道当前Pod的"部署"进度

##### 体验Deployment

1. 创建nginx_deployment.yaml文件

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx-deployment
     labels:
       app: nginx
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: nginx
     template:
       metadata:
         labels:
           app: nginx
       spec:
         containers:
         - name: nginx
           image: nginx:1.7.9
           ports:
           - containerPort: 80
   ```

2. 根据nginx_deployment.yaml文件创建pod

   ```
   kubectl apply -f nginx_deployment.yaml
   ```

3. 查看Pod

   ```
   kubectl get pods -o wide
   kubectl get deployment
   kubectl get rs
   kubectl get deployment -o wide
   ```

4. 当前nginx的版本

   ```
   kubectl get deployment -o wide
   
   NAME    READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES      SELECTOR
   nginx-deployment   3/3         3     3  3m27s      nginx    nginx:1.7.9   app=nginx
   ```

5. 更新nginx的image版本

   ```
   kubectl set image deployment nginx-deployment nginx=nginx:1.9.1
   ```

### Labels and Selectors

> Labels are key/value pairs that are attached to objects, such as pods. 

- 官网 https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/

- 查看Pod的Label标签

  ```
  kubectl get pods --show-labels
  ```

  ```
  apiVersion: apps/v1
  kind: Deployment         
  metadata: 
    name: nginx-deployment 
    labels:
      app: nginx           # label中的key为app，value为nginx
  spec:
    replicas: 3
    selector:              # 匹配具有同一个label属性的pod标签
      matchLabels:
        app: nginx         
    template:              # 定义pod的模板
      metadata:
        labels:
          app: nginx       # 定义当前pod的label属性，app为key，value为nginx
      spec:
        containers:
        - name: nginx
          image: nginx:1.7.9
          ports:
          - containerPort: 80
  ```

### Namespace

命名空间就是为了隔离不通的资源，例如，Pod、Service、Deployment等等。可以在输入命令时使用参数 `-n` 指定命名空间，如不指定，则使用默认的命名空间：default

- 查看当前命名空间

  ```
  kubectl get namespaces
  ```

- 查看指定命名空间下的Pod

  ```
  kubectl get pods -n kube-system
  ```

#### 创建命名空间

1. 创建myns-namespace.yaml文件

   ```
   apiVersion: v1
   kind: Namespace
   metadata:
     name: myns
   ```

2. 执行创建命名空间

   ```
   kubectl apply -f myns-namespace.yaml
   ```

3. 查看命名空间

   ```
   kubectl get ns
   ```

#### 指定命名空间下的资源

1. 修改nginx-pod.yaml 

   ```
   apiVersion: v1
   kind: Pod
   metadata:
     name: nginx-pod
     namespace: myns
   spec:
     containers:
     - name: nginx-container
       image: nginx
       ports:
       - containerPort: 80
   ```

2. 执行创建Pod

   ```
   kubectl apply -f nginx-pod.yaml 
   ```

3. 查看myns命名空间下的Pod和资源

   ```
   kubectl get pods
   kubectl get pods -n myns
   kubectl get all -n myns
   kubectl get pods --all-namespaces
   ```

### Network

#### 同一个Pod中的Container通信

K8s的最小操作单元是Pod，那么同一个Pod中的多个容器如何进行通信？从官网可以知道同一个Pod中的容器是共享网络IP地址和端口号的，自然可以互相通信。如果需要通过容器的名称进行通信，就需要将Pod中所有的容器加入到同一个容器的网络中，这个容器被称为Pod中的pause container。

#### 集群内Pod之间的通信

准备两个Pod，一个nginx，一个busybox

- nginx_pod.yaml

  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx-pod
    labels:
      app: nginx
  spec:
    containers:
    - name: nginx-container
      image: nginx
      ports:
      - containerPort: 80
  ```

- busybox_pod.yaml

  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: busybox
    labels:
      app: busybox
  spec:
    containers:
    - name: busybox
      image: busybox
      command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  ```

- 将两个Pod运行起来并查看运行情况

  ```
  kubectl apply -f nginx_pod.yaml
  kubectl apply -f busy_pod.yaml
  kubectl get pods -o wide
  ```

- 运行信息

  ```
  [root@m ~]# kubectl get pods -o wide
  NAME        READY   STATUS    RESTARTS   AGE     IP               NODE   NOMINATED NODE   READINESS GATES
  busybox     1/1     Running   0          30s     192.168.80.202   w2     <none>           <none>
  nginx-pod   1/1     Running   0          9m18s   192.168.190.74   w1     <none>           <none>
  ```

w1上运行nginx-pod，ip为192.168.190.74；w2上运行busybox，ip为192.168.80.202

##### 同一集群中的同一台机器

1. 切换到w1节点：`ping 192.168.190.74`
2. 再在w1节点上：`curl 192.168.190.74`

发现可以正常访问

##### 同一集群中的不同机器

1. 切换到w2节点：`ping 192.168.190.74`
2. 再在w2节点上：`curl 192.168.190.74`
3. 切换到manager节点上：`ping 192.168.190.74`、`curl 192.168.190.74`

发现仍可以正常访问

##### K8s集群内通信原理

- 通过网络插件实现，例如Calico
-  https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-networking-model 

#### 集群内部Service-Cluster IP

对于上述的Pod虽然实现了集群内部互相通信，但是Pod是不稳定的，比如通过Deployment管理Pod，随时可能对Pod进行扩缩容，这时候Pod的IP地址是变化的。能够有一个固定的IP，使得集群内能够访问。也就是之前在架构描述的时候所提到的，能够把相同或者具有关联的Pod，打上Label，组成Service。而Service有固定的IP，不管Pod怎么创建和销毁，都可以通过Service的IP进行访问

官网 <https://kubernetes.io/docs/concepts/services-networking/service/>

##### 体验Service-Cluster IP

1. 创建whoami-deployment.yaml并运行

   ```
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: whoami-deployment
     labels:
       app: whoami
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: whoami
     template:
       metadata:
         labels:
           app: whoami
       spec:
         containers:
         - name: whoami
           image: jwilder/whoami
           ports:
           - containerPort: 8000
   ```

2. 查看Pod及Service

   ```
   [root@m ~]# kubectl get pods -o wide
   NAME                                 READY   STATUS    RESTARTS   AGE     IP               NODE   NOMINATED NODE   READINESS GATES
   whoami-deployment-678b64444d-g45t8   1/1     Running   0          2m25s   192.168.80.203   w2     <none>           <none>
   whoami-deployment-678b64444d-hf6x9   1/1     Running   0          2m25s   192.168.190.75   w1     <none>           <none>
   whoami-deployment-678b64444d-hglqc   1/1     Running   0          2m25s   192.168.190.76   w1     <none>           <none>
   ```

   ```
   [root@m ~]# kubectl get svc
   NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
   kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   5d2h
   ```

3. 集群内部访问

   ```
   curl 45.77.87.94:8000/45.76.68.23:8000/144.202.117.198:8000
   ```

4. 创建whoami的service

   注意：该Service IP地址只能在集群内部访问

   ```
   kubectl expose deployment whoami-deployment
   ```

   ```
   [root@m ~]# kubectl get svc
   NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
   kubernetes          ClusterIP   10.96.0.1        <none>        443/TCP    5d2h
   whoami-deployment   ClusterIP   10.104.237.142   <none>        8000/TCP   8s
   ```

   ```
   #删除svc
   kubectl delete service whoami-deployment
   ```

5. 通过Service的Cluster IP访问

   ```
   [root@m ~]# curl 10.104.237.142:8000
   I'm whoami-deployment-678b64444d-hglqc
   [root@m ~]# curl 10.104.237.142:8000
   I'm whoami-deployment-678b64444d-hf6x9
   [root@m ~]# curl 10.104.237.142:8000
   I'm whoami-deployment-678b64444d-g45t8
   ```

6. 查看whoami deployment信息

   ```
   [root@m ~]# kubectl describe svc whoami-deployment
   Name:              whoami-deployment
   Namespace:         default
   Labels:            app=whoami
   Annotations:       <none>
   Selector:          app=whoami
   Type:              ClusterIP
   IP:                10.104.237.142
   Port:              <unset>  8000/TCP
   TargetPort:        8000/TCP
   Endpoints:         192.168.190.75:8000,192.168.190.76:8000,192.168.80.203:8000
   Session Affinity:  None
   Events:            <none>
   ```

7. 将whoami扩容

   ```
   kubectl scale deployment whoami-deployment --replicas=5
   ```

8. 再次访问 `curl 10.104.237.142:8000`

9. 查看Service具体信息 `kubectl describe svc whoami-deployment`

10. 对于Service的创建，可以通过kubectl expose，也可以通过定义yaml文件

    ```
    apiVersion: v1
    kind: Service
    metadata:
      name: my-service
    spec:
      selector:
        app: MyApp
      ports:
        - protocol: TCP
          port: 80
          targetPort: 9376
      type: Cluster
    ```

总结：Service存在的意义就是为了解决Pod的不稳定性，上述讨论的就是Service的一种类型Cluster IP，只能供集群内部访问

#### Pod访问外部服务

直接访问即可...

#### 外部访问集群内的Pod

##### Service-NodePort

- Service的一种类型，可以通过NodePort的方式对外暴露端口
- 因为外部能够访问到集群中物理机的IP，所以就在集群中每台物理机上暴露一个相同的端口

###### 体验Service-NodePort

1. 根据whoami-deployment.yaml创建pod

2. 创建NodePort类型的Service，名称为whoami-deployment

   ```
   kubectl delete svc whoami-deployment
   kubectl expose deployment whoami-deployment --type=NodePort
   ```

   ```
   [root@m ~]# kubectl get svc
   NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
   kubernetes          ClusterIP   10.96.0.1       <none>        443/TCP          5d3h
   whoami-deployment   NodePort    10.104.152.85   <none>        8000:30983/TCP   11s
   ```

3. 上述端口30983即为物理机暴露在外的可访问端口，验证物理机端口监听情况

   ```
   lsof -i tcp:30983
   netstat -ntlp|grep 30983
   ```

4. 浏览器访问物理机的IP进行测试

   ```
   http://10.104.152.85:30983
   curl 10.104.152.85:30983
   ```

总结：NodePort虽然能够实现外部访问Pod的需求，但是占用了各个物理机上的端口，并不太友好

##### Service-LoadBalance

一般由第三方云提供商支持，有约束性

##### Ingress

> An API object that manages external access to the services in a cluster, typically HTTP.
> Ingress can provide load balancing, SSL termination and name-based virtual hosting.

- 官网 <https://kubernetes.io/docs/concepts/services-networking/ingress/>
- GitHub Ingress Nginx <https://github.com/kubernetes/ingress-nginx>
- Nginx Ingress Controller <https://kubernetes.github.io/ingress-nginx/

1. 以Deployment方式创建Pod，该Pod为Ingress Nginx Controller，要想让外界访问，可以通过Service的NodePort或者HostPort方式，这里选择HostPort，比如指定w1节点运行

   ```
   # 确保nginx-controller运行到w1节点上
   kubectl label node w1 name=ingress
   
   kubectl apply -f mandatory.yaml
   
   kubectl get all -n ingress-nginx
   ```

2. 查看w1的80和443端口，确认80/443端口未被占用

   ```
   lsof -i tcp:80
   lsof -i tcp:443
   ```

3. 创建tomcat的Pod和Service

   tomcat.yaml

   ```
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: tomcat-deployment
     labels:
       app: tomcat
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: tomcat
     template:
       metadata:
         labels:
           app: tomcat
       spec:
         containers:
         - name: tomcat
           image: tomcat
           ports:
           - containerPort: 8080
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: tomcat-service
   spec:
     ports:
     - port: 80   
       protocol: TCP
       targetPort: 8080
     selector:
       app: tomcat
   ```

   ```
   vi tomcat.yaml
   kubectl apply -f tomcat.yaml
   kubectl get svc 
   kubectl get pods
   ```

4. 创建Ingress以及定义转发规则

   nginx_ingress.yaml

   ```
   #ingress
   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
     name: nginx-ingress
   spec:
     rules:
     - host: tomcat.jack.com
       http:
         paths:
         - path: /
           backend:
             serviceName: tomcat-service
             servicePort: 80
   ```

   ```
   kubectl apply -f nginx-ingress.yaml
   
   kubectl get ingress
   
   kubectl describe ingress nginx-ingress
   ```

5. 修改hosts文件，添加dns解析

6. 浏览器访问验证

总结：使用Ingress网络时，只要定义好ingress、service、pod即可，前提是需要先配置好Ingress Nginx Controller。































