---
title: Mybatis源码解析-类型模块
author: 长歌
categories:
  - Java
  - Mybatis
tags:
  - 源码
  - Mybatis
abbrlink: 2139696773
date: 2019-08-07 17:13:00
---
① MyBatis 为简化配置文件提供了别名机制，该机制是类型转换模块的主要功能之一。
② 类型转换模块的另一个功能是实现 JDBC 类型与 Java 类型之间的转换，该功能在为 SQL 语句绑定实参以及映射查询结果集时都会涉及：

<!-- more -->

本文涉及的类图如下:
{% asset_img Type.png Type 类图 %}

# TypeHandler
`org.apache.ibatis.type.TypeHandler` ，类型转换处理器。代码如下：
```java
public interface TypeHandler<T> {

  /**
   * 设置 PreparedStatement 的指定参数
   *
   * Java Type => JDBC Type
   *
   * @param ps PreparedStatement 对象
   * @param i 参数占位符的位置
   * @param parameter 参数
   * @param jdbcType JDBC 类型
   * @throws SQLException 当发生 SQL 异常时
   */
  void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException;

  /**
   * 获得 ResultSet 的指定字段的值
   *
   * JDBC Type => Java Type
   *
   * @param rs ResultSet 对象
   * @param columnName 字段名
   * @return 值
   * @throws SQLException 当发生 SQL 异常时
   */
  T getResult(ResultSet rs, String columnName) throws SQLException;

  /**
   * 获得 ResultSet 的指定字段的值
   *
   * JDBC Type => Java Type
   *
   * @param rs ResultSet 对象
   * @param columnIndex 字段位置
   * @return 值
   * @throws SQLException 当发生 SQL 异常时
   */
  T getResult(ResultSet rs, int columnIndex) throws SQLException;

  /**
   * 获得 CallableStatement 的指定字段的值
   *
   * JDBC Type => Java Type
   *
   * @param cs CallableStatement 对象，支持调用存储过程
   * @param columnIndex 字段位置
   * @return 值
   * @throws SQLException
   */
  T getResult(CallableStatement cs, int columnIndex) throws SQLException;

}
```
## BaseTypeHandler
`org.apache.ibatis.type.BaseTypeHandler` ，实现 `TypeHandler` 接口，继承 `TypeReference` 抽象类，TypeHandler 基础抽象类。

### setParameter
`#setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType)` 方法，代码如下：
```java
  @Override
  public void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException {
    // <1> 参数为空时，设置为 null 类型
    if (parameter == null) {
      if (jdbcType == null) {
        throw new TypeException("JDBC requires that the JdbcType must be specified for all nullable parameters.");
      }
      try {
        ps.setNull(i, jdbcType.TYPE_CODE);
      } catch (SQLException e) {
        throw new TypeException("Error setting null for parameter #" + i + " with JdbcType " + jdbcType + " . "
              + "Try setting a different JdbcType for this parameter or a different jdbcTypeForNull configuration property. "
              + "Cause: " + e, e);
      }
    // 参数非空时，设置对应的参数
    } else {
      try {
        setNonNullParameter(ps, i, parameter, jdbcType);
      } catch (Exception e) {
        throw new TypeException("Error setting non null for parameter #" + i + " with JdbcType " + jdbcType + " . "
              + "Try setting a different JdbcType for this parameter or a different configuration property. "
              + "Cause: " + e, e);
      }
    }
  }
```
 - <1> 处，参数为空，设置为 null 类型。
 - <2> 处，参数非空，调用 #setNonNullParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) 抽象方法，设置对应的参数。代码如下：

### getResult
`#getResult(...) `方法，代码如下：
```java
// BaseTypeHandler.java

@Override
public T getResult(ResultSet rs, String columnName) throws SQLException {
    try {
        return getNullableResult(rs, columnName);
    } catch (Exception e) {
        throw new ResultMapException("Error attempting to get column '" + columnName + "' from result set.  Cause: " + e, e);
    }
}

@Override
public T getResult(ResultSet rs, int columnIndex) throws SQLException {
    try {
        return getNullableResult(rs, columnIndex);
    } catch (Exception e) {
        throw new ResultMapException("Error attempting to get column #" + columnIndex + " from result set.  Cause: " + e, e);
    }
}

@Override
public T getResult(CallableStatement cs, int columnIndex) throws SQLException {
    try {
        return getNullableResult(cs, columnIndex);
    } catch (Exception e) {
        throw new ResultMapException("Error attempting to get column #" + columnIndex + " from callable statement.  Cause: " + e, e);
    }
}
```
 - 调用 #getNullableResult(...) 抽象方法，获得指定结果的字段值。代码如下：
    ```java
    // BaseTypeHandler.java

    public abstract T getNullableResult(ResultSet rs, String columnName) throws SQLException;

    public abstract T getNullableResult(ResultSet rs, int columnIndex) throws SQLException;

    public abstract T getNullableResult(CallableStatement cs, int columnIndex) throws SQLException;
    ```
     - 该方法由子类实现

## 子类
TypeHandler 有非常多的子类，当然所有子类都是继承自 BaseTypeHandler 抽象类

### IntegerTypeHandler
`org.apache.ibatis.type.IntegerTypeHandler` ，继承 BaseTypeHandler 抽象类，Integer 类型的 TypeHandler 实现类。代码如下：
```java
public class IntegerTypeHandler extends BaseTypeHandler<Integer> {

  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, Integer parameter, JdbcType jdbcType)
      throws SQLException {
    // 直接设置参数即可
    ps.setInt(i, parameter);
  }

  @Override
  public Integer getNullableResult(ResultSet rs, String columnName)
      throws SQLException {
    // 获得字段的值
    int result = rs.getInt(columnName);
    // 先通过 rs 判断是否空，如果是空，则返回 null ，否则返回 result
    return result == 0 && rs.wasNull() ? null : result;
  }

  @Override
  public Integer getNullableResult(ResultSet rs, int columnIndex)
      throws SQLException {
    // 获得字段的值
    int result = rs.getInt(columnIndex);
    // 先通过 rs 判断是否空，如果是空，则返回 null ，否则返回 result
    return result == 0 && rs.wasNull() ? null : result;
  }

  @Override
  public Integer getNullableResult(CallableStatement cs, int columnIndex)
      throws SQLException {
    // 获得字段的值
    int result = cs.getInt(columnIndex);
    // 先通过 rs 判断是否空，如果是空，则返回 null ，否则返回 result
    return result == 0 && cs.wasNull() ? null : result;
  }
}
```

### DateTypeHandler
`org.apache.ibatis.type.DateTypeHandler` ，继承 BaseTypeHandler 抽象类，Date 类型的 TypeHandler 实现类。代码如下：
```java
public class DateTypeHandler extends BaseTypeHandler<Date> {

  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, Date parameter, JdbcType jdbcType)
      throws SQLException {
    // 将 Date 转换成 Timestamp 类型
    // 然后设置到 ps 中
    ps.setTimestamp(i, new Timestamp(parameter.getTime()));
  }

  @Override
  public Date getNullableResult(ResultSet rs, String columnName)
      throws SQLException {
    // 获得 Timestamp 的值
    Timestamp sqlTimestamp = rs.getTimestamp(columnName);
    // 将 Timestamp 转换成 Date 类型
    if (sqlTimestamp != null) {
      return new Date(sqlTimestamp.getTime());
    }
    return null;
  }

  @Override
  public Date getNullableResult(ResultSet rs, int columnIndex)
      throws SQLException {
    // 获得 Timestamp 的值
    Timestamp sqlTimestamp = rs.getTimestamp(columnIndex);
    // 将 Timestamp 转换成 Date 类型
    if (sqlTimestamp != null) {
      return new Date(sqlTimestamp.getTime());
    }
    return null;
  }

  @Override
  public Date getNullableResult(CallableStatement cs, int columnIndex)
      throws SQLException {
    // 获得 Timestamp 的值
    Timestamp sqlTimestamp = cs.getTimestamp(columnIndex);
    // 将 Timestamp 转换成 Date 类型
    if (sqlTimestamp != null) {
      return new Date(sqlTimestamp.getTime());
    }
    return null;
  }
}
```
 - `java.util.Date` 和 `java.sql.Timestamp` 的互相转换。

### DateOnlyTypeHandler
`org.apache.ibatis.type.DateOnlyTypeHandler` ，继承 BaseTypeHandler 抽象类，Date 类型的 TypeHandler 实现类。代码如下：
```java
public class DateOnlyTypeHandler extends BaseTypeHandler<Date> {

  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, Date parameter, JdbcType jdbcType)
      throws SQLException {
    // 将 java Date 类型抓换为 sql Date 类型
    ps.setDate(i, new java.sql.Date(parameter.getTime()));
  }

  @Override
  public Date getNullableResult(ResultSet rs, String columnName)
      throws SQLException {
    // 获得 sql Date 的值
    java.sql.Date sqlDate = rs.getDate(columnName);
    // 将 sql Date 转换成 java Date 类型
    if (sqlDate != null) {
      return new Date(sqlDate.getTime());
    }
    return null;
  }

  @Override
  public Date getNullableResult(ResultSet rs, int columnIndex)
      throws SQLException {
    // 获得 sql Date 的值
    java.sql.Date sqlDate = rs.getDate(columnIndex);
    // 将 sql Date 转换成 java Date 类型
    if (sqlDate != null) {
      return new Date(sqlDate.getTime());
    }
    return null;
  }

  @Override
  public Date getNullableResult(CallableStatement cs, int columnIndex)
      throws SQLException {
    // 获得 sql Date 的值
    java.sql.Date sqlDate = cs.getDate(columnIndex);
    // 将 sql Date 转换成 java Date 类型
    if (sqlDate != null) {
      return new Date(sqlDate.getTime());
    }
    return null;
  }

}
```
 - `java.util.Date` 和 `java.sql.Date` 的互相转换。
 - 数据库里的时间有多种类型，以 MySQL 举例子，有 date、timestamp、datetime 三种类型。

### EnumTypeHandler
`org.apache.ibatis.type.EnumTypeHandler` ，继承 `BaseTypeHandler` 抽象类，Enum 类型的 TypeHandler 实现类。代码如下：
```java
public class EnumTypeHandler<E extends Enum<E>> extends BaseTypeHandler<E> {

  /**
   * 枚举类
   */
  private final Class<E> type;

  public EnumTypeHandler(Class<E> type) {
    if (type == null) {
      throw new IllegalArgumentException("Type argument cannot be null");
    }
    this.type = type;
  }

  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, E parameter, JdbcType jdbcType) throws SQLException {
    // 将 Enum 转换成 String 类型
    if (jdbcType == null) {
      ps.setString(i, parameter.name());
    } else {
      ps.setObject(i, parameter.name(), jdbcType.TYPE_CODE); // see r3589
    }
  }

  @Override
  public E getNullableResult(ResultSet rs, String columnName) throws SQLException {
    // 获得 String 的值
    String s = rs.getString(columnName);
    // 将 String 转换成 Enum 类型
    return s == null ? null : Enum.valueOf(type, s);
  }

  @Override
  public E getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
    // 获得 String 的值
    String s = rs.getString(columnIndex);
    // 将 String 转换成 Enum 类型
    return s == null ? null : Enum.valueOf(type, s);
  }

  @Override
  public E getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
    // 获得 String 的值
    String s = cs.getString(columnIndex);
    // 将 String 转换成 Enum 类型
    return s == null ? null : Enum.valueOf(type, s);
  }
}
```

### EnumOrdinalTypeHandler
`org.apache.ibatis.type.EnumOrdinalTypeHandler` ，继承 BaseTypeHandler 抽象类，Enum 类型的 TypeHandler 实现类。代码如下：
```java
public class EnumOrdinalTypeHandler<E extends Enum<E>> extends BaseTypeHandler<E> {

  /**
   * 枚举类
   */
  private final Class<E> type;

  /**
   * {@link #type} 下所有的枚举
   *
   * @see Class#getEnumConstants()
   */
  private final E[] enums;

  public EnumOrdinalTypeHandler(Class<E> type) {
    if (type == null) {
      throw new IllegalArgumentException("Type argument cannot be null");
    }
    this.type = type;
    this.enums = type.getEnumConstants();
    if (this.enums == null) {
      throw new IllegalArgumentException(type.getSimpleName() + " does not represent an enum type.");
    }
  }

  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, E parameter, JdbcType jdbcType) throws SQLException {
    // 将 Enum 转换成 int 类型
    ps.setInt(i, parameter.ordinal());
  }

  @Override
  public E getNullableResult(ResultSet rs, String columnName) throws SQLException {
    // 获得 int 的值
    int ordinal = rs.getInt(columnName);
    // 将 int 转换成 Enum 类型
    if (ordinal == 0 && rs.wasNull()) {
      return null;
    }
    return toOrdinalEnum(ordinal);
  }

  @Override
  public E getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
    // 获得 int 的值
    int ordinal = rs.getInt(columnIndex);
    // 将 int 转换成 Enum 类型
    if (ordinal == 0 && rs.wasNull()) {
      return null;
    }
    return toOrdinalEnum(ordinal);
  }

  @Override
  public E getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
    // 获得 int 的值
    int ordinal = cs.getInt(columnIndex);
    // 将 int 转换成 Enum 类型
    if (ordinal == 0 && cs.wasNull()) {
      return null;
    }
    return toOrdinalEnum(ordinal);
  }

  private E toOrdinalEnum(int ordinal) {
    try {
      return enums[ordinal];
    } catch (Exception ex) {
      throw new IllegalArgumentException("Cannot convert " + ordinal + " to " + type.getSimpleName() + " by ordinal value.", ex);
    }
  }
}
```

### ObjectTypeHandler
`org.apache.ibatis.type.ObjectTypeHandler` ，继承 BaseTypeHandler 抽象类，Object 类型的 TypeHandler 实现类。代码如下：
```java
// ObjectTypeHandler.java

public class ObjectTypeHandler extends BaseTypeHandler<Object> {

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, Object parameter, JdbcType jdbcType)
            throws SQLException {
        ps.setObject(i, parameter);
    }

    @Override
    public Object getNullableResult(ResultSet rs, String columnName)
            throws SQLException {
        return rs.getObject(columnName);
    }

    @Override
    public Object getNullableResult(ResultSet rs, int columnIndex)
            throws SQLException {
        return rs.getObject(columnIndex);
    }

    @Override
    public Object getNullableResult(CallableStatement cs, int columnIndex)
            throws SQLException {
        return cs.getObject(columnIndex);
    }

}
```

### UnknownTypeHandler
`org.apache.ibatis.type.UnknownTypeHandler`，继承 BaseTypeHandler 抽象类，未知的 TypeHandler 实现类。通过获取对应的 TypeHandler ，进行处理。代码如下：
```java
public class UnknownTypeHandler extends BaseTypeHandler<Object> {

  /**
   * ObjectTypeHandler 单例
   */
  private static final ObjectTypeHandler OBJECT_TYPE_HANDLER = new ObjectTypeHandler();


  /**
   * TypeHandler 注册表
   */
  private TypeHandlerRegistry typeHandlerRegistry;

  public UnknownTypeHandler(TypeHandlerRegistry typeHandlerRegistry) {
    this.typeHandlerRegistry = typeHandlerRegistry;
  }

  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, Object parameter, JdbcType jdbcType)
      throws SQLException {
    // 获得参数对应的处理器
    TypeHandler handler = resolveTypeHandler(parameter, jdbcType); // <1>
    // 使用 handler 设置参数
    handler.setParameter(ps, i, parameter, jdbcType);
  }

  @Override
  public Object getNullableResult(ResultSet rs, String columnName)
      throws SQLException {
    // 获得参数对应的处理器
    TypeHandler<?> handler = resolveTypeHandler(rs, columnName);// <2>
    // 使用 handler 获得值
    return handler.getResult(rs, columnName);
  }

  @Override
  public Object getNullableResult(ResultSet rs, int columnIndex)
      throws SQLException {
    // 获得参数对应的处理器
    TypeHandler<?> handler = resolveTypeHandler(rs.getMetaData(), columnIndex); // <3>
    // 如果找不到对应的处理器，使用 OBJECT_TYPE_HANDLER
    if (handler == null || handler instanceof UnknownTypeHandler) {
      handler = OBJECT_TYPE_HANDLER;
    }
    // 使用 handler 获得值
    return handler.getResult(rs, columnIndex);
  }

  @Override
  public Object getNullableResult(CallableStatement cs, int columnIndex)
      throws SQLException {
    return cs.getObject(columnIndex);
  }

  private TypeHandler<?> resolveTypeHandler(Object parameter, JdbcType jdbcType) {
    TypeHandler<?> handler;
    // 参数为空，返回 OBJECT_TYPE_HANDLER
    if (parameter == null) {
      handler = OBJECT_TYPE_HANDLER;
    } else {
      // 参数非空，使用参数类型获得对应的 TypeHandler
      handler = typeHandlerRegistry.getTypeHandler(parameter.getClass(), jdbcType);
      // check if handler is null (issue #270)
      // 获取不到，则使用 OBJECT_TYPE_HANDLER
      if (handler == null || handler instanceof UnknownTypeHandler) {
        handler = OBJECT_TYPE_HANDLER;
      }
    }
    return handler;
  }

  private TypeHandler<?> resolveTypeHandler(ResultSet rs, String column) {
    try {
      // 获得 columnIndex
      Map<String,Integer> columnIndexLookup;
      columnIndexLookup = new HashMap<>();// 通过 metaData
      ResultSetMetaData rsmd = rs.getMetaData();
      int count = rsmd.getColumnCount();
      for (int i = 1; i <= count; i++) {
        String name = rsmd.getColumnName(i);
        columnIndexLookup.put(name,i);
      }
      Integer columnIndex = columnIndexLookup.get(column);
      TypeHandler<?> handler = null;
      // 首先，通过 columnIndex 获得 TypeHandler
      if (columnIndex != null) {
        handler = resolveTypeHandler(rsmd, columnIndex);// <3>
      }
      // 获得不到，使用 OBJECT_TYPE_HANDLER
      if (handler == null || handler instanceof UnknownTypeHandler) {
        handler = OBJECT_TYPE_HANDLER;
      }
      return handler;
    } catch (SQLException e) {
      throw new TypeException("Error determining JDBC type for column " + column + ".  Cause: " + e, e);
    }
  }

  private TypeHandler<?> resolveTypeHandler(ResultSetMetaData rsmd, Integer columnIndex) {
    TypeHandler<?> handler = null;
    // 获得 JDBC Type 类型
    JdbcType jdbcType = safeGetJdbcTypeForColumn(rsmd, columnIndex);
    // 获得 Java Type 类型
    Class<?> javaType = safeGetClassForColumn(rsmd, columnIndex);
    //获得对应的 TypeHandler 对象
    if (javaType != null && jdbcType != null) {
      handler = typeHandlerRegistry.getTypeHandler(javaType, jdbcType);
    } else if (javaType != null) {
      handler = typeHandlerRegistry.getTypeHandler(javaType);
    } else if (jdbcType != null) {
      handler = typeHandlerRegistry.getTypeHandler(jdbcType);
    }
    return handler;
  }

  private JdbcType safeGetJdbcTypeForColumn(ResultSetMetaData rsmd, Integer columnIndex) {
    try {
      // 从 ResultSetMetaData 中，获得字段类型
      // 获得 JDBC Type
      return JdbcType.forCode(rsmd.getColumnType(columnIndex));
    } catch (Exception e) {
      return null;
    }
  }

  private Class<?> safeGetClassForColumn(ResultSetMetaData rsmd, Integer columnIndex) {
    try {
      // 从 ResultSetMetaData 中，获得字段类型
      // 获得 Java Type
      return Resources.classForName(rsmd.getColumnClassName(columnIndex));
    } catch (Exception e) {
      return null;
```


# TypeReference
`org.apache.ibatis.type.TypeReference` ，引用泛型抽象类。目的很简单，就是解析类上定义的泛型。代码如下：
```java
public abstract class TypeReference<T> {

  /*
  * 泛型
  */
  private final Type rawType;

  protected TypeReference() {
    rawType = getSuperclassTypeParameter(getClass());
  }

  Type getSuperclassTypeParameter(Class<?> clazz) {
    // 【1】从父类中获取 <T>
    Type genericSuperclass = clazz.getGenericSuperclass();
    if (genericSuperclass instanceof Class) {
      // try to climb up the hierarchy until meet something useful
      // 能满足这个条件的，例如 GenericTypeSupportedInHierarchiesTestCase.CustomStringTypeHandler 这个类
      if (TypeReference.class != genericSuperclass) { // 排除 TypeReference 类
        return getSuperclassTypeParameter(clazz.getSuperclass());
      }

      throw new TypeException("'" + getClass() + "' extends TypeReference but misses the type parameter. "
        + "Remove the extension or add a type parameter to it.");
    }

    // 【2】获取 <T>
    Type rawType = ((ParameterizedType) genericSuperclass).getActualTypeArguments()[0];
    // TODO remove this when Reflector is fixed to return Types
    // 必须是泛型，才获取 <T>
    if (rawType instanceof ParameterizedType) {
      rawType = ((ParameterizedType) rawType).getRawType();
    }

    return rawType;
  }

  public final Type getRawType() {
    return rawType;
  }

  @Override
  public String toString() {
    return rawType.toString();
  }
}
```

# 注解
`type` 包中，也定义了三个注解，我们逐个来看看。

## @MappedTypes
`org.apache.ibatis.type.@MappedTypes` ，匹配的 Java Type 类型的注解。代码如下：
```java
/**
 * @author Eduardo Macarron
 */
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE) // 注册到类
public @interface MappedTypes {
  /**
   * @return 匹配的 Java Type 类型的数组
   */
  Class<?>[] value();
}
```

## @MappedJdbcTypes
`org.apache.ibatis.type.@MappedJdbcTypes` ，匹配的 JDBC Type 类型的注解。代码如下：
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface MappedJdbcTypes {
  
  /**
   * @return 匹配的 JDBC Type 类型的注解
   */
  JdbcType[] value();

  /**
   * @return 是否包含 {@link java.sql.JDBCType#NULL}
   * @throws false [description]
   */
  boolean includeNullJdbcType() default false;
}
```

## @Alias
`org.apache.ibatis.type.@Alias` ，别名的注解。代码如下：
```java
// Alias.java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Alias {

    /**
     * @return 别名
     */
    String value();
}
```

# JdbcType
`org.apache.ibatis.type.JdbcType`，Jdbc Type 枚举。代码如下：
```java
public enum JdbcType {
  /*
   * This is added to enable basic support for the
   * ARRAY data type - but a custom type handler is still required
   */
  ARRAY(Types.ARRAY),
  BIT(Types.BIT),
  TINYINT(Types.TINYINT),
  SMALLINT(Types.SMALLINT),
  INTEGER(Types.INTEGER),
  BIGINT(Types.BIGINT),
  FLOAT(Types.FLOAT),
  REAL(Types.REAL),
  DOUBLE(Types.DOUBLE),
  NUMERIC(Types.NUMERIC),
  DECIMAL(Types.DECIMAL),
  CHAR(Types.CHAR),
  VARCHAR(Types.VARCHAR),
  LONGVARCHAR(Types.LONGVARCHAR),
  DATE(Types.DATE),
  TIME(Types.TIME),
  TIMESTAMP(Types.TIMESTAMP),
  BINARY(Types.BINARY),
  VARBINARY(Types.VARBINARY),
  LONGVARBINARY(Types.LONGVARBINARY),
  NULL(Types.NULL),
  OTHER(Types.OTHER),
  BLOB(Types.BLOB),
  CLOB(Types.CLOB),
  BOOLEAN(Types.BOOLEAN),
  CURSOR(-10), // Oracle
  UNDEFINED(Integer.MIN_VALUE + 1000),
  NVARCHAR(Types.NVARCHAR), // JDK6
  NCHAR(Types.NCHAR), // JDK6
  NCLOB(Types.NCLOB), // JDK6
  STRUCT(Types.STRUCT),
  JAVA_OBJECT(Types.JAVA_OBJECT),
  DISTINCT(Types.DISTINCT),
  REF(Types.REF),
  DATALINK(Types.DATALINK),
  ROWID(Types.ROWID), // JDK6
  LONGNVARCHAR(Types.LONGNVARCHAR), // JDK6
  SQLXML(Types.SQLXML), // JDK6
  DATETIMEOFFSET(-155), // SQL Server 2008
  TIME_WITH_TIMEZONE(Types.TIME_WITH_TIMEZONE), // JDBC 4.2 JDK8
  TIMESTAMP_WITH_TIMEZONE(Types.TIMESTAMP_WITH_TIMEZONE); // JDBC 4.2 JDK8

  /**
   * 类型编号
   */
  public final int TYPE_CODE;
  /**
   * 代码编号和 {@link JdbcType} 的映射
   */
  private static Map<Integer,JdbcType> codeLookup = new HashMap<>();

  static {
    // 初始化 codeLookup
    for (JdbcType type : JdbcType.values()) {
      codeLookup.put(type.TYPE_CODE, type);
    }
  }

  JdbcType(int code) {
    this.TYPE_CODE = code;
  }

  public static JdbcType forCode(int code)  {
    return codeLookup.get(code);
  }

}
```

# TypeHandlerRegistry
`org.apache.ibatis.type.TypeHandlerRegistry` ，TypeHandler 注册表，相当于管理 TypeHandler 的容器，从其中能获取到对应的 TypeHandler 。

## 构造方法