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



## 测试

### SimpleContextConfig

```java
public class SimpleContextConfig implements LifecycleListener {

    private Context context;

    /**
     * 监听方法
     * @param event 事件
     */
    public void lifecycleEvent(LifecycleEvent event) {
        if (Lifecycle.START_EVENT.equals(event.getType())) {
            context = (Context) event.getLifecycle();
            authenticatorConfig();
            context.setConfigured(true);
        }
    }

    /**
     * 验证器配置
     */
    private synchronized void authenticatorConfig() {
        // 是否配置过滤参数
        SecurityConstraint constraints[] = context.findConstraints();
        if ((constraints == null) || (constraints.length == 0))
            return;
        // 是否进行验证器配置
        LoginConfig loginConfig = context.getLoginConfig();
        if (loginConfig == null) {
            loginConfig = new LoginConfig("NONE", null, null, null);
            context.setLoginConfig(loginConfig);
        }

        // 如果已经设置了验证器直接返回
        Pipeline pipeline = ((StandardContext) context).getPipeline();
        if (pipeline != null) {
            Valve basic = pipeline.getBasic();
            if ((basic != null) && (basic instanceof Authenticator))
                return;
            Valve valves[] = pipeline.getValves();
            for (int i = 0; i < valves.length; i++) {
                if (valves[i] instanceof Authenticator)
                    return;
            }
        }
        else { // no Pipeline, cannot install authenticator valve
            return;
        }

        // Context是否配置验证器
        if (context.getRealm() == null) {
            return;
        }

        // 确定需要配置的验证器类名
        String authenticatorName = "org.apache.catalina.authenticator.BasicAuthenticator";
        // 实例化验证器并安装
        Valve authenticator = null;
        try {
            Class authenticatorClass = Class.forName(authenticatorName);
            authenticator = (Valve) authenticatorClass.newInstance();
            ((StandardContext) context).addValve(authenticator);
            System.out.println("Added authenticator valve to Context");
        }
        catch (Throwable t) {
        }
    }

```

- 监听器类，当触发`START`事件时，进行验证器配置，增加`BasicAuthenticator`到流水线

### BasicAuthenticator

```java
// BasicAuthenticator.java

	public boolean authenticate(HttpRequest request, HttpResponse response, LoginConfig config) throws IOException {
        Principal principal = ((HttpServletRequest)request.getRequest()).getUserPrincipal();
        if (principal != null) {
            if (this.debug >= 1) {
                this.log("Already authenticated '" + principal.getName() + "'");
            }

            return true;
        } else {
			// ...省略部分代码
            principal = this.context.getRealm().authenticate(username, password);	// <1>
            if (principal != null) {
                this.register(request, response, principal, "BASIC", username, password);
                return true;
            } else {
                String realmName = config.getRealmName();
                if (realmName == null) {
                    realmName = hreq.getServerName() + ":" + hreq.getServerPort();
                }

                hres.setHeader("WWW-Authenticate", "Basic realm=\"" + realmName + "\"");
                hres.setStatus(401);
                return false;
            }
        }
    }

```

- `<1>`处，调用容器配置的验证器进行权限验证

### SimpleRealm

```java
public class SimpleRealm implements Realm {

    public SimpleRealm() {
        createUserDatabase();
    }

    private Container container;
    private ArrayList users = new ArrayList();

    public Container getContainer() {
        return container;
    }

    public void setContainer(Container container) {
        this.container = container;
    }

    public String getInfo() {
        return "A simple Realm implementation";
    }

    public void addPropertyChangeListener(PropertyChangeListener listener) {
    }

    public Principal authenticate(String username, String credentials) {
        System.out.println("SimpleRealm.authenticate()");
        if (username==null || credentials==null)
            return null;
        User user = getUser(username, credentials);
        if (user==null)
            return null;
        return new GenericPrincipal(this, user.username, user.password, user.getRoles());
    }

    public Principal authenticate(String username, byte[] credentials) {
        return null;
    }

    public Principal authenticate(String username, String digest, String nonce,
                                  String nc, String cnonce, String qop, String realm, String md5a2) {
        return null;
    }

    public Principal authenticate(X509Certificate certs[]) {
        return null;
    }

    public boolean hasRole(Principal principal, String role) {
        if ((principal == null) || (role == null) ||
                !(principal instanceof GenericPrincipal))
            return (false);
        GenericPrincipal gp = (GenericPrincipal) principal;
        if (!(gp.getRealm() == this))
            return (false);
        boolean result = gp.hasRole(role);
        return result;
    }

    public void removePropertyChangeListener(PropertyChangeListener listener) {
    }

    private User getUser(String username, String password) {
        Iterator iterator = users.iterator();
        while (iterator.hasNext()) {
            User user = (User) iterator.next();
            if (user.username.equals(username) && user.password.equals(password))
                return user;
        }
        return null;
    }

    private void createUserDatabase() {
        User user1 = new User("admin", "admin");
        user1.addRole("manager");
        user1.addRole("programmer");
        User user2 = new User("cindy", "bamboo");
        user2.addRole("programmer");

        users.add(user1);
        users.add(user2);
    }

    class User {

        public User(String username, String password) {
            this.username = username;
            this.password = password;
        }

        public String username;
        public ArrayList roles = new ArrayList();
        public String password;

        public void addRole(String role) {
            roles.add(role);
        }
        public ArrayList getRoles() {
            return roles;
        }
    }
}
```

- `#authenticate`方法，验证方法，验证通过返回`Principal`对象

### Bootstrap1

```java
public final class Bootstrap1 {
    public static void main(String[] args) {

        //invoke: http://localhost:8080/Modern or  http://localhost:8080/Primitive

        System.setProperty("catalina.base", System.getProperty("user.dir"));
        Connector connector = new HttpConnector();
        Wrapper wrapper1 = new SimpleWrapper();
        wrapper1.setName("Primitive");
        wrapper1.setServletClass("PrimitiveServlet");
        Wrapper wrapper2 = new SimpleWrapper();
        wrapper2.setName("Modern");
        wrapper2.setServletClass("ModernServlet");

        Context context = new StandardContext();
        // StandardContext's start method adds a default mapper
        context.setPath("/myApp");
        context.setDocBase("myApp");

        // 添加事件监听器
        LifecycleListener listener = new SimpleContextConfig();
        ((Lifecycle) context).addLifecycleListener(listener);	// <1>

        context.addChild(wrapper1);
        context.addChild(wrapper2);

        Loader loader = new WebappLoader();
        context.setLoader(loader);
        // context.addServletMapping(pattern, name);
        context.addServletMapping("/Primitive", "Primitive");
        context.addServletMapping("/Modern", "Modern");
        // add ContextConfig. This listener is important because it configures
        // StandardContext (sets configured to true), otherwise StandardContext
        // won't start

        // 添加安全链接集合	<2>
        SecurityCollection securityCollection = new SecurityCollection();
        securityCollection.addPattern("/");
        securityCollection.addMethod("GET");

        // 设置安全规则 <3>
        SecurityConstraint constraint = new SecurityConstraint();
        constraint.addCollection(securityCollection);
        constraint.addAuthRole("manager");
        LoginConfig loginConfig = new LoginConfig();
        loginConfig.setRealmName("Simple Realm");
        
        // 设置验证器	<4>
        Realm realm = new SimpleRealm();
        context.setRealm(realm);
        context.addConstraint(constraint);
        context.setLoginConfig(loginConfig);

        connector.setContainer(context);

        try {
            connector.initialize();
            ((Lifecycle) connector).start();
            ((Lifecycle) context).start();

            // make the application wait until we press a key.
            System.in.read();
            ((Lifecycle) context).stop();
        }
        catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

- `<1>`处，添加自定义事件监听器，当容器启动时，增加`BasicAuthenticator`到容器内置流水线中。

- `<2>`处，增加安全连接集合
- `<3>`处，设置安全规则
- `<4>`处，设置自定义验证器到容器

### 验证

启动`Bootstrap1`,访问`http://localhost:8080/Modern`

