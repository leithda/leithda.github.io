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
```java
/**
   * JDBC Type 和 {@link TypeHandler} 的映射
   */
  private final Map<JdbcType, TypeHandler<?>>  jdbcTypeHandlerMap = new EnumMap<>(JdbcType.class);

  /**
   * {@link TypeHandler}
   */
  private final Map<Type, Map<JdbcType, TypeHandler<?>>> typeHandlerMap = new ConcurrentHashMap<>();

  /**
   * {@link UnknownTypeHandler}
   */
  private final TypeHandler<Object> unknownTypeHandler = new UnknownTypeHandler(this);

  /**
   * 所有 {@link TypeHandler} 的集合
   */
  private final Map<Class<?>, TypeHandler<?>> allTypeHandlersMap = new HashMap<>();

  /**
   * 空的 {@link TypeHandler} 集合
   */
  private static final Map<JdbcType, TypeHandler<?>> NULL_TYPE_HANDLER_MAP = Collections.emptyMap();

  /**
   * 默认的枚举类型的 TypeHandler 对象
   */
  private Class<? extends TypeHandler> defaultEnumTypeHandler = EnumTypeHandler.class;

  public TypeHandlerRegistry() {

    // <1>
    register(Date.class, new DateTypeHandler());
    register(Date.class, JdbcType.DATE, new DateOnlyTypeHandler());
    register(Date.class, JdbcType.TIME, new TimeOnlyTypeHandler());
    // <2>
    register(JdbcType.TIMESTAMP, new DateTypeHandler());
    register(JdbcType.DATE, new DateOnlyTypeHandler());
    register(JdbcType.TIME, new TimeOnlyTypeHandler());
    // 省略其他类型的注册
  }

```
- `TYPE_HANDLER_MAP`属性，TypeHandler的映射
  - 一个Java Type 可以对应多个 JDBC Type,也就是多个TypeHandler,所以Map的第一层的值是`Map<JdbcTyoe,TypeHandler<?>`。在`<1>`处，我们可以看到，Date对应了对个JDBC的TypeHandler的注册
  - 当一个`Java Type`不存在对应的JDBC Type时，就使用`NULL_TYPE_HANDLER_MAP`静态属性，添加到`TYPE_HANDLER_MAP`中进行占位。
- `JDBC_TYPE_HANDLER_MAP`属性，JDBC Type个TypeHandler的映射。
  - 一个JDBC Type只对应一个Java Type，也就是一个TypeHandler,不同于`TYPE_HANDLER_MAP`属性，在`<2>`
处，我们可以看到，三个时间类型的Jdbc Type注册到`JDBC_TYPE_HANDLER_MAP`中。
  - `JDBC_TYPE_HANDLER_MAP`是一一映射，简单就可以获得 JDBC Type 对应的 TypeHandler ，而 `TYPE_HANDLER_MAP`是一对多映射，一个Java Type该如何获取对应的TypeHandler呢，答案在`#getTypeHandler(Type type,JdbcType jdbcType)`方法。

## getInstance
`#getInstance(Class<?> javaTypeClass, Class<?> typeHandlerClass)`方法，创建TypeHandler对象。代码如下：

```java
  @SuppressWarnings("unchecked")
  public <T> TypeHandler<T> getInstance(Class<?> javaTypeClass, Class<?> typeHandlerClass) {
    // 获得 Class 类型的构造方法
    if (javaTypeClass != null) {
      try {
        Constructor<?> c = typeHandlerClass.getConstructor(Class.class);
        return (TypeHandler<T>) c.newInstance(javaTypeClass); // 符合这个条件的，例如：EnumTypeHandler
      } catch (NoSuchMethodException ignored) {
        // ignored  // ignored 忽略该异常，继续向下
      } catch (Exception e) {
        throw new TypeException("Failed invoking constructor for handler " + typeHandlerClass, e);
      }
    }
    // 获得空参的构造方法
    try {
      Constructor<?> c = typeHandlerClass.getConstructor();
      return (TypeHandler<T>) c.newInstance();
    } catch (Exception e) {
      throw new TypeException("Unable to find a usable constructor for " + typeHandlerClass, e);
    }
  }
```

## register
`#register(...)`方法，注册TypeHandler。TypeHandlerRegistry中有大量该方法的重载实现，整理如下：

> FROM 徐群明 [<Mybatis技术内幕>](https://item.jd.com/12125531.html)
> {% asset_img register.png register 方法 %}

除了⑤以外，所有方法最终都会调用④，即`#register(Type javaType, JdbcType jdbcType, TypeHandler<?> handler)`方法，代码如下：
```java
  private void register(Type javaType, JdbcType jdbcType, TypeHandler<?> handler) {
    // <1> 添加 handler 到 TYPE_HANDLER_MAP 中
    if (javaType != null) {
      // 获取 Java Type 对应的 map
      Map<JdbcType, TypeHandler<?>> map = typeHandlerMap.get(javaType);
      if (map == null || map == NULL_TYPE_HANDLER_MAP) {
        map = new HashMap<>();
        typeHandlerMap.put(javaType, map);
      }
      // 添加到 handler 的 map 中
      map.put(jdbcType, handler);
    }
    // <2> 添加 handler 到 ALL_TYPE_HANDLER__MAP 中
    allTypeHandlersMap.put(handler.getClass(), handler);
  }
```

1. `#register(String packageName)` 方法，扫描指定包下的所有TypeHandler类，并发起注册。代码如下：
```java
  public void register(String packageName) {
    // 扫描指定包下的所有 TypeHandler 类
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<>();
    resolverUtil.find(new ResolverUtil.IsA(TypeHandler.class), packageName);
    Set<Class<? extends Class<?>>> handlerSet = resolverUtil.getClasses();
    // 遍历 TypeHandler 数组，发起注册
    for (Class<?> type : handlerSet) {
      //Ignore inner classes and interfaces (including package-info.java) and abstract classes
      // 排除匿名类、接口、抽象类
      if (!type.isAnonymousClass() && !type.isInterface() && !Modifier.isAbstract(type.getModifiers())) {
        register(type);
      }
    }
  }
```
- 上述方法中，会调用⑥ `#register(Class<?> typeHandlerClass)`方法，注册指定的 TypeHandler 类。代码如下：
  ```java
  public void register(Class<?> typeHandlerClass) {
    boolean mappedTypeFound = false;
    // <3> 获得 @MappedTypes 注解
    MappedTypes mappedTypes = typeHandlerClass.getAnnotation(MappedTypes.class);
    if (mappedTypes != null) {
      // 遍历注解的 Java Type 数组逐个进行注册
      for (Class<?> javaTypeClass : mappedTypes.value()) {
        register(javaTypeClass, typeHandlerClass);
        mappedTypeFound = true;
      }
    }

    // <4> 未使用 @MappedTypes 注解，直接进行注册
    if (!mappedTypeFound) {
      register(getInstance(null, typeHandlerClass));
    }
  }
  ```
  - 分成`<3>` `<4>`两种情况
  - `<3>` 处,基于`@MappedTypes`注解,调用`#register(Class<?> javaTypeClass,Class<?> typeHandlerClass)`方法，注册指定 Java Type的指定TypeHandler类。代码如下：
  ```java
  public void register(Class<?> javaTypeClass, Class<?> typeHandlerClass) {
    register(javaTypeClass, getInstance(javaTypeClass, typeHandlerClass));
  }
  ```
  - 调用③ `#register(Clas<T> javaType, TypeHandler<? extends T> typeHandler)`方法，注册指定Java Type的指定 TypeHandler对象，代码如下：
  ```java
    private <T> void register(Type javaType, TypeHandler<? extends T> typeHandler) {
    // 获得 @MappedJdbcTypes 注解
    MappedJdbcTypes mappedJdbcTypes = typeHandler.getClass().getAnnotation(MappedJdbcTypes.class);
    if (mappedJdbcTypes != null) {
      // 遍历 MappedJdbcTypes 注册的 JDBC Type 进行注册
      for (JdbcType handledJdbcType : mappedJdbcTypes.value()) {
        register(javaType, handledJdbcType, typeHandler);
      }
      if (mappedJdbcTypes.includeNullJdbcType()) {
        // <5>
        register(javaType, null, typeHandler);  // jdbcType = null
      }
    } else {
      // <5>
      register(javaType, null, typeHandler);  // jdbcType = null
    }
  }
  ```
  - 有`@MappedJdbcTypes`注解的④ `#register(Type javaType, JdbcType jdbcType, TypeHandler<?> handler)`方法，发起最终注册
  - 对于`<5>`处，发起注册时， `jdbcType`参数为`null`。后续说明原因.
- `<4>`处，调用② `#register(TypeHandler<T> typeHandler)`方法，未使用`@MappedTypes`注解，调用`#register(TypeHandler<T> typeHandler)`方法，注册 TypeHandler对象。代码如下：
```java
  @SuppressWarnings("unchecked")
  public <T> void register(TypeHandler<T> typeHandler) {
    boolean mappedTypeFound = false;
    // <5> 获得 @MappedTypes 注解
    MappedTypes mappedTypes = typeHandler.getClass().getAnnotation(MappedTypes.class);
    // 优先，使用 @MappedTypes 注解的 Java Type 进行注册
    if (mappedTypes != null) {
      for (Class<?> handledType : mappedTypes.value()) {
        register(handledType, typeHandler);
        mappedTypeFound = true;
      }
    }
    // @since 3.1.0 - try to auto-discover the mapped type
    // <6> 其次，当 typeHandler 为 TypeReference 子类时，进行注册
    if (!mappedTypeFound && typeHandler instanceof TypeReference) {
      try {
        TypeReference<T> typeReference = (TypeReference<T>) typeHandler;
        register(typeReference.getRawType(), typeHandler);  // Java Type 为 <T> 泛型
        mappedTypeFound = true;
      } catch (Throwable t) {
        // maybe users define the TypeReference with a different type and are not assignable, so just ignore it
      }
    }
    // <7> 最差，使用Java Type 为 null 进行注册
    if (!mappedTypeFound) {
      register((Class<T>) null, typeHandler);
    }
  }
```
- 分成三种情况，最终都是调用`#register(Type javaType, TypeHandler<? extends T> typeHandler)`方法，进行注册，也就是跳到③

- ⑤ `#register(JdbcType jdbcType, TypeHandler<?> handler)`方法，注册`handler`到`JDBC_TYPE_HANDLER_MAP`中，代码如下：
```java
  public void register(JdbcType jdbcType, TypeHandler<?> handler) {
    jdbcTypeHandlerMap.put(jdbcType, handler);
  }
```
- 和上述方法不同

## getTypeHandler
`#getTypeHandler(...)`方法，获得`TypeHandler`。有大量的重载实现，大体如下：

> FROM 徐群明 [<Mybatis技术内幕>](https://item.jd.com/12125531.html)
> {% asset_img getTypeHandler.png getTypeHandler 方法 %}

从图中，我们可以看到，最终会调用 ① 处的 `#getTypeHandler(Type type, JdbcType jdbcType)` 方法。当然，我们先来看看三种调用的情况：

- 调用情况一： `#getTypeHandler(Class<?> type)`方法，代码如下:
```java
// TypeHandlerRegistry.java

public <T> TypeHandler<T> getTypeHandler(Class<T> type) {
    return getTypeHandler((Type) type, null);
}
```
  - jdbcType 为 null
- 调用情况二：`#getTypeHandler(Class<T> type, JdbcType jdbcType)` 方法，代码如下：
```java
// TypeHandlerRegistry.java

public <T> TypeHandler<T> getTypeHandler(Class<T> type, JdbcType jdbcType) {
    return getTypeHandler((Type) type, jdbcType);
}
```
 - 将`tyoe`转换为Type类型

- 调用情况三：`#getTypeHandler(TypeReference<T> javaTypeReference, ...)`方法，代码如下：
```java
// TypeHandlerRegistry.java

public <T> TypeHandler<T> getTypeHandler(TypeReference<T> javaTypeReference) {
    return getTypeHandler(javaTypeReference, null);
}

public <T> TypeHandler<T> getTypeHandler(TypeReference<T> javaTypeReference, JdbcType jdbcType) {
    return getTypeHandler(javaTypeReference.getRawType(), jdbcType);
}
```

- 看看①处的方法
```java
 @SuppressWarnings("unchecked")
  private <T> TypeHandler<T> getTypeHandler(Type type, JdbcType jdbcType) {
    // 忽略 ParamMap 的情况
    if (ParamMap.class.equals(type)) {
      return null;
    }
    // <1> 获得 Java Type 对应的 TypeHandler 的集合
    Map<JdbcType, TypeHandler<?>> jdbcHandlerMap = getJdbcHandlerMap(type);
    TypeHandler<?> handler = null;
    if (jdbcHandlerMap != null) {
      // <2.1> 优先，使用 jdbcType 获取对应的 TypeHandler
      handler = jdbcHandlerMap.get(jdbcType);
      if (handler == null) {
        // <2.2> 其次，使用 null 获取对应的 TypeHandler ，可以认为是默认的 TypeHandler
        handler = jdbcHandlerMap.get(null);
      }
      // <2.3> 最差，从 TypeHandler 集合中选择一个唯一的 TypeHandler
      if (handler == null) {
        // #591
        handler = pickSoleHandler(jdbcHandlerMap);
      }
    }
    // type drives generics here
    return (TypeHandler<T>) handler;
  }
```
- `<1>`处，调用 `#getJdbcHandlerMap(Type type)`` 方法，获得 Java Type 对应的 TypeHandler 集合。代码如下：
  ```java
  private Map<JdbcType, TypeHandler<?>> getJdbcHandlerMap(Type type) {
    // <1> 获得 Java Type 对应的 TypeHandler 集合
    Map<JdbcType, TypeHandler<?>> jdbcHandlerMap = typeHandlerMap.get(type);
    // <1.2> 如果为 NULL_TYPE_HANDLER_MAP ，意味着为空，直接返回
    if (NULL_TYPE_HANDLER_MAP.equals(jdbcHandlerMap)) {
      return null;
    }
    // <1.3> 如果找不到
    if (jdbcHandlerMap == null && type instanceof Class) {
      Class<?> clazz = (Class<?>) type;
      // 如果是枚举类型
      if (Enum.class.isAssignableFrom(clazz)) {
        // 获得父类对应的 TypeHandler 集合
        Class<?> enumClass = clazz.isAnonymousClass() ? clazz.getSuperclass() : clazz;
        jdbcHandlerMap = getJdbcHandlerMapForEnumInterfaces(enumClass, enumClass);
        // 如果找不到
        if (jdbcHandlerMap == null) {
          // 注册 defaultEnumTypeHandler ，并使用它
          register(enumClass, getInstance(enumClass, defaultEnumTypeHandler));
          // 返回结果
          return typeHandlerMap.get(enumClass);
        }
        // 非枚举类型
      } else {
        // 获得父类对应的 TypeHandler 集合
        jdbcHandlerMap = getJdbcHandlerMapForSuperclass(clazz);
      }
    }
    // <1.4> 如果结果为空，设置为 NULL_TYPE_HANDLER_MAP ，提升查找速度，避免二次查找
    typeHandlerMap.put(type, jdbcHandlerMap == null ? NULL_TYPE_HANDLER_MAP : jdbcHandlerMap);
    // 返回结果
    return jdbcHandlerMap;
  }
  ```
  - <1.1> 处，获得 Java Type 对应的 TypeHandler 集合。
  - <1.2> 处，如果为 NULL_TYPE_HANDLER_MAP ，意味着为空，直接返回。原因可见 <1.4> 处。
  - <1.3> 处，找不到，则根据 type 是否为枚举类型，进行不同处理。
    + 【枚举】
    + 先调用 `#getJdbcHandlerMapForEnumInterfaces(Class<?> clazz, Class<?> enumClazz)` 方法， 获得父类对应的 TypeHandler 集合。代码如下：
    ```java
    private Map<JdbcType, TypeHandler<?>> getJdbcHandlerMapForEnumInterfaces(Class<?> clazz, Class<?> enumClazz) {
      // 遍历枚举类的所有接口
      for (Class<?> iface : clazz.getInterfaces()) {
        // 获得该接口对应的 jdbcHandlerMap 集合
        Map<JdbcType, TypeHandler<?>> jdbcHandlerMap = typeHandlerMap.get(iface);
        // 为空，递归 getJdbcHandlerMapForEnumInterfaces 方法，继续从父类对应的 TypeHandler 集合
        if (jdbcHandlerMap == null) {
          jdbcHandlerMap = getJdbcHandlerMapForEnumInterfaces(iface, enumClazz);
        }
        // 如果找到，则从 jdbcHandlerMap 初始化中 newMap 中，并进行返回
        if (jdbcHandlerMap != null) {
          // Found a type handler regsiterd to a super interface
          HashMap<JdbcType, TypeHandler<?>> newMap = new HashMap<>();
          for (Entry<JdbcType, TypeHandler<?>> entry : jdbcHandlerMap.entrySet()) {
            // Create a type handler instance with enum type as a constructor arg
            newMap.put(entry.getKey(), getInstance(enumClazz, entry.getValue().getClass()));
          }
          return newMap;
        }
      }
      // 找不到，则返回 null
      return null;
    }
    ```
    + 调用 `#getJdbcHandlerMapForSuperclass(Class<?> clazz)` 方法，获得父类对应的 TypeHandler 集合。代码如下：
    ```java
    private Map<JdbcType, TypeHandler<?>> getJdbcHandlerMapForSuperclass(Class<?> clazz) {
      // 获得父类
      Class<?> superclass =  clazz.getSuperclass();
      // 不存在非 Object 的父类，返回null
      if (superclass == null || Object.class.equals(superclass)) {
        return null;
      }
      // 找到父类对应的 TypeHandler 集合
      Map<JdbcType, TypeHandler<?>> jdbcHandlerMap = typeHandlerMap.get(superclass);
      // 找到则直接返回，找不到递归调用，继续查找父类的 TypeHandler
      if (jdbcHandlerMap != null) {
        return jdbcHandlerMap;
      } else {
        return getJdbcHandlerMapForSuperclass(superclass);
      }
    }
    ```
  - <2.1> 处，优先，使用 jdbcType 获取对应的 TypeHandler 。

  - <2.2> 处，其次，使用 null 获取对应的 TypeHandler ，可以认为是默认的 TypeHandler 。这里是解决一个 Java Type 可能对应多个 TypeHandler 的方式之一。
  - <2.3> 处，最差，调用 #pickSoleHandler(Map<JdbcType, TypeHandler<?>> jdbcHandlerMap) 方法，从 TypeHandler 集合中选择一个唯一的 TypeHandler 。代码如下：
  ```java
  private TypeHandler<?> pickSoleHandler(Map<JdbcType, TypeHandler<?>> jdbcHandlerMap) {
    TypeHandler<?> soleHandler = null;
    // 选择一个
    for (TypeHandler<?> handler : jdbcHandlerMap.values()) {
      if (soleHandler == null) {
        soleHandler = handler;
      // 如果还有，并且不同类，那么不好选择，所以返回 null
      } else if (!handler.getClass().equals(soleHandler.getClass())) {
        // More than one type handlers registered.
        return null;
      }
    }
    return soleHandler;
  }
  ```

# TypeAliasRegistry
`org.apache.ibatis.type.TypeAliasRegistry` ，类型与别名的注册表。通过别名，我们在 Mapper XML 中的 resultType 和 parameterType 属性，直接使用，而不用写全类名。

## 构造方法
```java
 /**
   * 类型与别名的映射
   */
  private final Map<String, Class<?>> typeAliases = new HashMap<>();

  /**
   * 构造方法，初始化默认的类型与别名
   */
  public TypeAliasRegistry() {
    registerAlias("string", String.class);

    registerAlias("byte", Byte.class);
    registerAlias("long", Long.class);
    registerAlias("short", Short.class);
    registerAlias("int", Integer.class);
    registerAlias("integer", Integer.class);
    registerAlias("double", Double.class);
    registerAlias("float", Float.class);
    registerAlias("boolean", Boolean.class);

    // ... 省略其他注册
  }
```

## registerAlias
`#registerAlias(Class<?> type)` 方法，注册指定类。代码如下：
```java
public void registerAlias(Class<?> type) {
  // <1> 默认为，简单类名
  String alias = type.getSimpleName();
  // <2> 如果有注解，使用注解上的名称
  Alias aliasAnnotation = type.getAnnotation(Alias.class);
  if (aliasAnnotation != null) {
    alias = aliasAnnotation.value();
  }
  // <3> 注册类型与别名的注册表
  registerAlias(alias, type);
}
```
- `<1>` ，默认为，简单类名。
- `<2>` ，可通过 @Alias 注解的别名。
- `<3>` ，调用 `#registerAlias(String alias, Class<?> value)` 方法，注册类型与别名的注册表。代码如下：

```java
  public void registerAlias(String alias, Class<?> value) {
    if (alias == null) {
      throw new TypeException("The parameter alias cannot be null");
    }
    // issue #748
    // <1> 转换成小写
    String key = alias.toLowerCase(Locale.ENGLISH);
    // <2> 如果冲突，抛出 TypeException 异常
    if (typeAliases.containsKey(key) && typeAliases.get(key) != null && !typeAliases.get(key).equals(value)) {
      throw new TypeException("The alias '" + alias + "' is already mapped to the value '" + typeAliases.get(key).getName() + "'.");
    }
    // <3>
    typeAliases.put(key, value);
  }
```

- `<1>` 处，将别名转换成**小写**。这样的话，无论我们在 Mapper XML 中，写 `String` 还是 `string` 甚至是 `STRING` ，都是对应的 String 类型。
- `<2>` 处，如果已经注册，并且类型不一致，说明有冲突，抛出 TypeException 异常。
- `<3>` 处，添加到 `TYPE_ALIASES` 中。
- 另外，`#registerAlias(String alias, String value)` 方法，也会调用该方法。代码如下：
```java
  public void registerAlias(String alias, String value) {
    try {
      registerAlias(alias, Resources.classForName(value));
    } catch (ClassNotFoundException e) {
      throw new TypeException("Error registering type alias " + alias + " for " + value + ". Cause: " + e, e);
    }
  }
```

## registerAliases
`#registerAliases(String packageName, ...)` 方法，扫描指定包下的所有类，并进行注册。代码如下：
```java
  /**
   * 注册指定包下的别名与类的映射
   * @param packageName 指定包
   */
  public void registerAliases(String packageName) {
    registerAliases(packageName, Object.class);
  }

  /**
   * 注册指定包下的别名与类的映射。另外，要求类必须是 {@param superType} 类型（包括子类）。
   * @param packageName 指定包
   * @param superType 指定父类
   */
  public void registerAliases(String packageName, Class<?> superType) {
    // 获得指定包下的所有类
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<>();
    resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
    Set<Class<? extends Class<?>>> typeSet = resolverUtil.getClasses();
    // 遍历，逐个注册类型与别名的注册表
    for (Class<?> type : typeSet) {
      // Ignore inner classes and interfaces (including package-info.java)
      // Skip also inner classes. See issue #6
      // 排除匿名类，接口，内部类
      if (!type.isAnonymousClass() && !type.isInterface() && !type.isMemberClass()) {
        registerAlias(type);
      }
    }
  }
```

## 7.4 resolveAlias

`#resolveAlias(String string)` 方法，获得别名对应的类型。代码如下：
```java
  public <T> Class<T> resolveAlias(String string) {
    try {
      if (string == null) {
        return null;
      }
      // issue #748
      // <1> 转换成小写
      String key = string.toLowerCase(Locale.ENGLISH);
      Class<T> value;
      // <2.1> 首先，从 TYPE_ALIASES 中获取
      if (typeAliases.containsKey(key)) {
        value = (Class<T>) typeAliases.get(key);
      } else {
        // <2.2> 其次，直接获得对应类
        value = (Class<T>) Resources.classForName(string);
      }
      return value;
    } catch (ClassNotFoundException e) { // <2.3> 异常
      throw new TypeException("Could not resolve type alias '" + string + "'.  Cause: " + e, e);
    }
  }
```

# 其他
## SimpleTypeRegistry
`org.apache.ibatis.type.SimpleTypeRegistry` ，简单类型注册表。

## ByteArrayUtils
`org.apache.ibatis.type.ByteArrayUtils` ，Byte 数组的工具类。











