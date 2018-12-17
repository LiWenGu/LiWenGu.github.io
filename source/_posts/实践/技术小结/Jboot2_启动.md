---
title: Jboot_2
date: 2018-01-10 12:00:00
updated: 2018-01-12 18:37:00
tags:
  - jboot
categories: 
  - 小结
  - jboot
comments: true
permalink: tech/Jboot_2.html    
---

# 0 启动流程

1. 解析启动参数，打印 logo。
2. 通过工厂对配置进行判断获取相应的应用服务器（默认 undertow）。
3. 判断是否是开发模式（默认），如果是则定期对文件进行扫描（3 * 1010）。
4. 回调各个 listener 的 onJbootStarted() 方法。

# 1 如何使用 main 文件启动一个应用服务器？

如果你会使用，可以直接跳过这节。

pom.xml:  
```xml
...
<dependency>
    <groupId>org.eclipse.jetty</groupId>
    <artifactId>jetty-server</artifactId>
    <version>9.4.8.v20171121</version>
</dependency>

<dependency>
    <groupId>org.eclipse.jetty</groupId>
    <artifactId>jetty-servlet</artifactId>
    <version>9.4.8.v20171121</version>
</dependency>
```

SimpleServer.java:  
```java
public class SimpleServer {

    public static void main(String[] args) throws Exception {
        InetSocketAddress address = new InetSocketAddress("0.0.0.0", 8081);
        Server server = new Server(address);
        ResourceHandler handler = new ResourceHandler();
        handler.setDirectoriesListed(true);
        handler.setResourceBase("/Users/liwenguang/Downloads");
        server.setHandler(handler);
        server.start();
    }
}
```

参考资料：http://blog.csdn.net/kiterunner/article/details/51695293

# 2 Jboot 启动精简版

Jboot.java 主文件：  
```java
public class Jboot {

    private JbootServer jbootServer;

    public void start() {
        ensureServerCreated();
        if (!startServer()) {
            System.err.println("jboot start fail!!!");
            return;
        }
    }

    private void ensureServerCreated() {
        if (jbootServer == null) {
            JbootServerFactory factory = JbootServerFactory.me();
            jbootServer = factory.buildServer();
        }
    }

    private boolean startServer() {
        return jbootServer.start();
    }

    public static void main(String[] args) {
        new Jboot().start();
    }
}
```

JbootServer 抽象类，方便各种应用服务器的工厂创建，其中作者只编写了 undertow 和 jetty 的实现。（可知作者对 tomcat 不大喜欢）：  
```java
public abstract class JbootServer {
    public abstract boolean start();
    public abstract boolean restart();
    public abstract boolean stop();
}
``` 
JbootServerFactory 工厂类：
```java
public class JbootServerFactory {

    private static JbootServerFactory me = new JbootServerFactory();
    public static JbootServerFactory me() {
        return me;
    }
    public JbootServer buildServer() {
        // switch 
        return new JettyServer();
    }
}
```

接着是应用服务器的配置文件， JbootServerConfig：  
```java
public class JbootServerConfig {

    public static final String TYPE_UNDERTOW = "undertow";
    public static final String TYPE_TOMCAT = "tomcat";
    public static final String TYPE_JETTY = "jetty";

    private String type = TYPE_UNDERTOW;
    private String host = "0.0.0.0";
    private int port = 8080;
    private String contextPath = "/";

    // set/get 省略
}
```

最后是实现的 Jetty 应用服务器， JettyServer：
```java
public class JettyServer extends JbootServer {

    private static Log log = Log.getLog(JettyServer.class);

    private JbootServerConfig config;
    // private JbootWebConfig webConfig;

    private Server jettyServer;
    private ServletContextHandler handler;

    public JettyServer() {
        config = new JbootServerConfig();
        // webConfig = Jboot.config(JbootWebConfig.class);
    }

    @Override
    public boolean start() {
        try {
            initJettyServer();
            // JbootAppListenerManager.me().onAppStartBefore(this);
            jettyServer.start();
        } catch (Throwable ex) {
            log.error(ex.toString(), ex);
            stop();
            return false;
        }
        return true;
    }

    private void initJettyServer() {
        InetSocketAddress address = new InetSocketAddress(config.getHost(), config.getPort());
        jettyServer = new Server(address);

        handler = new ServletContextHandler();
        handler.setContextPath(config.getContextPath());
        handler.setClassLoader(new JbootServerClassloader(JettyServer.class.getClassLoader()));
        handler.setResourceBase(getRootClassPath());
        /*
        增加 shiro 全局过滤器
        JbootShiroConfig shiroConfig = Jboot.config(JbootShiroConfig.class);
        if (shiroConfig.isConfigOK()) {
            handler.addEventListener(new EnvironmentLoaderListener());
            handler.addFilter(ShiroFilter.class, "/*", EnumSet.of(DispatcherType.REQUEST));
        }
        */
        /*
        增加 Jfinal Handler，Jboot 基于 Jfinal
        //JFinal
        FilterHolder jfinalFilter = handler.addFilter(JFinalFilter.class, "/*", EnumSet.of(DispatcherType.REQUEST));
        jfinalFilter.setInitParameter("configClass", Jboot.me().getJbootConfig().getJfinalConfig());
        增加 Hystrix 监控 servlet
        JbootHystrixConfig hystrixConfig = Jboot.config(JbootHystrixConfig.class);
        if (StringUtils.isNotBlank(hystrixConfig.getUrl())) {
            handler.addServlet(HystrixMetricsStreamServlet.class, hystrixConfig.getUrl());
        }

        增加 metric 监控
        JbootMetricConfig metricsConfig = Jboot.config(JbootMetricConfig.class);
        if (StringUtils.isNotBlank(metricsConfig.getUrl())) {
            handler.addEventListener(new JbootMetricServletContextListener());
            handler.addEventListener(new JbootHealthCheckServletContextListener());
            handler.addServlet(AdminServlet.class, metricsConfig.getUrl());
        }
        最后增加 Jboot 本身的 servlet
        io.jboot.server.Servlets jbootServlets = new io.jboot.server.Servlets();
        ContextListeners listeners = new ContextListeners();
        JbootAppListenerManager.me().onJbootDeploy(jbootServlets, listeners);
        for (Map.Entry<String, io.jboot.server.Servlets.ServletInfo> entry : jbootServlets.getServlets().entrySet()) {
            for (String path : entry.getValue().getUrlMapping()) {
                handler.addServlet(entry.getValue().getServletClass(), path);
            }
        }
        事件监听
        for (Class<? extends ServletContextListener> listenerClass : listeners.getListeners()) {
            handler.addEventListener(ClassKits.newInstance(listenerClass));
        }

        */
        jettyServer.setHandler(handler);
    }

    private static String getRootClassPath() {
        String path = null;
        try {
            path = JettyServer.class.getClassLoader().getResource("").toURI().getPath();
            return new File(path).getAbsolutePath();
        } catch (URISyntaxException e) {
            e.printStackTrace();
        }
        return path;
    }

    @Override
    public boolean restart() {
        stop();
        start();
        return true;
    }

    @Override
    public boolean stop() {
        try {
            jettyServer.stop();
            return true;
        } catch (Exception ex) {
            log.error(ex.toString(), ex);
        }
        return false;
    }
}
```

最后是自定义 ClassLoader，JbootServerClassLoader：  
```java
public class JbootServerClassloader extends ClassLoader {

    public JbootServerClassloader(ClassLoader parent) {
        super(parent);
    }


    @Override
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        return loadClass(name, false);
    }
}
```
自定义 ClassLoader 在应用服务器中都会自定义，用于文件的隔离和热更新。  
  
目录结构如下：  
![][1]

# 3 启动到底启动了什么

## 1. 参数解析

类似 JVM options 的 -Dxxx=xxx 参数的作用，用于全局访问，Jboot 将启动参数使用 Jboot.setBootArg() 放在了一个 Map 中，你可以使用 Jboot.getBootArg() 获取。  
Jboot.java  
```java
private static void parseArgs(String[] args) {
        if (args == null || args.length == 0) {
            return;
        }

        for (String arg : args) {
            int indexOf = arg.indexOf("=");
            if (arg.startsWith("--") && indexOf > 0) {
                String key = arg.substring(2, indexOf);
                String value = arg.substring(indexOf + 1);
                setBootArg(key, value);
            }
        }
    }
```

## 2. 判断启动模式

默认为 dev 模式，查看 JbootConfig.java 文件可知，但是我们可能为想，我们怎么才能设置启动模式呢？  
  
没错，使用启动参数！请看 JbootConfigManager 文件，该文件是用于读取配置文件，你可能会想，为什么配置文件都加了 `@PropertyConfig(prefix = "")` 这样的注解呢，其实，这是作者为了方便 JavaBean 与 参数 进行转换。直接上代码：  
  
第一种：启动参数，如下图：  
![][2]  
我们配置了两个参数（对照 JbootConfig 你就知道，只有 mode 有 set 方法，而 version 是只有 get 方法的）。  
最后启动 debug 的时候你就会发现 Jboot.isDevMode() 方法返回 false 而不是默认的 true。 
>有很多地方判断了，如果是 dev 模式，则会打印一些参数，例如 JbootEventManager 方法。

第二种：使用 `Jboot.setBootArg("jboot.mode", "test");` 这种，从前面的 **参数解析** 一节我们已经知道，其实启动参数底层使用的就是 setBootArg 方法。
>测试类中很多使用了这种方法，例如 `DubboClientZookeeperDemo`。

如果是 dev 模式，就会定时 3 秒扫描应用服务器文件夹，但是作者注释了，这里不懂作者的意思。  
AutoDeployManager.java  
```java
public void run() {

        File file = new File(PathKit.getRootClassPath());
        JbootFileScanner scanner = new JbootFileScanner(file.getAbsolutePath(), 3) {
            @Override
            public void onChange(String action, String file) {
                try {
//                    System.err.println("file changes : " + file);
//                    Jboot.me().getServer().restart();
//                    JbootServerFactory.me().buildServer().start();
//                    System.err.println("Loading complete.");
                } catch (Exception e) {
                    System.err.println("Error reconfiguring/restarting webapp after change in watched files");
                    LogKit.error(e.getMessage(), e);
                }
            }
        };

        scanner.start();
    }
```

## 3. 回调所有 JbootAppListener 实现类的 onJbootStarted()方法

在 Jboot 启动的最后一步，实例化了 JbootAppListenerManager 类：
```java
 private JbootAppListenerManager() {
    // 扫描获取所有 JbootAppListener 的子类
    List<Class<JbootAppListener>> allListeners = ClassScanner.scanSubClass(JbootAppListener.class, true);
    if (allListeners == null || allListeners.size() == 0) {
        return;
    }
    // 去除 JbootAppListenerManager 本身
    for (Class<? extends JbootAppListener> clazz : allListeners) {
        if (JbootAppListenerManager.class == clazz || JbootAppListenerBase.class == clazz) {
            continue;
        }
        // 实例化
        JbootAppListener listener = ClassKits.newInstance(clazz, false);
        if (listener != null) {
            listeners.add(listener);
        }
    }
}

@Override
public void onJbootStarted() {
    for (JbootAppListener listener : listeners) {
        try {
            listener.onJbootStarted();
        } catch (Throwable ex) {
            log.error(ex.toString(), ex);
        }
    }
}
```

并通过 `JbootAppListenerManager.me().onJbootStarted();` 回调了 `onJbootStarted()` 方法，来调用用户的逻辑。

# 4 其它

1. 从一些 Manager 方法看的出作者习惯通过构造方法进行一些必要的初始化，我以前看的 《架构探险——从零开始写Java Web框架》 则喜欢用静态块进行初始化。  
2. 启动的一些细节需要大家去 debug 一步一步看，看懂了也是很高兴的，毕竟作者也是大牛，更近了一步。  
3. 作者代码习惯方法名由于注释。说实话初看有点不习惯，因为习惯看注释了，但是作者方法名真的能让你可以不用注释（除却一些必要方法作者加了注释）。
4. jbootfly 是入门，不要想直接看源码，欲速则不达。
5. 你要懂 jfinal 的知识，至少看过 jfinal 文档，写过 jfinal 经典的 blog 项目。

## 如果有错误，请指出，谢谢，共勉。

[1]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/others/jboot_start_server_demo.png
[2]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/others/jboot_start_server_args.png
