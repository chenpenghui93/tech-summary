# Kibana查询语法

- 单项term查询

  - uaa,master

- 字段field查询

  - hostname:devnode13

- 通配符(不能位于开头)

  - ？
  - *

- 范围查询

  - age:[20 TO 30] 包含端点数值
  - age:{20 TO 30} 不包含端点数值

- 逻辑操作

  - AND
  - OR

- 高级查询

  - +: 搜索结果中必须包含此项
  - +firstname:H* -age:20 city:H*   firstname字段结果中必须存在H开头的，不能有年龄是20的，city字段H开头的可有可无
    
  - -: 搜索结果中不能含有此项

    - (firstname:H* OR age:20) AND state:KS   先查询名字H开头年龄或者是20的结果，然后再与国家是KS的结合

  - 字段分组

    - firstname:(+H* -He*)   搜索firstname字段里H开头的结果，并且排除firstname里He开头的结果

  - 需要使用 \ 转义的字符

    - \+ - && || ! () {} [] ^" ~ * ? : \



## 参考

-  https://blog.csdn.net/hu948162999/article/details/51258257 
-  https://www.cnblogs.com/yiwangzhibujian/p/7137546.html 