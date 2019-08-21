---
title: Jboot_1
date: 2018-01-10 12:00:00
updated: 2018-01-12 18:37:00
tags:
  - jboot
categories: 
  - 实践
  - 技术小结
comments: true
permalink: tech/Jboot_1.html    
---

Jboot的入门demo —— JbooFly。

<!--more-->

---

# 1 项目组成

后端基础框架：Jboot 1.2.7  
前端基础框架：Fly Template 社区模版  
界面渲染框架：JFinal Template Engine

# 2 数据库

使用 Mysql，一共有七张表，暂时使用的只有六张表，即文章分类表、评论表、文章表、用户表、用户行为表、用户收藏表。用户消息表没有用上。

# 3 前端页面

 1. 社区首页：外层架子（_layout.html），分类导航（_navigation.html），置顶文章（_top_posts.html），内容列表（content()），右边四个小页面（_signin_panel/_recommend/_hot_posts/_links）
2. 个人页面：外层架子，左侧导航（_user_left_menu.html），我的主页（index.html），我的帖子（post.html/collection.html），基本设置（setting.html），我的消息（message.html），账号激活（activate.html）

# 4 定时任务

1. 文章浏览量，一分钟一次。使用了 `ConcurrentHashMap` + `AtomicLong` 的方式处理线程问题。最后更新缓存（默认缓存开启并有五种）：  
```java
/**
 * name: io.jboot.core.cache.JbootCacheConfig
 */
public static final String TYPE_EHCACHE = "ehcache";
    public static final String TYPE_REDIS = "redis";
    public static final String TYPE_EHREDIS = "ehredis";
    public static final String TYPE_NONE_CACHE = "none";
    public static final String TYPE_J2CACHE = "j2cache";


    private String type = TYPE_EHCACHE;
    ......
```
2. 文章评论量，和文章浏览量一样的业务逻辑。

# 5 Caffeine 缓存使用

1. 签到缓存：SigninManager，缓存用户以及签到时间的映射关系，过期时间为两小时。
2. 消息缓存：MessageManager，作者未开发完，猜测是缓存用户以及用户未读消息的映射关系。

# 6 拦截器

1. 全局强制拦截器 UserIntercepor ，每次请求服务，都会使用解密算法查看存储在 cookie 中的用户信息，并获取该用户的签到缓存以及消息缓存信息。
2. ajax api 强制拦截器 ApiNeedUser。
3. 页面强制拦截器 NeedUser。

# 7 本地事件

1. 用户注册事件监听 UserRegister ，作者未开发完，猜测应该是用户注册后发送邮件或者验证码，抑或是通知版主？

# 8 JFinal 

1. JFinal Template Engine，查看 JFinal 自定义指令文档即可。主要是 directive 下的文件。

# 9 目的

猜测应该是 JBoot 论坛。

