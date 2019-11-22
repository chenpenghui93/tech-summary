# Docker基础(四)——数据持久化&单机集群部署

作为Docker学习笔记，数据持久化、部署实战相关，以备后查。 虚拟机地址为实验环境，折腾较多，示例不保证能随时访问。 

### Docker数据持久化

volumes ???

bind mounts ???

#### Volumes

1. 创建mysql容器

   `docker run -d --name mysql01 -e MYSQL_ROOT_PASSWORD=cph123 mysql`

2. 列出所有volume 

   `docker volume ls`

   ```
   DRIVER              VOLUME NAME
   local               6f97012d2f8521c73a3cb84e0073fe66233bdb26eaae9c25d57f56388ea26d75
   ```

3. 检查指定volume

   `docker volume inspect 6f97012d2f8521c73a3cb84e0073fe66233bdb26eaae9c25d57f56388ea26d75`

4. 使用-v参数指定volume名称

   `docker run -d --name mysql01 -v mysql01_volume:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=cph123 mysql`

5. 再次查看volume

    `docker volume ls`

   ```
   DRIVER              VOLUME NAME
   local               6f97012d2f8521c73a3cb84e0073fe66233bdb26eaae9c25d57f56388ea26d75
   local               mysql01_volume
   ```

6. 检查指定volume 

   `docker inspect mysql01_volume`

##### 持久化测试

1. 进入容器中  `docker exec -it mysql01 bash` 

2. 登录mysql  `mysql -uroot -pcph123`

3. 创建测试库  `create database db_test;`

4. 退出mysql、container

5. 删除mysql容器 `docker rm -f mysql01`

6. 查看volume `docker volume ls`  

   ```
   DRIVER              VOLUME NAME
   local               6f97012d2f8521c73a3cb84e0073fe66233bdb26eaae9c25d57f56388ea26d75
   local               mysql01_volume
   ```

7. 新建mysql container，指定使用mysql01_volume

    `docker run -d --name test-mysql -v mysql01_volume:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=cph123 mysql`

8. 进入容器中，登录mysql服务器，查看数据库，可以看到db_test仍然存在

   ```
   +--------------------+
   | Database           |
   +--------------------+
   | db_test            |
   | information_schema |
   | mysql              |
   | performance_schema |
   | sys                |
   +--------------------+
   ```

#### Bind mounts

1. 创建tomcat容器 

   `docker run -d --name tomcat01 -p 9090:8080 -v /tmp/test:/usr/local/tomcat/webapps/test tomcat`

2. 查看对应目录

   - centos7：/tmp/test

   - tomcat01容器：/usr/local/webapps/test

3. 在centos的/tmp/test/目录下新建a.html

   ```
   <p style="color:red; font-size=40pt;">Bind mounts test</p>
   ```

4. 进入tomcat容器对应目录下发现存在相同内容的a.html

   - centos7：curl localhost:9090/test/a.html

   - windows浏览器中： http://45.32.103.56:9090/test/a.html 



### 单机集群部署实战

#### 搭建MySQL高可用集群

集群环境

![]()

拉取镜像 docker pull percona/percona-xtradb-cluster:5.7.21

重命名镜像(方便操作) docker tag percona/percona-xtradb-cluster:5.7.21 pxc

创建单独网段供mysql集群使用 

- 创建 docker network create --subnet=172.18.0.0/24 pxc-net
- 查看网段详情 docker network inspect pxc-net
- 删除网段 docker network rm pxc-net

创建和删除volume(提前创建v1、v2、v3以备后续使用)

- 创建 docker volume create --name v1
- 查看详情 docker volume inspect v1
- 删除 docker volume inspect v1

##### 搭建pxc[mysql]集群

创建node1

```
docker run -d -p 3306:3306 -v v1:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=cph123 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=cph123 --privileged --name=node1 --net=pxc-net --ip 172.18.0.2 pxc
```

- `-e CLUSTER_NAME=PXC` 指定PXC集群名字
- `-e XTRABACKUP_PASSWORD=cph123` 指定数据库同步密码
- `--privileged` 设置优先级

创建node2

```
docker run -d -p 3302:3306 -v v2:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=cph123 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=cph123 -e CLUSTER_JOIN=node1 --privileged --name=node2 --net=pxc-net --ip 172.18.0.3 pxc
```

- `-e CLUSTER_JOIN=node1` 将该数据库加入到节点node1上组成集群

创建node3

```
docker run -d -p 3303:3306 -v v3:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=cph123 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=cph123 -e CLUSTER_JOIN=node1 --privileged --name=node3 --net=pxc-net --ip 172.18.0.4 pxc
```

##### 数据库负载均衡

拉取haproxy镜像 `docker pull haproxy`

创建haproxy配置文件(使用bind mounts方式)  `touch /tmp/haproxy/haproxy.cfg`  

```
global
    #工作目录，这边要和创建容器指定的目录对应
    chroot /usr/local/etc/haproxy
    #日志文件
    log 127.0.0.1 local5 info
    #守护进程运行
    daemon
defaults
    log global
    mode http
    #日志格式
    option httplog
    #日志中不记录负载均衡的心跳检测记录
    option dontlognull
    #连接超时（毫秒）
    timeout connect 5000
    #客户端超时（毫秒）
    timeout client 50000
    #服务器超时（毫秒）
    timeout server 50000
    #监控界面
    listen admin_stats
    #监控界面的访问的IP和端口
    bind 0.0.0.0:8888
    #访问协议
    mode http
    #URI相对地址
    stats uri /dbs_monitor
    #统计报告格式
    stats realm Global\ statistics
    #登陆帐户信息
    stats auth admin:admin
    #数据库负载均衡
    listen proxy-mysql
    #访问的IP和端口，haproxy开发的端口为3306
    #假如有人访问haproxy的3306端口，则将请求转发给下面的数据库实例
    bind 0.0.0.0:3306
    #网络协议
    mode tcp
    #负载均衡算法（轮询算法）
    #轮询算法：roundrobin
    #权重算法：static-rr
    #最少连接算法：leastconn
    #请求源IP算法：source
    balance roundrobin
    #日志格式
    option tcplog
    #在MySQL中创建一个没有权限的haproxy用户，密码为空。
    #Haproxy使用这个账户对MySQL数据库心跳检测
    option mysql-check user haproxy
    server MySQL_1 172.18.0.2:3306 check weight 1 maxconn 2000
    server MySQL_2 172.18.0.3:3306 check weight 1 maxconn 2000
    server MySQL_3 172.18.0.4:3306 check weight 1 maxconn 2000
    #使用keepalive检测死链
    option tcpka
```

创建haproxy镜像

```
docker run -it -d -p 8888:8888 -p 3307:3306 -v /tmp/haproxy:/usr/local/etc/haproxy --name haproxy01 --privileged --net=pxc-net haproxy
```

- 创建node1时占用了3306端口，此处对外提供3307端口

使用指定的配置文件启动haproxy

- `docker exec -it haproxy01 bash`
- `haproxy -f /usr/local/etc/haproxy/haproxy.cfg`

MySQL数据库中创建haproxy用户，用于心跳检测

- `CREATE USER 'haproxy'@'%' IDENTIFIED BY '';`

- 如果创建用户失败，执行以下命令

  ```
  drop user 'haproxy'@'%';
  flush privileges;
  CREATE USER 'haproxy'@'%' IDENTIFIED BY '';
  ```

windows中浏览器访问监控界面： http://45.32.103.56:8888/dbs_monitor 

！[]()

windows中通过haproxy连接数据库集群，进行数据操作可验证集群同步情况^_^



#### 搭建Nginx+Spring Boot+MySQL环境

模拟集群环境，示例如下：

![]()

##### 搭建MySQL

1. 创建volume

    `docker volume create v1_pro`

2. 创建mysql容器 

   `docker run -d --name my-mysql -v v1_pro:/var/lib/mysql -p 3301:3306 -e MYSQL_ROOT_PASWORD=cph123 --net=pro-net --ip 172.19.0.6 mysql`

3. 使用DataGrip连接数据库，执行以下语句

   ```
   create schema db_springboot collate utf8mb4_0900_ai_ci;
   use db_springboot;
   create table t_user
   (
   id int not null
   primary key,
   username varchar(50) not null,
   password varchar(50) not null,
   number varchar(100) not null
   );
   ```

##### 搭建Spring Boot项目

1. 基于Spring Boot+MyBatis实现CRUD操作，项目名称为“springboot-mybatis”

2. 项目根目录下执行 mvn clean package打出jar包并上传至docker环境中

3. 创建Dockerfile

   ```
   FROM openjdk:8-jre-alpine
   MAINTAINER cph
   LABEL name="springboot-mybatis" version="1.0" author="cph"
   COPY springboot-mybatis-0.0.1-SNAPSHOT.jar springboot-mybatis.jar
   CMD ["java","-jar","springboot-mybatis.jar"]
   ```

4. 基于Dockerfile构建镜像 `docker build -t sbm-image .`

5. 创建java应用的Container  `docker run -d --name sbc1 -p 8081:8080 --net=pro-net --ip 172.19.0.11 sbm-image`  

6. 查看启动日志 `docker logs sbc1`

7. 在windows浏览器中访问： http://45.32.103.56:8081/user/listall 

8. 继续创建多个java应用的Container

   ```
   docker run -d --name sbc2 -p 8082:8080 --net=pro-net --ip 172.19.0.12 sbm-image
   docker run -d --name sbc3 -p 8083:8080 --net=pro-net --ip 172.19.0.13 sbm-image
   ```

##### 搭建Nginx

1. 在centos的/tmp/nginx/目录下新建nginx.conf，进行相应配置

   ```
   user nginx;
   worker_processes 1;
   events {
       worker_connections 1024;
   } 
   http {
       include /etc/nginx/mime.types;
       default_type application/octet-stream;
       sendfile on;
       keepalive_timeout 65;
   	
       server {
           listen 80;
           location / {
           proxy_pass http://balance;
           }
       }
   	
   	upstream balance{
           server 172.19.0.11:8080;
           server 172.19.0.12:8080;
           server 172.19.0.13:8080;
       } 
   	include /etc/nginx/conf.d/*.conf;
   }
   ```

2. 创建nginx容器

   `docker run -d --name my-nginx -p 80:80 -v /tmp/nginx/nginx.conf:/etc/nginx/nginx.conf --network=pro-net --ip 172.19.0.10 nginx`

3. 在windows浏览器中访问  http://45.32.103.56/user/listall ，能看到返回数据表示集群搭建成功^_^