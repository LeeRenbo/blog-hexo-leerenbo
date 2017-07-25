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
BeanPostProcessor.postProcessAfterInitialization



### 2. 调用InstantiationAwareBeanPostProcessor
#### 2.1 createBean时，doCreateBean之前，调用InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation
AbstractBeanFactory.doGetBean ->
DefaultSingletonBeanRegistry.getSingleton ->
AbstractAutowireCapableBeanFactory.createBean ->
AbstractAutowireCapableBeanFactory.resolveBeforeInstantiation ->
AbstractAutowireCapableBeanFactory.applyBeanPostProcessorsBeforeInstantiation ->
InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation


#### 2.2 createBean时，doCreateBean之前，调用InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation之后bean不为空之后，调用BeanPostProcessor.postProcessAfterInitialization
AbstractBeanFactory.doGetBean ->
DefaultSingletonBeanRegistry.getSingleton ->
AbstractAutowireCapableBeanFactory.createBean ->
AbstractAutowireCapableBeanFactory.resolveBeforeInstantiation ->
AbstractAutowireCapableBeanFactory.applyBeanPostProcessorsAfterInitialization ->
BeanPostProcessor.postProcessAfterInitialization


### 3. MergedBeanDefinitionPostProcessor
#### 3.1 doCreateBean时，createBeanInstance之后，earlySingletonExposure之前。调用MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition 进行注释融合
AbstractBeanFactory.doGetBean ->
DefaultSingletonBeanRegistry.getSingleton ->
AbstractAutowireCapableBeanFactory.createBean ->
AbstractAutowireCapableBeanFactory.doCreateBean ->
AbstractAutowireCapableBeanFactory.applyMergedBeanDefinitionPostProcessors ->
MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition

### 4. SmartInstantiationAwareBeanPostProcessor
#### 4.1 循环引用时，ObjectFactory.getObject获取对象时，调用SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference，用于提前返回引用，支持单例的循环引用
AbstractBeanFactory.doGetBean ->
DefaultSingletonBeanRegistry.getSingleton ->
AbstractAutowireCapableBeanFactory.createBean ->
AbstractAutowireCapableBeanFactory.doCreateBean ->
AbstractAutowireCapableBeanFactory.addSingletonFactory ->
ObjectFactory.getObject ->
AbstractAutowireCapableBeanFactory.getEarlyBeanReference ->
SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference

### 5. InstantiationAwareBeanPostProcessor
#### 5.1 populateBean时，在自动装配之前，调用InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation，用于字段注入，可停止属性填充
AbstractBeanFactory.doGetBean ->
DefaultSingletonBeanRegistry.getSingleton ->
AbstractAutowireCapableBeanFactory.createBean ->
AbstractAutowireCapableBeanFactory.doCreateBean ->
AbstractAutowireCapableBeanFactory.populateBean ->
InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation


### 6.InstantiationAwareBeanPostProcessor
#### 6.1 populateBean时，在装配之后，调用InstantiationAwareBeanPostProcessor.postProcessPropertyValues，用于
AbstractBeanFactory.doGetBean ->
DefaultSingletonBeanRegistry.getSingleton ->
AbstractAutowireCapableBeanFactory.createBean ->
AbstractAutowireCapableBeanFactory.doCreateBean ->
AbstractAutowireCapableBeanFactory.populateBean ->
InstantiationAwareBeanPostProcessor.postProcessPropertyValues


### 7 BeanPostProcessor
#### 7.1 initializeBean初始化时，invokeInitMethods之前。调用BeanPostProcessor.postProcessBeforeInitialization
AbstractBeanFactory.doGetBean ->
DefaultSingletonBeanRegistry.getSingleton ->
AbstractAutowireCapableBeanFactory.createBean ->
AbstractAutowireCapableBeanFactory.doCreateBean ->
AbstractAutowireCapableBeanFactory.initializeBean ->
AbstractAutowireCapableBeanFactory.applyBeanPostProcessorsBeforeInitialization ->
BeanPostProcessor.postProcessBeforeInitialization

#### 7.2 initializeBean初始化时，invokeInitMethods之后。调用BeanPostProcessor.postProcessAfterInitialization
AbstractBeanFactory.doGetBean ->
DefaultSingletonBeanRegistry.getSingleton ->
AbstractAutowireCapableBeanFactory.createBean ->
AbstractAutowireCapableBeanFactory.doCreateBean ->
AbstractAutowireCapableBeanFactory.initializeBean ->
AbstractAutowireCapableBeanFactory.applyBeanPostProcessorsAfterInitialization ->
BeanPostProcessor.postProcessAfterInitialization

