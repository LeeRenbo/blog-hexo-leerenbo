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
调用InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation


#### 2.2 createBean时，doCreateBean之前，调用InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation之后bean不为空之后，调用BeanPostProcessor.postProcessAfterInitialization
AbstractBeanFactory.doGetBean ->
DefaultSingletonBeanRegistry.getSingleton ->
AbstractAutowireCapableBeanFactory.createBean ->
AbstractAutowireCapableBeanFactory.resolveBeforeInstantiation ->
AbstractAutowireCapableBeanFactory.applyBeanPostProcessorsAfterInitialization ->
调用BeanPostProcessor.postProcessAfterInitialization


### 3. MergedBeanDefinitionPostProcessor
#### 3.1 doCreateBean时，createBeanInstance之后，earlySingletonExposure之前。调用MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition进行注释融合
AbstractBeanFactory.doGetBean ->
DefaultSingletonBeanRegistry.getSingleton ->
AbstractAutowireCapableBeanFactory.createBean ->
AbstractAutowireCapableBeanFactory.doCreateBean ->
AbstractAutowireCapableBeanFactory.applyMergedBeanDefinitionPostProcessors ->
MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition

### 4. SmartInstantiationAwareBeanPostProcessor
#### 4.1 循环引用时，ObjectFactory.getObject获取对象时，调用
AbstractBeanFactory.doGetBean ->
DefaultSingletonBeanRegistry.getSingleton ->
AbstractAutowireCapableBeanFactory.createBean ->
AbstractAutowireCapableBeanFactory.doCreateBean ->
AbstractAutowireCapableBeanFactory.addSingletonFactory ->
ObjectFactory.getObject ->
AbstractAutowireCapableBeanFactory.getEarlyBeanReference ->
SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference


