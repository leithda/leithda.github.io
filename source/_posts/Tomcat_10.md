title: Tomcat-安全
categories:
  - Java
  - Tomcat
tags:
  - 源码
  - Tomcat
author: 长歌
date: 2019-10-18
---

有些 web 应用程序的内容是有限制的，只允许有权限的用户在提供正确的用户名和密码的情况下才允许访问。Servlet 通过配置部署文件 web.xml 来对安全性提 供技术支持。本章的主要内容是容器对于安全性限制的支持
<!-- More -->

## 介绍
一个`servlet`通过一个`authenticator`的处理器进行安全性限制，容器启动时，`authenticator`被添加到容器的流水线上

## (域)Realm
```java
public interface Realm {
    Container getContainer();

    void setContainer(Container var1);

    String getInfo();

    void addPropertyChangeListener(PropertyChangeListener var1);

    Principal authenticate(String var1, String var2);

    Principal authenticate(String var1, byte[] var2);

    Principal authenticate(String var1, String var2, String var3, String var4, String var5, String var6, String var7, String var8);

    Principal authenticate(X509Certificate[] var1);

    boolean hasRole(Principal var1, String var2);

    void removePropertyChangeListener(PropertyChangeListener var1);
}
```
- 基本实现类为`org.apache.catalina.realm.RealmBase`

## (用户)GenericPrincipal 
```java
public interface Principal {


    public boolean equals(Object another);


    public String toString();

 
    public int hashCode();

 
    public String getName();

 
    public default boolean implies(Subject subject) {
        if (subject == null)
            return false;
        return subject.getPrincipals().contains(this);
    }
}
```
- 一个 Principal 用接口表示，tomcat中，该接口的实现为``
```java
public class GenericPrincipal implements TomcatPrincipal, Serializable {
    private static final long serialVersionUID = 1L;
    protected final String name;
    protected final String password;
    protected final String[] roles;
    protected final Principal userPrincipal;
    protected final transient LoginContext loginContext;
    protected transient GSSCredential gssCredential;

    public GenericPrincipal(String name, String password, List<String> roles) {
        this(name, password, roles, (Principal)null);
    }

    public GenericPrincipal(String name, String password, List<String> roles, Principal userPrincipal) {
        this(name, password, roles, userPrincipal, (LoginContext)null);
    }

    public GenericPrincipal(String name, String password, List<String> roles, Principal userPrincipal, LoginContext loginContext) {
        this(name, password, roles, userPrincipal, loginContext, (GSSCredential)null);
    }

    public GenericPrincipal(String name, String password, List<String> roles, Principal userPrincipal, LoginContext loginContext, GSSCredential gssCredential) {
        this.gssCredential = null;
        this.name = name;
        this.password = password;
        this.userPrincipal = userPrincipal;
        if (roles == null) {
            this.roles = new String[0];
        } else {
            this.roles = (String[])roles.toArray(new String[roles.size()]);
            if (this.roles.length > 1) {
                Arrays.sort(this.roles);
            }
        }

        this.loginContext = loginContext;
        this.gssCredential = gssCredential;
    }

    public String getName() {
        return this.name;
    }

    public String getPassword() {
        return this.password;
    }

    public String[] getRoles() {
        return this.roles;
    }

    public Principal getUserPrincipal() {
        return (Principal)(this.userPrincipal != null ? this.userPrincipal : this);
    }

    public GSSCredential getGssCredential() {
        return this.gssCredential;
    }

    protected void setGssCredential(GSSCredential gssCredential) {
        this.gssCredential = gssCredential;
    }

    public boolean hasRole(String role) {
        if ("*".equals(role)) {
            return true;
        } else if (role == null) {
            return false;
        } else {
            return Arrays.binarySearch(this.roles, role) >= 0;
        }
    }

    public String toString() {
        StringBuilder sb = new StringBuilder("GenericPrincipal[");
        sb.append(this.name);
        sb.append("(");

        for(int i = 0; i < this.roles.length; ++i) {
            sb.append(this.roles[i]).append(",");
        }

        sb.append(")]");
        return sb.toString();
    }

    public void logout() throws Exception {
        if (this.loginContext != null) {
            this.loginContext.logout();
        }

        if (this.gssCredential != null) {
            this.gssCredential.dispose();
        }

    }

    private Object writeReplace() {
        return new GenericPrincipal.SerializablePrincipal(this.name, this.password, this.roles, this.userPrincipal);
    }

    private static class SerializablePrincipal implements Serializable {
        private static final long serialVersionUID = 1L;
        private final String name;
        private final String password;
        private final String[] roles;
        private final Principal principal;

        public SerializablePrincipal(String name, String password, String[] roles, Principal principal) {
            this.name = name;
            this.password = password;
            this.roles = roles;
            if (principal instanceof Serializable) {
                this.principal = principal;
            } else {
                this.principal = null;
            }

        }

        private Object readResolve() {
            return new GenericPrincipal(this.name, this.password, Arrays.asList(this.roles), this.principal);
        }
    }
}
```
- 通过构造方法，关联Reale
- `#hashRole`方法用于检查一个 Principal 是否有特定的角色

## LoginConfig
```java
public final class LoginConfig {
    private String authMethod = null;
    private String errorPage = null;
    private String loginPage = null;
    private String realmName = null;

    public LoginConfig() {
    }

    public LoginConfig(String authMethod, String realmName, String loginPage, String errorPage) {
        this.setAuthMethod(authMethod);
        this.setRealmName(realmName);
        this.setLoginPage(loginPage);
        this.setErrorPage(errorPage);
    }

    // ... setter/getter

    public String toString() {
        // ... 省略部分代码
    }
}
```
- 用来保存安全校验信息，校验的方法`authMethod`，登录页面`loginPage`，错误页面`errorPage`以及域名`realmName`

## Authenticator
- `org.apache.catalina.Authenticator`接口用于表示一个验证器。接口内无方法，只是一个标志，使用时使用`#instanceof`检查组件是否为验证器

- Catalina提供了Authenticator接口的基本实现:`org.apache.catalina.authenticator.AuthenticatorBase`.除了实现接口，这个类还集成了`ValueBase`类，这个类也是一个处理器。

- `org.apache.catalina.authenticator`包下还有几个其他实现类：

  | `LogConfig`的authMethod | 对应的验证器          | 说明                                                       |
  | ----------------------- | --------------------- | ---------------------------------------------------------- |
  | BASIC                   | BasicAutnenticator    | 基本验证                                                   |
  | FORM                    | FormAuthenticator     | 表单验证                                                   |
  | DIGEST                  | DigestAuthenticator   | 摘要(digest)验证                                           |
  | CLIENT-CERT             | SSLAuthenticator      | SSL验证                                                    |
  |                         | NonLoginAutnenticator | 只检查安全限制，不校验用户，当Tomcat没有指定验证元素时使用 |

  - `AuthenticatorBase`中使用了抽象方法`#authenticate`,具体的实现由子类完成

- 对应的类结构如图:
{% asset_img Authenticator.png 验证器类结构%}
