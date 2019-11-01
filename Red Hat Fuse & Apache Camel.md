存在的问题？为什么使用？

- 企业中应用、服务、流程、设备等的各种跨系统集成痛点
- 普元ESB不支持接口服务上云



如何解决？

- 跨系统的接口



前置知识：

- Enterprise Integration Patterns



Apache Camel

-  https://camel.apache.org/ 
- [what exactly is apache camel？](https://stackoverflow.com/questions/8845186/what-exactly-is-apache-camel )
- [Open Source Integration With Apache Camel and How Fuse IDE Can Help](https://dzone.com/articles/open-source-integration-apache)

常见使用场景

- 消息汇聚，例如，将 ActiveMQ，RabbitMQ，WebService等的消息存储至日志文件中
- 消息分发
  - 顺序分发
  - 并行分发
- 消息转换，例如，将xml数据转换为json数据
- 规则引擎

代码

- 关键类
  - Endpoint
  - Processer
    - **使用时注意线程安全问题**
  - Exchange
  - Message





其它

- 《 *Enterprise Integration Patterns: Designing, Building, and Deploying Messaging Solutions* 》
- 《*Camel In Action*》
- *development guide https://access.redhat.com/documentation/en-us/red_hat_fuse/7.4/html/apache_camel_development_guide/index*
- *tooling tutorials https://access.redhat.com/documentation/en-us/red_hat_fuse/7.4/html/tooling_tutorials/index*