---
title: spring 源码解析 BeanPostProcessor
date: 2017-07-21 17:35:29
tags:
- spring container
categories:
- spring
- code
---

### 1. BeanPostProcessor
#### 1.1 FactoryBean构造Bean时，调用BeanPostProcessor.postProcessAfterInitialization
AbstractBeanFactory.doGetBean ->
AbstractBeanFactory.getObjectForBeanInstance ->
FactoryBeanRegistrySupport.getObjectFromFactoryBean ->
FactoryBeanRegistrySupport.postProcessObjectFromFactoryBean
@Override AbstractAutowireCapableBeanFactory.postProcessObjectFromFactoryBean ->
AbstractAutowireCapableBeanFactory.applyBeanPostProcessorsAfterInitialization
beanProcessor.