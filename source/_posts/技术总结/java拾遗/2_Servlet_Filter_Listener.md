---
title: Servlet_Filter_Listener
date: 2018-04-12 23:00:00
updated: 2018-04-13 01:00:00
comments: true
tags:
  - java
categories: 
  - [技术总结, java拾遗]
comments: true
permalink: A2B_Java/2_Servlet_Filter_Listener.html    
---

# 1. web.xml 的加载顺序

ServletContext -> context-param -> listener -> filter -> servlet。  
前面的两个用于在 web.xml 设置变量，重要的是后面三个，即 Listener、Filter、Servlet，为 javaweb 的三大组件。

# 2. Listener

1. ServletContext 的监听器：ServletContextListener
2. HttpSession 的监听器：HttpSessionListener
3. ServletRequest 的监听器：ServletRequestListener

>其中 Spring MVC 中 web.xml 常见配置的  
>`org.springframework.web.context.request.RequestContextListener` 就是实现了 `ServletRequestListener` 用于监听每次请求。  
>`org.springframework.web.context.ContextLoaderListener` 就是实现了 `ServletContextListener` 用于监听 ServletContext 的创建（即容器的启动），进而初始化 Spring 容器。

# 3. Filter

1. Filter 和 Servlet 类似也有三个生命周期方法，init、doFilter、destroy 三个。  
2. Filter 是对多个请求进行拦截处理放行的，可以有多个 Filter。  
3. 如果某个请求匹配到了多个过滤器，则根据 filter-mapping 的顺序进行过滤

> 其中 Spring MVC 中 web.xml 常见配置的 `org.springframework.web.filter.CharacterEncodingFilter` 就是实现了 `Filter` 用于每次过滤的字符编码设置

# 4. Servlet

1. Servlet 用于处理客户端匹配的请求，获取请求，处理请求，返回响应。
2. 三个生命周期方法，init、service、destroy。
3. `<load-on-startup>` 如果是非负整数或零时，Servlet 容器先加载数值小的 servlet；如果是负数则 Servlet 容器将在首次访问时加载（懒汉模式）。
4. Servlet 属于单例，多个请求可能会请求同一个 Servlet，即一个类只有一个对象，类由我们编写并写入 web.xml 配置中，但对象由容器创建，由容器调用相应的方法。

> 其中 Spring MVC 中 web.xml 常见配置的 `org.springframework.web.servlet.DispatcherServlet` 就是继承了 `HttpServlet` 用于每次用户的请求。

# 5. Spring 容器启动

下面是 web.xml 的常见配置：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns="http://java.sun.com/xml/ns/javaee"
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
         id="WebApp_ID" version="2.5">

    <display-name>Archetype Created Web Application</display-name>

    <filter>
        <filter-name>characterEncodingFilter</filter-name>
        <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
        <init-param>
            <param-name>encoding</param-name>
            <param-value>UTF-8</param-value>
        </init-param>
        <init-param>
            <param-name>forceEncoding</param-name>
            <param-value>true</param-value>
        </init-param>
    </filter>
    <filter-mapping>
        <filter-name>characterEncodingFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>


    <listener>
        <listener-class>org.springframework.web.context.request.RequestContextListener</listener-class>
    </listener>

    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>
            classpath:applicationContext.xml
        </param-value>
    </context-param>

    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>*.do</url-pattern>
    </servlet-mapping>

</web-app>

```
1. 创建 ServletContext 上下文并将 `context-param` 属性写入此上下文。
2. 执行 `ContextLoaderListener` 将 Spring 的上下文 `WebApplicationContext` 写入 ServletContext 用于 Servlet 容器共享，接着读取 context-param 节点（已经被加载了）的 `contextConfigLocation` 解析 xml 并创建 Spring 容器和初始化。
3. 执行 `CharacterEncodingFilter` 过滤器设置相应设置，用于以后的每次请求。
4. 执行 `DispatcherServlet`  用于 Servlet 的创建（根据 load-on-startup）。

# 6. 参考

三大组件参考：https://blog.csdn.net/xiaojie119120/article/details/73274759  
Spring 容器启动参考：https://blog.csdn.net/u013510838/article/details/75066884

