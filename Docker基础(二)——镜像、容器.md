# Docker学习(二)——镜像、容器

作为docker学习笔记，以备后查。虚拟机地址为实验环境，折腾较多，不保证能随时访问。

<!-- more -->

### 镜像（Image）

#### 定义

- 一个只读的模板，模板中包含创建docekr容器的指令，每条指令都会在镜像中创建一层layer
  - 官方github地址 https://github.com/docker-library 

#### Dockerfile

- FROM 指定基础镜像。例， `FROM openjdk:8-jdk` 
- RUN 在镜像内部执行一些命令（安装软件、配置环境等）。例， `RUN mkdir -p "$CATALINA_HOME"` 
- ENV 设置变量的值，可通过docker run --e key=value修改;也可以直接使用${MYSQL_MAJOR}。例，  `ENV MYSQL_MAJOR 5.7`  
- LABEL 设置镜像标签。例， `LABEL name="crazyman"`
- VOLUME 指定数据的挂载目录。例，`VOLUME /var/lib/mysql`  
- COPY 将宿主机的文件复制到镜像内，如果目录不存在，会自动创建所需目录。仅仅复制，不会提取和解压。例，  `COPY docker-entrypoint.sh /usr/local/bin/`  
- ADD 将宿主机的文件复制到镜像内，与COPY类似，只不过会对文件提取和解压。例，`ADD application.yml /etc/itcrazy2016/`  
- WORKDIR 指定镜像的工作目录，之后的命令都是基于此目录工作，若不存在则自动创建。例，`WORKDIR /usr/local`  
- CMD 容器启动时默认执行的命令，例，`CMD ["mysqld"]` 。若有多个CMD命令，则最后一个生效。
- ENTRYPOINT 和CMD使用类似，区别在于执行`docker run`时，CMD命令会被覆盖，ENTRYPOINT命令不会被覆盖 。例，`ENTRYPOINT ["docker-entrypoint.sh"]`  
- EXPOSE 指定镜像要暴露的端口，启动镜像时，可用-p将该端口映射给宿主机。例， `EXPOSE 8080` 

#### 利用Dockerfile生成镜像

- 新建spring-boot示例项目

- 运行`mvn clean package` 打成jar包`dockerfile-demo-0.0.1-SNAPSHOT.jar`

- 在docker环境中新建目录 first-dockerfile

- 上传jar包至first-dockerfile。新建文件Dockerfile，内容如下

  ```
  FROM openjdk:8
  MAINTAINER cph
  LABEL name="dockerfile-demo" version="1.0" author="cph"
  COPY dockerfile-demo-0.0.1-SNAPSHOT.jar dockerfile-image.jar
  CMD ["java","-jar","dockerfile-image.jar"]
  ```

- 生成镜像 `docker build -t [image name] .`（最后的"**.**"表示Dockerfile在当前目录）

#### 镜像仓库（Registry）

##### 官方 Docker Hub 

- https://hub.docker.com/ 
- 在docker机器上登录 `docker login`
- 输入username、password
- 重命名镜像 `docker tag dockerfile-image chenpenghui93/dockerfile-image`
  - 镜像名称要和docker id一致
- 推送镜像 `docker push chenpenghui93/dockerfile-image`
- 拉取镜像 `docker pull chenpenghui93/dockerfile-image`
- 运行 `docker run -d --name first-docker-image -p 6666:8080 chenpenghui93/dockerfile-image`

##### 阿里云 Docker Hub 

- https://cr.console.aliyun.com/cn-hangzhou/instances/repositories 
- 登录 `sudo docker login --username=chenpenghui93@hotmail.com registry.cn-hangzhou.aliyuncs.com`
- 输入密码
- 重命名镜像 `sudo docker tag [container name/id] registry.cn-hangzhou.aliyuncs.com/crazyman/aliyun-dockerfile-image`
- 推送镜像 `sudo docker push registry.cn-hangzhou.aliyuncs.com/crazyman/aliyun-dockerfile-image`
- 拉取镜像 `docker pull registry.cn-hangzhou.aliyuncs.com/crazyman/aliyun-dockerfile-image`

##### 使用Docker Harbor搭建私库

-  https://github.com/goharbor/harbor 
- 下载版本https://github.com/goharbor/harbor/releases ，例如1.7.1
- 将安装包上传至安装了docker-compose的机器，解压`tar -zxvf  harbor-offline-installer-v1.7.1.tgz`
- 进入 harbor目录，修改harbor.cfg，将hostname修改为指定域名或本机ip
- 安装harbor `sh install.sh`
- 浏览器访问http://172.96.243.172/ ，输入用户名密码即可
- 重命名镜像 `docker tag chenpenghui93/dockerfile-image 172.96.243.172/library/harbor-image`
- 推送镜像 `docker push 172.96.243.172/library/harbor-image`
- 拉取镜像 `docker pull 172.96.243.172/library/harbor-image`



### 容器（Container）

#### 定义

- 运行着的镜像实例
- 理解：container只是基于image(只读)之后的一个layer(读写)

#### 由Container生成Image

1. 拉取centos镜像  `docker pull centos`  

2. 根据centos镜像创建container实例  `docker run -d -it --name my-centos centos`  

3. 进入my-centos容器中  `docker exec -it my-centos bash`  

4. 输入vim命令提示  bash: vim: command not found  

5. 修改container，安装vim，生成新的centos

6. 在container中安装vim   `yum install -y vim`  

7. 退出容器，生成新的centos   `docker commit my-centos vim-centos-image`  

8. 基于“vim-centos-image”生成新的容器   `docker run -d -it --name my-vim-centos vim-centos-image`  

9. 进入my-vim-centos中检查vim命令是否可用 

   ```
   docker exec -it my-vim-centos bash
   vim
   ```

   结论：docker commit命令可基于一个container重新生成一个Image。不建议这么做，排查问题时无法确认Image的生成步骤。

#### 管控Container占用资源

不对container的资源做限制时，会无限使用物理机的资源。查看资源使用情况 `docker stats`

##### 使用参数`--memory`限制内存

 `docker run -d --memory 100M --name tomcat1 tomcat`  

##### 使用参数`--cpu-shares`限制CPU

`docker run -d --cpu-shares 10 --name tomcat2 tomcat`  

##### 图形化工具监控

-   https://github.com/weaveworks/scope  

- 安装使用

  ```
  sudo curl -L git.io/scope -o /usr/local/bin/scope
  sudo chmod a+x /usr/local/bin/scope
  scope launch 172.96.243.172
  ```

-   浏览器访问  http://172.96.243.172:4040/ 

- 停止scope时执行 `scope stop`

- 同时监控两台机器，在两台机器中分别执行`scope launch ip1 ip2`  



### 底层技术

- container是一种轻量级的虚拟化技术，不用模拟硬件创建虚拟机
- Docker是基于Linux Kernel的Namespace、CGroups、UnionFileSystem等技术封装成的一种自定义容器格式，从而提供虚拟运行环境
  - Namespace：用来做隔离，比如，pid、net、mnt
  - CGroups：controller groups用来做资源限制，例如，内存、cpu
  - Union file system：用来做image和container分层