MyBatis源码学习
=================
&nbsp;&nbsp;&nbsp;&nbsp;MyBatis 是一款优秀的持久层框架，它支持定制化 SQL、存储过程以及高级映射。MyBatis 避免了几乎所有的 JDBC 代码和手动设置参数以及获取结果集。MyBatis 可以使用简单的 XML 或注解来配置和映射原生信息，将接口和 Java 的 POJOs(Plain Old Java Objects,普通的 Java对象)映射成数据库中的记录。

![mybatis架构](static/mybatis架构.jpg)

# 1. 数据库交互操作接口

&nbsp;&nbsp;&nbsp;&nbsp;数据库交互操作包括两种方式：1）使用传统MyBatis提供的API；2）使用Mapper接口操作。

# 1.1 使用传统MyBatis提供的API操作

&nbsp;&nbsp;&nbsp;&nbsp;使用SqlSession对象完成和数据库交互，请求时候根据statement和参数来执行。面向对象方式推荐使用接口直接来调用，因此实际上对SQL操作使用第二种方式，Mapper接口来实现。
~~~txt
List<?>|int|void|Map<?,?> SqlSession.select|insert|update|delete|......(statement, [parameterObject])
~~~

# 1.2 使用Mapper接口操作

&nbsp;&nbsp;&nbsp;&nbsp;MyBatis加载配置后，会将配置文件每个Mapper节点抽象为Mapper接口，接口中申请的方法跟Mapper节点中<select|update|insert|delete>节点项对应，即<select|update|insert|delete>节点的ID值为Mapper接口中的方法名称。parameterType值表示方法对应的入参，resultMap值对应Mapper接口表示返回值类型或者返回结果集的类型。
~~~java
public interface RuleMapper {

    Rule selectOne(long id);
}
~~~
~~~xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.wx.somecode.RuleMapper">
    <select id="selectOne" resultType="com.wx.somecode.Rule">
        select * from route_rule where id = #{id}
    </select>
</mapper>
~~~

&nbsp;&nbsp;&nbsp;&nbsp;根据MyBatis的配置规范配置好后，通过SqlSession.getMapper(XXXMapper.class)方法，MyBatis会根据相应的接口声明的方法信息，通过动态代理机制生成一个Mapper实例，我们使用Mapper接口的某一个方法时，MyBatis会根据这个方法的方法名和参数类型，确定StatementId，底层还是通过SqlSession.select("statementId",parameterObject);或者SqlSession.update("statementId",parameterObject);等等来实现对数据库的操作。

# 1.3 Mapper动态代理执行逻辑

&nbsp;&nbsp;&nbsp;&nbsp;Mapper接口没有具体实现，是通过动态代理实现调用，具体执行动态代理逻辑如下，根据Mapper接口调用的不同包括三个分支：1）若Mapper调用Object的原生方法，那么会走第一个分支；2）若Mapper执行的是接口的默认方法（jdk8+），那么会走第二个分支；3）执行Mapper中定义的sql接口，会走第三个分支。
~~~java
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
          // Object原生方法
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
          // 接口默认default方法
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    // Mapper接口中定义的sql方法
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }
~~~

&nbsp;&nbsp;&nbsp;&nbsp;Mapper接口定义的SQL接口会走第三个分支，执行逻辑是通过MapperMethod来完成。
~~~java
  public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
        /*
            1. 插入、修改、删除的返回值是根据Mapper接口中定义的返回值决定
            2. 查询的返回值是根据Mapper定义的返回对象决定，分为void（null）、单个对象（Map）、多个对象（List）、游标查询（大数据量查询时候内存优化）
        */
      case INSERT: {
        // 请求参数转换时候，会根据参数的个数，确定转换为null（0个）、具体值（1个）、Map（多个参数）
        Object param = method.convertArgsToSqlCommandParam(args);
        // 最终执行sql，还是委托给session完成的，session中executor专门去执行
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName() 
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
~~~