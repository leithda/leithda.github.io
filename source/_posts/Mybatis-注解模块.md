---
title: Mybatis-注解模块
abbrlink: 2645650477
date: 2020-04-20 19:24:03
categories:
  - Java
  - Mybatis
tags:
  - 源码
  - Mybatis
author: 长歌
---


{% cq %}
随着 Java 注解的慢慢流行，MyBatis 提供了注解的方式，使得我们方便的在 Mapper 接口上编写简单的数据库 SQL 操作代码，而无需像之前一样，必须编写 SQL 在 XML 格式的 Mapper 文件中。
{% endcq %}

<!-- More -->

# CRUD 常用操作注解

```java
import java.util.List;
import org.apache.ibatis.annotations.Delete;
import org.apache.ibatis.annotations.Insert;
import org.apache.ibatis.annotations.Param;
import org.apache.ibatis.annotations.Select;
import org.apache.ibatis.annotations.Update;
import com.whut.model.User;

// 最基本的注解CRUD
public interface IUserDAO {

    @Select("select *from User")
    public List<User> retrieveAllUsers();
                                                                                                                                                                                                                                  
    //注意这里只有一个参数，则#{}中的标识符可以任意取
    @Select("select *from User where id=#{idss}")
    public User retrieveUserById(int id);
                                                                                                                                                                                                                                  
    @Select("select *from User where id=#{id} and userName like #{name}")
    public User retrieveUserByIdAndName(@Param("id")int id,@Param("name")String names);
                                                                                                                                                                                                                                  
    @Insert("INSERT INTO user(userName,userAge,userAddress) VALUES(#{userName},"
            + "#{userAge},#{userAddress})")
    public void addNewUser(User user);
                                                                                                                                                                                                                                  
    @Delete("delete from user where id=#{id}")
    public void deleteUser(int id);
                                                                                                                                                                                                                                  
    @Update("update user set userName=#{userName},userAddress=#{userAddress}"
            + " where id=#{id}")
    public void updateUser(User user);
    
}
```

## @Select
> 查询语句注解，代码如下：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD) // 方法
public @interface Select {
	
  	// 查询语句
    String[] value();

}
```

## @Insert

> 插入语句注解，代码如下：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD) // 方法
public @interface Insert {

    // 插入语句
    String[] value();

}
```



## @Update

> 更新语句注解，代码如下：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Update {

    // @return 更新语句
    String[] value();

}
```



## @Delete

> 删除语句注解，代码如下：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD) // 方法
public @interface Delete {

    // @return 删除语句
    String[] value();

}
```



## @Param

> 方法参数名的注解
>
> 默认通过索引使用，例如: `#{1}`、`#{2}`。当使用`@Param("name")`时，`SQL`中参数应该被命名为`#{name}`
>
> 代码如下：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface Param {

  // 参数名
  String value();
}
```





# CRUD高级操作注解

- 使用实例如下：
  + IBlogDAO 接口：
```java
import java.util.List;
import org.apache.ibatis.annotations.CacheNamespace;
import org.apache.ibatis.annotations.DeleteProvider;
import org.apache.ibatis.annotations.InsertProvider;
import org.apache.ibatis.annotations.Options;
import org.apache.ibatis.annotations.Param;
import org.apache.ibatis.annotations.Result;
import org.apache.ibatis.annotations.ResultMap;
import org.apache.ibatis.annotations.Results;
import org.apache.ibatis.annotations.SelectProvider;
import org.apache.ibatis.annotations.UpdateProvider;
import org.apache.ibatis.type.JdbcType;
import com.whut.model.Blog;
import com.whut.sqlTool.BlogSqlProvider;

@CacheNamespace(size=100)
public interface IBlogDAO {

    @SelectProvider(type = BlogSqlProvider.class, method = "getSql") 
    @Results(value ={ 
            @Result(id=true, property="id",column="id",javaType=Integer.class,jdbcType=JdbcType.INTEGER),
            @Result(property="title",column="title",javaType=String.class,jdbcType=JdbcType.VARCHAR),
            @Result(property="date",column="date",javaType=String.class,jdbcType=JdbcType.VARCHAR),
            @Result(property="authername",column="authername",javaType=String.class,jdbcType=JdbcType.VARCHAR),
            @Result(property="content",column="content",javaType=String.class,jdbcType=JdbcType.VARCHAR),
            }) 
    public Blog getBlog(@Param("id") int id);
                                                                                                                                                                                      
    @SelectProvider(type = BlogSqlProvider.class, method = "getAllSql") 
    @Results(value ={ 
            @Result(id=true, property="id",column="id",javaType=Integer.class,jdbcType=JdbcType.INTEGER),
            @Result(property="title",column="title",javaType=String.class,jdbcType=JdbcType.VARCHAR),
            @Result(property="date",column="date",javaType=String.class,jdbcType=JdbcType.VARCHAR),
            @Result(property="authername",column="authername",javaType=String.class,jdbcType=JdbcType.VARCHAR),
            @Result(property="content",column="content",javaType=String.class,jdbcType=JdbcType.VARCHAR),
            }) 
    public List<Blog> getAllBlog();
                                                                                                                                                                                      
    @SelectProvider(type = BlogSqlProvider.class, method = "getSqlByTitle") 
    @ResultMap(value = "sqlBlogsMap") 
    // 这里调用resultMap，这个是SQL配置文件中的,必须该SQL配置文件与本接口有相同的全限定名
    // 注意文件中的namespace路径必须是使用@resultMap的类路径
    public List<Blog> getBlogByTitle(@Param("title")String title);
                                                                                                                                                                                      
    @InsertProvider(type = BlogSqlProvider.class, method = "insertSql") 
    public void insertBlog(Blog blog);
                                                                                                                                                                                      
    @UpdateProvider(type = BlogSqlProvider.class, method = "updateSql")
    public void updateBlog(Blog blog);
                                                                                                                                                                                      
    @DeleteProvider(type = BlogSqlProvider.class, method = "deleteSql")
    @Options(useCache = true, flushCache = false, timeout = 10000) 
    public void deleteBlog(int ids);
                                                                                                                                                                                      
}
```
  + BlogSqlProvider 类：
```java
import java.util.Map;
import static org.apache.ibatis.jdbc.SqlBuilder.*;
package com.whut.sqlTool;
import java.util.Map;
import static org.apache.ibatis.jdbc.SqlBuilder.*;

public class BlogSqlProvider {

    private final static String TABLE_NAME = "blog";
    
    public String getSql(Map<Integer, Object> parameter) {
        BEGIN();
        //SELECT("id,title,authername,date,content");
        SELECT("*");
        FROM(TABLE_NAME);
        //注意这里这种传递参数方式，#{}与map中的key对应，而map中的key又是注解param设置的
        WHERE("id = #{id}");
        return SQL();
    }
    
    public String getAllSql() {
        BEGIN();
        SELECT("*");
        FROM(TABLE_NAME);
        return SQL();
    }
    
    public String getSqlByTitle(Map<String, Object> parameter) {
        String title = (String) parameter.get("title");
        BEGIN();
        SELECT("*");
        FROM(TABLE_NAME);
        if (title != null)
            WHERE(" title like #{title}");
        return SQL();
    }
    
    public String insertSql() {
        BEGIN();
        INSERT_INTO(TABLE_NAME);
        VALUES("title", "#{title}");
        //  VALUES("title", "#{tt.title}");
        //这里是传递一个Blog对象的，如果是利用上面tt.方式，则必须利用Param来设置别名
        VALUES("date", "#{date}");
        VALUES("authername", "#{authername}");
        VALUES("content", "#{content}");
        return SQL();
    }
    
    public String deleteSql() {
        BEGIN();
        DELETE_FROM(TABLE_NAME);
        WHERE("id = #{id}");
        return SQL();
    }
    
    public String updateSql() {
        BEGIN();
        UPDATE(TABLE_NAME);
        SET("content = #{content}");
        WHERE("id = #{id}");
        return SQL();
    }
}
```
    - 该示例使用 org.apache.ibatis.jdbc.SqlBuilder 来实现 SQL 的拼接与生成。实际上，目前该类已经废弃，推荐使用个的是 org.apache.ibatis.jdbc.SQL 类。
    - 具体的 SQL 使用示例，可参见 org.apache.ibatis.jdbc.SQLTest 单元测试类。

    + Mappper.xml
```java
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
"http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.whut.inter.IBlogDAO">
   <resultMap type="Blog" id="sqlBlogsMap">
      <id property="id" column="id"/>
      <result property="title" column="title"/>
      <result property="authername" column="authername"/>
      <result property="date" column="date"/>
      <result property="content" column="content"/>
   </resultMap> 
</mapper>
```

## @SelectProvider
`org.apache.ibatis.annotations.@SelectProvider` ，查询语句提供器。代码如下：
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD) // 方法
public @interface SelectProvider {

    /**
     * @return 提供的类
     */
    Class<?> type();

    /**
     * @return 提供的方法
     */
    String method();

}
```
- 从上面的使用示例可知，XXXProvider 的用途是，指定一个类( type )的指定方法( method )，返回使用的 SQL 。并且，该方法可以使用 Map<String,Object> params 来作为方法参数，传递参数。

## 其余提供器
> 同`@SelectProvider` 类似，不再赘述

- `@InsertProvider`
- `@UpdateProvider`
- `@DeleteProvider`

## @Results
> 对应xml标签为`<resultMap />`

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Results {
  /**
   * The name of the result map.
   */
  String id() default "";

  /**
   * {@link Result}数组
   * @return Result 数组
   */
  Result[] value() default {};
}
```

## @Result
`org.apache.ibatis.annotations.@Results` ，结果字段的注解。代码如下：
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({})
public @interface Result {
  /**
   * @return 是否是ID字段
   */
  boolean id() default false;

  /**
   * @return 数据库中的字段
   */
  String column() default "";

  /**
   * @return Java 类中的属性
   */
  String property() default "";

  /**
   * @return Java Type
   */
  Class<?> javaType() default void.class;

  /**
   * @return JDBC Type
   */
  JdbcType jdbcType() default JdbcType.UNDEFINED;

  /**
   * @return 使用的 TypeHandler 处理器
   */
  Class<? extends TypeHandler> typeHandler() default UnknownTypeHandler.class;

  /**
   * @return   {@link One} 注解
   */
  One one() default @One;

  /**
   * @return {@link Many} 注解
   */
  Many many() default @Many;
}
```


## @One
`org.apache.ibatis.annotations.@One` ，复杂类型的单独属性值的注解。代码如下：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({})
public @interface One {
  /**
   * @return 已映射语句（也就是映射器方法）的全限定名
   */
  String select() default "";

  /**
   * @return 加载类型
   */
  FetchType fetchType() default FetchType.DEFAULT;

}
```

## @Many
`org.apache.ibatis.annotations.@Many` ，复杂类型的集合属性值的注解。代码如下:

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({})
public @interface Many {

  /**
   * @return 已映射语句（也就是映射器方法）的全限定名
   */
  String select() default "";

  /**
   * @return 加载类型
   */
  FetchType fetchType() default FetchType.DEFAULT;

}
```


## @ResultMap
`org.apache.ibatis.annotations.@ResultMap` ，使用的结果集的注解。代码如下：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ResultMap {

  /**
   * @return 结果集
   */
  String[] value();
}
```
- 例如上述示例的 `#getBlogByTitle(@Param("title")String title) `方法，使用的注解为` @ResultMap(value = "sqlBlogsMap")`，
而 `"sqlBlogsMap"`` 中 `Mapper XML` 中有相关的定义。


## @ResultType
`org.apache.ibatis.annotations.@ResultType` ，结果类型。代码如下：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ResultType {

  /**
   * @return 类型
   */
  Class<?> value();
}
```

## @CacheNamespace
> 对应 XML 标签为 `<cache />`

`org.apache.ibatis.annotations.@CacheNamespace`，缓存空间配置的注解。代码如下：
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface CacheNamespace {

  /**
   * @return 对应 XML 标签为 <cache />
   */
  Class<? extends org.apache.ibatis.cache.Cache> implementation() default PerpetualCache.class;

  /**
   * @return 负责过期的 Cache 实现类
   */
  Class<? extends org.apache.ibatis.cache.Cache> eviction() default LruCache.class;

  /**
   * @return 清空缓存的频率。0 代表不清空
   */
  long flushInterval() default 0;

  /**
   * @return 缓存容器大小
   */
  int size() default 1024;

  /**
   * @return 是否序列化。{@link org.apache.ibatis.cache.decorators.SerializedCache}
   */
  boolean readWrite() default true;

  /**
   * @return 是否阻塞。{@link org.apache.ibatis.cache.decorators.BlockingCache}
   */
  boolean blocking() default false;

  /**
   * Property values for a implementation object.
   * @since 3.4.2
   *
   * {@link Property} 数组
   */
  Property[] properties() default {};

}
```


## @Property
`org.apache.ibatis.annotations.@Property` ，属性的注解。代码如下：
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({})
public @interface Property {

  /**
   * A target property name.
   * 属性名
   */
  String name();

  /**
   * A property value or placeholder.
   * 属性值
   */
  String value();
}
```


## @CacheNamespaceRef
> 对应 XML 标签为 `<cache-ref />`

`org.apache.ibatis.annotations.@CacheNamespaceRef` ，指向指定命名空间的注解。代码如下：
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface CacheNamespaceRef {

  /**
   * 见 {@link MapperAnnotationBuilder#parseCacheRef() } 方法
   * A namespace type to reference a cache (the namespace name become a FQCN of specified type).
   */
  Class<?> value() default void.class;

  /**
   * 指向的命名空间
   * A namespace name to reference a cache.
   * @since 3.4.2
   */
  String name() default "";
}
```

## Options
`org.apache.ibatis.annotations.@Options` ，操作可选项。代码如下：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Options {
  /**
   * The options for the {@link Options#flushCache()}.
   * The default is {@link FlushCachePolicy#DEFAULT}
   */
  enum FlushCachePolicy {
    /** <code>false</code> for select statement; <code>true</code> for insert/update/delete statement. */
    DEFAULT,
    /** Flushes cache regardless of the statement type. */
    TRUE,
    /** Does not flush cache regardless of the statement type. */
    FALSE
  }

  /**
   * @return 是否使用缓存
   */
  boolean useCache() default true;

  /**
   * @return 刷新缓存的策略
   */
  FlushCachePolicy flushCache() default FlushCachePolicy.DEFAULT;

  /**
   * @return 结果类型
   */
  ResultSetType resultSetType() default ResultSetType.DEFAULT;

  /**
   * @return 语句类型
   */
  StatementType statementType() default StatementType.PREPARED;

  /**
   * @return 加载数量
   */
  int fetchSize() default -1;

  /**
   * @return 超时时间
   */
  int timeout() default -1;

  /**
   * @return 是否生成主键
   */
  boolean useGeneratedKeys() default false;

  /**
   * @return 主键在Java类中的属性
   */
  String keyProperty() default "";

  /**
   * @return 键在数据库中的字段
   */
  String keyColumn() default "";

  /**
   * @return 结果集
   */
  String resultSets() default "";
}
```


## @SelectKey
`org.apache.ibatis.annotations.@SelectKey` ，通过 SQL 语句获得主键的注解。代码如下：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface SelectKey {
  /**
   * @return 语句
   */
  String[] statement();

  /**
   * @return Java 对象的属性
   */
  String keyProperty();

  /**
   * @return 数据库中的字段
   */
  String keyColumn() default "";

  /**
   * @return 在插入语句执行前，还是执行后
   */
  boolean before();

  /**
   * @return 返回类型
   */
  Class<?> resultType();

  /**
   * @return {@link #statement()} 的类型
   */
  StatementType statementType() default StatementType.PREPARED;
}
```

## MapKey
`org.apache.ibatis.annotations.@MapKey` ，Map 结果的键的注解。代码如下：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface MapKey {
  /**
   * @return 键名
   */
  String value();
}
```

## @Flush
> 如果使用了这个注解，定义在 Mapper 接口中的方法能够调用 SqlSession#flushStatements() 方法。（Mybatis 3.3及以上）

`org.apache.ibatis.annotations.@Flush` ，Flush 注解。代码如下：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Flush {
}
```

# 其他注解
## @Mapper
`org.apache.ibatis.annotations.Mapper` ，标记这是个 Mapper 的注解。代码如下：

```java
@Documented
@Inherited
@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.TYPE, ElementType.METHOD, ElementType.FIELD, ElementType.PARAMETER })
public @interface Mapper {
  // Interface Mapper
}

```

## @Lang
`org.apache.ibatis.annotations.@Lang` ，语言驱动的注解。代码如下：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Lang {

  /**
   * @return 驱动类
   */
  Class<? extends LanguageDriver> value();
}
```

## 其他
- `TypeDiscriminator + @Case`
- `@ConstructorArgs + @Arg`
- `@AutomapConstructor`






