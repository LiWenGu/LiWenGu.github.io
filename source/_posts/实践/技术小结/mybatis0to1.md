---
title: 《MyBatis 从入门到精通》总结
date: 2017-12-18 22:10:00
updated: 2017-12-18 22:10:00
tags:
  - mybatis
categories: 
  - 实践
  - 技术小结
comments: true
permalink: tech/mybatis0to1.html 
---

笔记总结+源码：https://github.com/LiWenGu/MySourceCode/tree/master/mybatis0to1  
在前七章都打了对应的标签，可以通过 git checkout来。  
![][1]  
总结就是，将 sql 语句从代码中抽离出来，通过 xml 的配置来实现单表、多表的映射，最后通过动态代理来执行方法，很强的解耦性。  
把 SQL 放在了 XML 中，然后用一些判断来实现动态 SQL ，最后通过 SqlSession 、SqlSessionFacotry 的生命周期来绑定一级、二级缓存。  
不学之前感觉很神奇，学完之后也就那么回事，不过还是要多学学基础，例如读取配置、缓存、一级动态代理等。

[1]: http://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/mybatis0to1/summary.png
