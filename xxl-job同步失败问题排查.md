# xxl-job同步失败问题排查

1. 问题背景(现象)

   - java.util.concurrent.ExecutionException: java.lang.NoClassDefFoundError: Could not initialize class org.apache.axis.client.AxisClient

2. 分析思路

   - 缺少jar包依赖
   - 依赖不正确  http://sw-academia.blogspot.com/2013/12/noclassdeffounderror-could-not.html
     - axis-jaxrpc.jar
     - axis-saaj.jar
     - axis-wsdl4j.jar
     - commons-logging.jar
     - commons-discovery.jar
   - 依赖冲突

3. 排查过程

   - 排查后确认项目中包含依赖的jar包
   - 网上搜索初步判断为jar包冲突

4. 根本原因

   - commons-discovery中包含的commons-logging 与 单独依赖的commons-logging 相冲突

5. 解决办法

   ```
   <dependency>
   
   	<groupId>commons-discovery</groupId>
   
   	<artifactId>commons-discovery</artifactId>
   
   	<version>0.5</version>
   
   	<exclusions>
   
   		<exclusion>
   
   			<groupId>commons-logging</groupId>
   
   			<artifactId>commons-logging</artifactId>
   
   		</exclusion>
   
   	</exclusions>
   
   </dependency>
   ```

6. 总结回顾

   - 查找问题的根本原因，不能留死角，否则以后需要付出更多时间和精力
   - 平时工作多翻一翻所用技术的官方文档，遇到问题才能快速、准确的解决
   
7. 其它

   - SQL Error: 1062, SQLState: 23000
   - Duplicate entry 'alexcliment' for key 'ux_user_login'