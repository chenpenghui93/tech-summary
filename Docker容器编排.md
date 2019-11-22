作为Docker学习笔记，容器编排相关，以备后查。虚拟机地址为实验环境，折腾较多，示例不保证能随时访问。 

### 容器编排

容器编排是指容器的集群化和调度，或者理解为管理容器化应用和组件的方式。常见的编排工具有Docker swarm mode、Kubernetes、Mesosphere DCOS等。本文只涉及Docker自身提供的工具，先对容器编排有个印象，便于后续理解其它容器编排工具。

### Docker Compose

docker-compose主要解决单主机内容器编排问题。一般通过yaml配置文件来使用，可以在yaml文件中记录多个容器启动的配置信息(镜像、启动命令、端口、网络等)，然后通过执行`docker-compose`命令就会像执行脚本一样批量创建或销毁容器。

#### 业务背景

参考官网例子，实现单机内统计应用访问次数

#### 传统实现

##### 写python代码&构建镜像

1. 创建指定文件夹

   `mkdir -p /tmp/composetest`
   `cd /tmp/composetest`

2. 创建app.py

   ```
   import time
   
   import redis
   from flask import Flask
   
   app = Flask(__name__)
   cache = redis.Redis(host='redis', port=6379)
   
   def get_hit_count():
       retries = 5
       while True:
           try:
               return cache.incr('hits')
           except redis.exceptions.ConnectionError as exc:
               if retries == 0:
                   raise exc
               retries -= 1
               time.sleep(0.5)
   
   @app.route('/')
   def hello():
       count = get_hit_count()
       return 'Hello World! I have been seen {} times.\n'.format(count)
   ```

3. 新建requirements.txt

   ```
   flask
   redis
   ```

4. 编写Dockerfile

   ```
   FROM python:3.7-alpine
   WORKDIR /code
   ENV FLASK_APP app.py
   ENV FLASK_RUN_HOST 0.0.0.0
   RUN apk add --no-cache gcc musl-dev linux-headers
   COPY requirements.txt requirements.txt
   RUN pip install -r requirements.txt
   COPY . .
   CMD ["flask", "run"]
   ```

5. 根据Dockerfile构建镜像

   ```
   docker build -t python-app-image .
   ```

##### 拉取redis镜像

- `docker pull redis:alpine`

##### 创建container镜像

1. 创建网络

   `docker network create --subnet=172.20.0.0/24 app-net`

2. 创建python程序的container，指定端口和网段

   `docker run -d --name web -p 5000:5000 --network app-net python-app-image`

3. 创建redis的container，指定网段

   `docker run -d --name redis --network app-net redis:alpine`

##### 访问测试

- 访问 http://45.32.103.56:5000/ ，可以看到类似如下输出

  ```
  Hello World! I have been seen 6 times.
  ```

#### 使用docker-compose实现

##### 安装docker-compose

-  https://docs.docker.com/compose/install/ 

##### 前期准备

1. 与传统方式类似，需要创建目录、进入相应目录编写app.py文件、创建requirements.txt文件、编写Dockerfile

2. 创建docker-compose.yml

   ```
   version: '3'
   services:
     web:
       build: .
       ports:
         - "5000:5000"
       networks:
         - app-net
   
     redis:
       image: "redis:alpine"
       networks:
         - app-net
   
   networks:
     app-net:
       driver: bridge
   ```

3. 使用docker-compose创建容器

   `docker-compose up -d`

4. windows浏览器内访问http://45.32.103.56:5000/ 可以看到类似输出

##### 详解docker-compose文件

- version: '3'
  - 指定docker-compose版本
- services
  - 一个service表示一个容器
- networks
  - 创建并指定网络
- volumes
  - 等同于`-v v1:/var/lib/mysql`
- image
  - 指定使用的镜像
- ports
  - 指定映射端口
- environment
  - 等同`-e`指定参数

##### 使用scale扩容

1. 修改docker-compose.yml去掉ports

   ```
   version: '3'
   services:
     web:
       build: .
       networks:
         - app-net
   
     redis:
       image: "redis:alpine"
       networks:
         - app-net
   
   networks:
     app-net:
       driver: bridge
   ```

2. 创建service

   `docker-compose up -d`

3. 对python容器进行扩容

   `docker-compose up --scale web=5 -d`

4. 查看相关信息

   `docker-compose ps`
   `docker-compose logs web`

### Docker Swarm

docker-swarm是解决多主机容器编排问题的。swarm是基于docker平台实现的集群技术，可以通过几条简单的指令快速的创建一个docker集群，接着在集群的共享网络上部署应用，最终实现分布式的服务。但是技术上还不太成熟，目前业内更多的是使用k8s来管理集群和调度容器。

#### 搭建Swarm集群

##### 环境准备

1. 需要提前准备3台能互相ping通的虚拟机

   - 官网提供的环境 http://labs.play-with-docker.com

2. 进入manager节点 

   `docker swarm init --advertise-addr=192.168.0.11` 

   manager node也可以作为worker node提供服务

3. 从manager节点启动日志中拿到worker node加入manager node的信息 

   `docker swarm join --token SWMTKN-1-0a5ph4nehwdm9wzcmlbj2ckqqso38pkd238rprzwcoawabxtdq-arcpra6yzltedpafk3qyvv0y3 192.168.0.11:2377`

4. 分别进入两个worker节点中执行

   `docker swarm join --token SWMTKN-1-0a5ph4nehwdm9wzcmlbj2ckqqso38pkd238rprzwcoawabxtdq-arcpra6yzltedpafk3qyvv0y3 192.168.0.11:2377` 

   可以看到worker节点中日志

    `This node joined a swarm as a worker.`

5. 进入到manager node查看集群状态

   `docker node ls`

6. node类型的转换

   - worker升级为manager `docker node promote worker01-node`
   - manager降级为worker `docker node demote worker01-node`

##### Swarm基本操作

1. 创建一个tomcat的service

   `docker service create --name my-tomcat tomcat`

2. 查看当前swarm的service

   `docker service ls`

3. 查看service启动日志

   `docker service logs my-tomcat`

4. 查看service详情

   `docker service inspect my-tomcat`

5. 查看my-tomcat运行节点

   `docker service ps my-tomcat`

6. 扩容

   `docker service scale my-tomcat=3`

7. 如果某个node上的tomcat挂掉，集群会自动扩展 

   - worker节点上执行 `docker rm -f [containerid]` 
   - manager节点上查看 `docker service ls docker` , `service ps my-tomcat`

8. 删除service

   `docker service rm my-tomcat`

#### 多主机通信overlay网络

以使用wordpress+mysql搭建博客为例

##### 传统方式创建

在一台基于centos的docker主机上分别创建容器

1. 创建mysql容器

   ```
   docker run -d --name mysql -v v1:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=examplepass -e MYSQL_DATABASE=db_wordpress mysql:5.6
   ```

2. 创建wordpress容器

   ```
   docker run -d --name wordpress --link mysql -e WORDPRESS_DB_HOST=mysql:3306 -e WORDPRESS_DB_USER=root -e WORDPRESS_DB_PASSWORD=examplepass -e WORDPRESS_DB_NAME=db_wordpress -p 8080:80 wordpress
   ```

3. 查看默认bridge网络（两个容器位于其中）

   ```
   docker network inspect bridge
   ```

4. 访问测试

   - http://45.32.103.56:8080/

##### 使用docker-compose创建

1. 创建wordpress-mysql文件夹

   `mkdir -p /tmp/wordpress-mysql`
   `cd /tmp/wordpress-mysql`

2. 创建docker-compose.yml文件

   ```
   version: '3.1'
   
   services:
   
     wordpress:
       image: wordpress
       restart: always
       ports:
         - 8080:80
       environment:
         WORDPRESS_DB_HOST: db
         WORDPRESS_DB_USER: exampleuser
         WORDPRESS_DB_PASSWORD: examplepass
         WORDPRESS_DB_NAME: exampledb
       volumes:
         - wordpress:/var/www/html
   
     db:
       image: mysql:5.7
       restart: always
       environment:
         MYSQL_DATABASE: exampledb
         MYSQL_USER: exampleuser
         MYSQL_PASSWORD: examplepass
         MYSQL_RANDOM_ROOT_PASSWORD: '1'
       volumes:
         - db:/var/lib/mysql
   
   volumes:
     wordpress:
     db:
   ```

3. 根据docker-compose.yml文件创建service

   `docker-compose up -d`

4. 访问测试

   - http://45.32.103.56:8080/

5. 可以查看网络

   - `docker network ls`
   - `docker network inspect wordpress-mysql_default`

##### 使用Swarm创建

1. 创建一个overlay网络，用于docker swarm中多机通信

   - `docker network create -d overlay my-overlay-net`
   - `docker network ls`

2. 创建mysql的service

   ```
   docker service create --name mysql --mount type=volume,source=v1,destination=/var/lib/mysql --env MYSQL_ROOT_PASSWORD=examplepass --env MYSQL_DATABASE=db_wordpress --network my-overlay-net mysql:5.6
   ```

   查看service

   `docker service ls`
   `docker service ps mysql`

3. 创建wordpress的service

   ```
   docker service create --name wordpress --env WORDPRESS_DB_USER=root --env WORDPRESS_DB_PASSWORD=examplepass --env WORDPRESS_DB_HOST=mysql:3306 --env WORDPRESS_DB_NAME=db_wordpress -p 8080:80 --network my-overlay-net wordpress
   ```

   查看service

   `docker service ls`
   `docker service ps mysql`

4. 访问测试，可以发现三个节点均能访问到

   - http://45.32.103.56:8080/
   - http://45.32.103.57:8080/
   - http://45.32.103.58:8080/

5. 查看my-overlay-net

   `docker network inspect my-overlay-net`

6. docker swarm中有自己的分布式存储机制，故无需使用etcd

####  Routing Mesh

> Docker Engine swarm mode makes it easy to publish ports for services to make them available to resources outside the swarm. All nodes participate in an ingress routing mesh. The routing mesh enables each node in the swarm to accept connections on published ports for any service running in the swarm, even if there’s no task running on the node. The routing mesh routes all incoming requests to published ports on available nodes to an active container.

访问示意图：

![](/images/docker/ingress_network.png "routing mesh")  

-  https://docs.docker.com/engine/swarm/ingress/ 

#### docker stack deploy

1. 新建service.yml文件

   ```
   version: '3'
   
   services:
   
     wordpress:
       image: wordpress
       ports:
         - 8080:80
       environment:
         WORDPRESS_DB_HOST: db
         WORDPRESS_DB_USER: exampleuser
         WORDPRESS_DB_PASSWORD: examplepass
         WORDPRESS_DB_NAME: exampledb
       networks:
         - ol-net
       volumes:
         - wordpress:/var/www/html
       deploy:
         mode: replicated
         replicas: 3
         restart_policy:
           condition: on-failure
           delay: 5s
           max_attempts: 3
         update_config:
           parallelism: 1
           delay: 10s
   
     db:
       image: mysql:5.7
       environment:
         MYSQL_DATABASE: exampledb
         MYSQL_USER: exampleuser
         MYSQL_PASSWORD: examplepass
         MYSQL_RANDOM_ROOT_PASSWORD: '1'
       volumes:
         - db:/var/lib/mysql
       networks:
         - ol-net
       deploy:
         mode: global
         placement:
           constraints:
             - node.role == manager
   
   volumes:
     wordpress:
     db:
   
   networks:
     ol-net:
       driver: overlay
   ```

2. 根据service.yml创建service

   `docker statck deploy -c service.yml my-service`

3. stack常用操作

   - 查看stack具体信息 `docker stack ls`
   - 查看具体的service `docker stack services my-service`
   - 查看某个service `docker service inspect my-service-db`