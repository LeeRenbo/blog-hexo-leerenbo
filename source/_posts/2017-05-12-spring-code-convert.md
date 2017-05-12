---
title: spring 源码解析 3.类型转换Convert
date: 2017-05-12 00:32:09
tags:
- spring container
categories:
- spring
- code
---
## 1.概述
Spring 3引入了一个提供通用类型转换系统的core.convert包。 该系统定义了一个SPI来实现类型转换逻辑，以及一个在运行时执行类型转换的API。 在Spring容器中，该系统可以用作PropertyEditor的替代方法，将外部化的bean属性值字符串转换为必需的属性类型。 公共API也可以在需要类型转换的应用程序的任何地方使用。

## 2.类图
![](/assets/img/spring/springConvert.png)

## 3. Service
### 3.1 接口
##### 3.1.1 [ConverterRegistry]()


## 4.convertor