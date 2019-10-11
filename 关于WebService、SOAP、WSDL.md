# 关于WebService、SOAP、WSDL、REST

##### 前置知识

- XML
  - Extensible Markup Language - 扩展性标记语言



##### Web Service

- 服务(Service)是计算机可以提供的一种功能
  - 本地服务：使用同一台计算机，不需要网络
  - 网络服务：使用其他计算机提供的服务，需要通过网络完成
- 网络服务的本质是通过网络调用其他网站的资源（跨平台协同）
- Web Service架构的基本思想，尽量将非核心功能交给其他人去做，自己全力开发核心功能
- 三种基本元素
  - SOAP
  - WSDL
  - UDDI
- 应用场景
  - 复用基础组件
  - 连接现有软件

###### SOAP

- SOAP（简单对象访问协议，Simple Object Access Protocol） 是一种简单的基于 XML 的协议，它使应用程序通过 HTTP 来交换信息
- SOAP=HTTP+XML

###### WSDL

- WSDL（网络服务描述语言，Web Services Description Language）是一门基于 XML 的语言，用于描述 Web Services 以及如何对它们进行访问

###### UDDI

- UDDI （通用的描述、发现以及整合，Universal Description, Discovery and Integration）是一种目录服务，通过它，企业可注册并搜索 Web services

###### REST

- REST(Representational State Transfer)一种轻量级的Web Service架构，可以完全通过HTTP协议实现



常用Web Service

- http://www.webxml.com.cn/zh_cn/support.aspx