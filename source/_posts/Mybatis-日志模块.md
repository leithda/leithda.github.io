---
title: Mybatis源码解析-日志模块
categories:
  - Java
  - Mybatis
tags:
  - 源码
  - Mybatis
author: 长歌
abbrlink: 1547130688
date: 2020-03-31 22:21:59
---

{% cq %}
MyBatis 作为一个设计优良的框架，除了提供详细的日志输出信息，还要能够集成多种日志框架，其日志模块的一个主要功能就是集成第三方日志框架。
{% endcq %}


# 概述
Mybatis的日志模块对应`logging`包，使用适配器模式对多种日志框架进行适配，涉及的类主要有以下这些：
{% asset_img Log.png 日志模块UML图 %}

# LogFactory
`org.apache.ibatis.logging.LogFactory ` ，Log 工厂类。

## 构造方法

```java
  public static final String MARKER = "MYBATIS";

  private static Constructor<? extends Log> logConstructor;

  static {
    // <1> 逐个尝试，判断使用哪个 Log 的实现类，即初始化 logConstructor 属性
    tryImplementation(LogFactory::useSlf4jLogging);
    tryImplementation(LogFactory::useCommonsLogging);
    tryImplementation(LogFactory::useLog4J2Logging);
    tryImplementation(LogFactory::useLog4JLogging);
    tryImplementation(LogFactory::useJdkLogging);
    tryImplementation(LogFactory::useNoLogging);
  }

  private LogFactory() {
    // disable construction
  }
```

- `<1>`处，调用`#tryImplementation(Runnable runnable)` 方法尝试初始化log实例。按照Slf4j,CommonsLog,Log4J2..顺序尝试初始化，即初始化`logConstructor`属性。
- `#tryImplementation(Runnable runnable)` 方法，判断使用哪个Log的实现类。代码如下：

```java
  private static void tryImplementation(Runnable runnable) {
    if (logConstructor == null) {
      try {
        runnable.run();
      } catch (Throwable t) {
        // ignore
      }
    }
  }
```

- `useSlf4jLogging` 方法，尝试使用Slf4j。代码如下：
```java
public static synchronized void useSlf4jLogging() {
    setImplementation(org.apache.ibatis.logging.slf4j.Slf4jImpl.class);
}
```
- 在该方法内部，会调用 `#setImplementation(Class<? extends Log> implClass)` 方法，尝试使用指定的 Log 实现类，例如此处为 `org.apache.ibatis.logging.slf4j.Slf4jImpl` 。代码如下：
```java
  private static void setImplementation(Class<? extends Log> implClass) {
    try {
      // 获得参数为String类型的构造方法
      Constructor<? extends Log> candidate = implClass.getConstructor(String.class);
      // 创建 Log 对象
      Log log = candidate.newInstance(LogFactory.class.getName());
      if (log.isDebugEnabled()) {
        log.debug("Logging initialized using '" + implClass + "' adapter.");
      }
      // 如果初始化成功，设置为 logConstructor
      logConstructor = candidate;
    } catch (Throwable t) {
      throw new LogException("Error setting Log implementation.  Cause: " + t, t);
    }
  }
```
- 如果初始化成功，logConstructor有值，其他的日志实例不会再进行初始化
- 可以通过 `useCustomLogging<Class<? extendslog> clazz) `进行自定义的日志配置

## getLog
`#getLog(...) `方法，获得 Log 对象。代码如下：

```java
public static Log getLog(Class<?> aClass) {
    return getLog(aClass.getName());
}

public static Log getLog(String logger) {
    try {
        return logConstructor.newInstance(logger);
    } catch (Throwable t) {
        throw new LogException("Error creating logger for logger " + logger + ".  Cause: " + t, t);
    }
}
```

# Log
`org.apache.ibatis.logging.Log ` ，MyBatis Log 接口。代码如下：

```java
public interface Log {

  boolean isDebugEnabled();

  boolean isTraceEnabled();

  void error(String s, Throwable e);

  void error(String s);

  void debug(String s);

  void trace(String s);

  void warn(String s);

}
```

## StdOutImpl
`org.apache.ibatis.logging.stdout.StdOutImpl `，实现 Log 接口，StdOut 实现类。代码如下：

```java
public class StdOutImpl implements Log {

  public StdOutImpl(String clazz) {
    // Do Nothing
  }

  @Override
  public boolean isDebugEnabled() {
    return true;
  }

  @Override
  public boolean isTraceEnabled() {
    return true;
  }

  @Override
  public void error(String s, Throwable e) {
    System.err.println(s);
    e.printStackTrace(System.err);
  }

  @Override
  public void error(String s) {
    System.err.println(s);
  }

  @Override
  public void debug(String s) {
    System.out.println(s);
  }

  @Override
  public void trace(String s) {
    System.out.println(s);
  }

  @Override
  public void warn(String s) {
    System.out.println(s);
  }
}
```

## Slf4jImpl
`org.apache.ibatis.logging.slf4j.Slf4jImpl ` ，实现 Log 接口，Slf4j 实现类。代码如下：

```java
public class Slf4jImpl implements Log {

  private Log log;

  public Slf4jImpl(String clazz) {
    // 使用 SLF LoggerFactory 获得 SLF Logger 对象
    Logger logger = LoggerFactory.getLogger(clazz);

    // 如果是 LocationAwareLogger ，则创建 Slf4jLocationAwareLoggerImpl 对象
    if (logger instanceof LocationAwareLogger) {
      try {
        // check for slf4j >= 1.6 method signature
        logger.getClass().getMethod("log", Marker.class, String.class, int.class, String.class, Object[].class, Throwable.class);
        log = new Slf4jLocationAwareLoggerImpl((LocationAwareLogger) logger);
        return;
      } catch (SecurityException | NoSuchMethodException e) {
        // fail-back to Slf4jLoggerImpl
      }
    }

    // Logger is not LocationAwareLogger or slf4j version < 1.6
    // 否则，创建 Slf4jLoggerImpl 对象
    log = new Slf4jLoggerImpl(logger);
  }

  @Override
  public boolean isDebugEnabled() {
    return log.isDebugEnabled();
  }

  @Override
  public boolean isTraceEnabled() {
    return log.isTraceEnabled();
  }

  @Override
  public void error(String s, Throwable e) {
    log.error(s, e);
  }

  @Override
  public void error(String s) {
    log.error(s);
  }

  @Override
  public void debug(String s) {
    log.debug(s);
  }

  @Override
  public void trace(String s) {
    log.trace(s);
  }

  @Override
  public void warn(String s) {
    log.warn(s);
  }

}
```

# BaseJdbcLogger
在 `org.apache.ibatis.logging ` 包下的 `jdbc` 包，有如下五个类：
- BaseJdbcLogger
  - ConnectionLogger
  - PreparedStatementLogger
  - StatementLogger
  - ResultSetLogger

- 一个基于 JDBC 接口实现增强的案例，而原理上，也是基于 JDK 实现动态代理。




















