# 构建管理之Maven

Maven是Apache中一个跨平台的项目管理工具，主要用来帮助实现项目的构建、测试、打包和部署。它提供了标准的软件生命周期模型和构建模型，通过配置对项目进行管理。它将构建的过程抽象成一个个的生命周期过程，在不同阶段使用不同的已实现插件来完成相应的实际工作。

<!-- more -->

### 为什么使用Maven

手工管理项目依赖过于繁琐

ant重复造轮子

### Maven的特性

- 项目管理工具
- 约定优于配置（convention over configuaration）
  - root pom.xml
  - maven-model-builder-3.6.1.jar\org\apache\maven\model\pom-4.0.0.xml
- 简单
- 测试支持
- 构建简单
- CI
- 插件丰富

### Maven vs Ant



### 核心概念

#### setting.xml

配置文件读取顺序

1. ~/.m2/setting.xml
2. apache-maven-3.6.1/conf/setting.xml

#### pom.xml

- groupId
- artfactId 功能名
- version 版本号
- packaging 打包方式 默认是jar
- dependencyManagement
  - 只能在父pom中
  - 统一版本号
  - 声明
- dependency
  - type 默认jar
  - scope 优化pom文件，是否编译/是否打进包
    - compile 编译
    - test 测试
    - provided 编译
    - runtime 运行时
    - system 添加本地jar
    - 依赖传递
    - 依赖仲裁
      - 最短路径原则
      - 加载先后原则
    - exclusions 排除包

#### 生命周期 

- lifecycle/phase/goal
- a build lifecycle is made up of phases
- a build phase is made up of plugin goals
- 三个阶段
  - clean
    - pre-clean
    - clean
    - post-clean
  - default
    - compile
    - package
    - install
    - deploy
    - ...
  - sit
    - pre-site
    - site
    - post-site
    - site-deploy

#### 常用命令

- mvn compile    编译
- mvn clean    删除 target/
- mvn test    test case junit/testNG 
- mvn package    打包
- mvn install    把项目install到local repo
- mvn deploy    将本地jar发送到remote repo

#### 插件和仓库

- 常用插件
  -  https://maven.apache.org/plugins/index.html 
  -  https://www.mojohaus.org/plugins.html 
  -  findbugs 静态代码检查
  -  versions 统一升级版本号
  -  source 打包源代码
  -  assembly 打包zip、war



### 参考

-  https://maven.apache.org/ 

- [用Maven做项目构建](https://www.ibm.com/developerworks/cn/java/j-lo-maven/)

  