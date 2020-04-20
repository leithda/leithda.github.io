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













