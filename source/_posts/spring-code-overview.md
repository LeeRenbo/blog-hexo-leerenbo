---
title: spring container 源码概述
date: 2017-04-30 11:45:41
tags:
- spring container
categories:
- spring
- code
---

## 1.模块
![](/assets/img/spring/spring-overview.png)

## 2.pom
![](/assets/img/spring/springContainerPom.png)

ArtifactId|功能
-|-
spring-context|应用程序上下文运行时，包括调度和远程抽象
spring-core|核心实用程序，许多其他Spring模块使用
spring-beans|Beans支持，包括Groovy
spring-expression|Spring Expression Language (SpEL)
spring-aop|基于代理的AOP支持

## 3.ApplicationContext
![](/assets/img/spring/ApplicationContextUML.png)