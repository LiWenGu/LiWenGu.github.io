---
title: 在JDK8中的JVM
date: 2019-12-11 09:12:00
updated: 2019-12-11 09:12:00
tags:
  - 锁库存
  - 超卖
categories: 
  - 实践
  - 技术小结
  - 转载
comments: true
permalink: tech/jvm_in_jdk8.html 
---

# 0. 背景


<!--more-->

1. MetaSpace 存储了哪些？

类、符号、基本数组、方法以及其字节码、方法计数器、常量池和CP缓存、注解，字段的名称、类型、偏移量，这些都可以视为类的元数据。