---
title: spring 源码解析 1.概述
date: 2017-04-30 11:45:41
tags:
- spring container
categories:
- spring
- code
---
## 一、干了这碗鸡汤
阅读源码比较晦涩。读不懂，读了就忘，abandon 肯定会发生。届时请自行喝鸡汤。
- 阅读前 - 必须要形成自己的阅读源码的动机，意识上做出决定。彻底拥抱spring，在所有 java 项目中使用 spring famework
- 阅读过程中 - 做有效阅读，一面要揣测作者意图；理解方法作用。一面分析设计模式；设想扩展场景。
- 阅读后 - 提炼精髓，提高编程修养，提升设计能力。
### 1.阅读目标
- 了解使用 spring 提供的绝大部分功能，工具类。不重复造轮子。达到脱离 ApplicationContext 独立使用 spring 模块。
- 扩展，定制 ApplicationContext。
### 2.阅读方法
- 正确的阅读方式：
    - 自下而上：根据包来绘制类图，从父类读到子类
    - 自上而下：根据方法绘制时序图，从子类读到父类
- ~~错误的阅读方式：debug跟踪~~
- 模块化集中阅读。

## 二、概述
基于 spring 5.0.0.M5 版本阅读。

阅读本系列博客，默认你已经拥有：
- java 基础
- uml 类图、时序图阅读能力
- [spring 5.0.0.M5参考文档](http://docs.spring.io/spring/docs/5.0.0.M5/spring-framework-reference/htmlsingle/#core-convert-ConversionService-API)阅读经验

### 1.模块
![](/assets/img/spring/springOverview.png)

### 2.pom
![](/assets/img/spring/springContainerPom.png)

|ArtifactId|功能|
|:-|:-|
|spring-context|应用程序，包括context组装，JMX，Jndi，缓存，定时调度，rmi，validation|
|spring-core|核心实用程序，asm，cglib，io，env，线程池|
|spring-beans|Beans支持，bean属性，BeanFactory，包括Groovy|
|spring-expression|Spring Expression Language (SpEL)|
|spring-aop|基于代理的AOP支持|

### 3.ApplicationContext
![](/assets/img/spring/ApplicationContextUML.png)
ApplicationContext提供的功能点：
1. Resource、ResourceLoader、ResourcePatternResolver - 提供统一的 io 操作
1. Enviroment - 提供各种配置源的变量统一读取
1. *BeanFactory - 提供层级化，可罗列的BeanFactory获取Bean
1. ApplicationEventPublish - 提供发布订阅的监听者模式实现
1. MessageSource - 提供i18n

本系列博客将按照 资源，加载环境，读取bean配置，生成Factory，处理事件，实例化Bean，aop的顺序(core,bean,context,aop)的顺序来阅读讲解