MyBatis源码学习
=================
&nbsp;&nbsp;&nbsp;&nbsp;MyBatis 是一款优秀的持久层框架，它支持定制化 SQL、存储过程以及高级映射。MyBatis 避免了几乎所有的 JDBC 代码和手动设置参数以及获取结果集。MyBatis 可以使用简单的 XML 或注解来配置和映射原生信息，将接口和 Java 的 POJOs(Plain Old Java Objects,普通的 Java对象)映射成数据库中的记录。

![mybatis架构](static/mybatis架构.jpg)

# 1. 数据库交互操作接口

&nbsp;&nbsp;&nbsp;&nbsp;数据库交互操作包括两种方式：1）使用传统MyBatis提供的API；2）使用Mapper接口操作。

# 1.1 使用传统MyBatis提供的API操作

&nbsp;&nbsp;&nbsp;&nbsp;使用SqlSession对象完成和数据库交互，请求时候根据Statement Id和参数来执行。
~~~txt
List<?>|int|void|Map<?,?> SqlSession.select|insert|update|delete|......(statement, [parameterObject])
~~~