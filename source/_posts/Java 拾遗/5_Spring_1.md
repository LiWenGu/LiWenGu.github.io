---
title: Spring_1
date: 2018-04-13 18:00:00
updated: 2018-04-13 19:00:00
comments: true
categories: 
- Java 拾遗
permalink: A2B_Java/5_Spring_1.html    
---

# 1. Spring 的核心

## 1. IOC/DI（控制反转/依赖注入）

注入方式： set方法、构造器、工厂方法

## 2. AOP（面向切面编程）

使用的是动态代理，通过 JDK（接口实现） 或 CGLib 字节码工具包的继承来实现动态代理

# 2. Bean 的生命周期

## 1. 实例化 Bean

对于 BeanFactory 容器，是懒实例；而 ApplicationiContext 容器则是容器启动时就会实例化所有的 Bean

## 2. 设置对象属性（依赖注入）

Spring 通过 BeanDefinition 中的信息进行依赖注入

## 3. 注入 Aware 接口

Spring 检查独享是否实现了 XXXAware 接口，并调用相应的方法

## 4. BeanPostProcessor

通过 BeanPostProcessor 接口的 postProcessBeforeInitialzation 方法，在 Bean 初始化前做调用，也称为前置处理（Aware 接口就是在这里完成注入的）  
postProcessAfterInitialzation 方法在 Bean 初始化之后进行调用，也称为后置处理

## 5. InitialzingBean 与 init-method

Initializing 只有一个 afterPropertiesSet() 方法，在 Bean 初始化前做调用，和前置处理唯一区别是不会对 Bean 本身处理（Bean 不被参数传递）  
同样功能在配置文件为 init-method

## 6. DisposableBean 和 destroy-method

Bean 被清理之前调用 DisposableBean 接口的 destroy() 方法  
同样功能在配置文件为 destroy-method

