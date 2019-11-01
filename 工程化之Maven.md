# 工程化之Maven

" Apache Maven is a software project management and comprehension tool. Based on the concept of a project object model (POM), Maven can manage a project's build, reporting and documentation from a central piece of information. "

<!-- more -->

### 为什么需要Maven

手工管理项目依赖过于繁琐

ant重复造轮子

### 什么是Maven

- 项目管理工具
- 约定优于配置（convention over configuaration）
  - root pom.xml
  - maven-model-builder-3.6.1.jar\org\apache\maven\model\pom-4.0.0.xml
- 简单
- 测试支持
- 构建简单
- CI
- 插件丰富

### setting.xml

#### 配置文件读取顺序

1. ~/.m2/setting.xml
2. apache-maven-3.6.1/conf/setting.xml

### pom.xml

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

### 生命周期 lifecycle/phase/goal

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