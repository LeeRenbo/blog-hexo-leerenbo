---
title: spring 源码解析 4.类型转换Convert
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

## 3.convertor
### 3.1 接口
##### 3.1.1 [ConditionalConverter](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/convert/converter/ConditionalConverter.java)
```java
/**
 * 允许 {@link Converter}，{@link GenericConverter}或{@link ConverterFactory}根据{@code source}和{@code target} {@link TypeDescriptor}的属性有条件地执行。
 * 通常用于根据字段或类级特征（如注释或方法）的存在选择性地匹配自定义转换逻辑。 例如，当从String字段转换为Date字段时，如果目标字段也已使用{@code @DateTimeFormat}注释，则实现可能返回{@code true}。
 * 另一个例子是，当从String字段转换为{@code Account}字段时，如果目标Account类定义了{@code public static findAccount（String）}方法，那么实现可能返回{@code true}。
 */
public interface ConditionalConverter {
	/**
	 * 当前{@code sourceType}到{@code targetType}的转换是否应当选择被选中？
	 */
	boolean matches(TypeDescriptor sourceType, TypeDescriptor targetType);
}
```

##### 3.1.2 [Converter<S, T>](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/convert/converter/Converter.java)
```java
/**
 * 转换器将类型为{@code S}的 source 对象转换为类型为{@code T}的 target。
 * 此接口的实现是线程安全的，可以共享。
 * 实现可以同时实现{@link ConditionalConverter}。
 */
@FunctionalInterface
public interface Converter<S, T> {
	/**
	 * 将类型为{@code S}的 source 对象转换为类型{@code T} target。
	 */
	T convert(S source);
}
```

##### 3.1.3 [ConverterFactory<S, R>](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/convert/converter/ConverterFactory.java)
```java
/**
 * 一个可以将对象从 S 转换成 R 的子类型的“范围”转换器的工厂。
 * 实现可以同时实现{@link ConditionalConverter}。
 */
public interface ConverterFactory<S, R> {
	/**
	 * 获取转换器从S转换为目标类型T，其中T也是R的实例。
	 */
	<T extends R> Converter<S, T> getConverter(Class<T> targetType);
}
```

##### 3.1.4 [GenericConverter](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/convert/converter/GenericConverter.java)
```java
/**
 * 用于在两种或多种类型之间转换的通用转换器接口。
 * 这是转换器SPI接口中最灵活的，也是最复杂的。 它是灵活的，因为GenericConverter可以支持在多个 source/target 类型对之间进行转换（参见{@link #getConvertibleTypes()}）。此外，GenericConverter实现在类型转换期间可以访问 source / target {@link TypeDescriptor field context} 这可以解决 source 和 target 字段元数据，如 annotations 和泛型信息，可用于影响转换逻辑。
 * 当简单的{@link Converter}或{@link ConverterFactory}接口就足够时，通常不会使用该接口。
 * 实现可以同时实现{@link ConditionalConverter}。
 */
public interface GenericConverter {

	/**
	 * 返回 source 和 target 的类型对，用于此转换器在类型之间转换。
	 * 每个条目都是可转换的源到目标类型对。
	 * 对于{@link ConditionalConverter conditional converters}，此方法可能会返回{@code null}以指示应考虑所有源对目标对。
	 */
	Set<ConvertiblePair> getConvertibleTypes();

	/**
	 * 将源对象转换为{@code TypeDescriptor}描述的targetType。
	 */
	Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType);

	final class ConvertiblePair {
		private final Class<?> sourceType;
		private final Class<?> targetType;
	}
}
```

##### 3.1.5 [ConverterAdapter](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/convert/support/GenericConversionService.java)
```java
/**
 * 将 {@link Converter} 适配到 {@link GenericConverter}。
 */
private final class ConverterAdapter implements ConditionalGenericConverter {
    private final Converter<Object, Object> converter;
    private final ConvertiblePair typeInfo;
    private final ResolvableType targetType;
}
```

##### 3.1.6 [ConverterFactoryAdapter](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/convert/support/GenericConversionService.java)
```java
/**
 * 将 {@link ConverterFactory} 适配到 {@link GenericConverter} 。
 */
private final class ConverterFactoryAdapter implements ConditionalGenericConverter {
    private final ConverterFactory<Object, Object> converterFactory;
    private final ConvertiblePair typeInfo;
}
```

##### 3.1.7 [Converters](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/convert/support/GenericConversionService.java)
```java
/**
 * 管理注册服务的所有转换器。
 */
private static class Converters {
    private final Set<GenericConverter> globalConverters = new LinkedHashSet<>();
    private final Map<ConvertiblePair, ConvertersForPair> converters = new LinkedHashMap<>(36);
}
```

## 4. Service
### 4.1 接口
##### 4.1.1 [ConversionService](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/convert/ConversionService.java)
```java
/**
 * 用于类型转换的服务接口。这是转换系统的切入点。
 * 调用 {@link #convert(Object, Class)} 以使用此系统执行线程安全类型转换。
 */
public interface ConversionService {

	/**
	 * 如果 {@code sourceType} 的对象可以转换为 {@code targetType} ，返回 {@code true}。
	 * 如果此方法返回{@code true}，则表示 {@link #convert(Object, Class)} 能够将 {@code sourceType} 的实例转换为 {@code targetType}。
	 * collections，arrays 和 map 类型的特别说明：
	 * 对于collections，arrays 和 map类型之间的转换，即使转换调用此方法将返回{@code true}，如果基础元素不可转换，仍然可能会生成{@link ConversionException}。 在使用 collections 和 maps 时，调用方应该处理这种特殊情况。
	 */
	boolean canConvert(Class<?> sourceType, Class<?> targetType);

	/**
	 * 如果{@code sourceType}的对象可以转换为{@code targetType}，返回{@code true}。
	 * TypeDescriptors提供了关于 source 和 target 转换发生的位置的附加上下文，通常是对象字段或属性位置。
	 * 如果此方法返回{@code true}，则表示 {@link #convert(Object, TypeDescriptor, TypeDescriptor)} 能够将 {@code sourceType} 的实例转换为{@code targetType}。
	 * collections，arrays 和 map 类型的特别说明：
	 * 对于collections，arrays 和 map类型之间的转换，即使转换调用此方法将返回{@code true}，如果基础元素不可转换，仍然可能会生成{@link ConversionException}。 在使用 collections 和 maps 时，调用方应该处理这种特殊情况。
	 */
	boolean canConvert(TypeDescriptor sourceType, TypeDescriptor targetType);

	/**
	 * 将给定的{@code source}转换为指定的{@code targetType}。
	 */
	<T> T convert(Object source, Class<T> targetType);

	/**
	 * 将给定的{@code source}转换为指定的{@code targetType}。
	 * TypeDescriptors提供了关于 source 和 target 转换发生的位置的附加上下文，通常是对象字段或属性位置。
	 */
	Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType);
}
```
##### 4.1.2 [ConverterRegistry](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/convert/converter/ConverterRegistry.java)
```java
/**
 * 用于注册类型转换系统的转换器
 */
public interface ConverterRegistry {

	/**
	 * 将一个普通转换器添加到此注册表。
	 * 可转换的 source/target 类型对，派生自转 Converter 的参数化类型。
	 */
	void addConverter(Converter<?, ?> converter);

	/**
	 * 将一个普通转换器添加到此注册表。
	 * 明确指定可转换 source/target 类型对。
	 * 允许将转换器重用于多个不同的对，而无需为每对创建一个Converter类。
	 */
	<S, T> void addConverter(Class<S> sourceType, Class<T> targetType, Converter<? super S, ? extends T> converter);

	/**
	 * 将通用转换器添加到此注册表。
	 */
	void addConverter(GenericConverter converter);

	/**
	 * 向此注册表添加一个远程转换器工厂。
	 * 可转换 source/target 类型对，派生自ConverterFactory的参数化类型。
	 */
	void addConverterFactory(ConverterFactory<?, ?> factory);

	/**
	 * 将任何从{@code sourceType}到{@code targetType}转换器移除
	 */
	void removeConvertible(Class<?> sourceType, Class<?> targetType);
}
```

##### 4.1.3 [ConfigurableConversionService](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/convert/support/ConfigurableConversionService.java)
```java
/**
 * 配置接口大多数（如果不是全部）由{@link ConversionService}类型实现。 整合{@link ConversionService}公开的只读操作以及{@link ConverterRegistry}的变异操作，以便于方便的点对点添加和删除{@link org.springframework.core.convert.converter.Converter Converters}。 后者在ApplicationContext引导代码中的{@link org.springframework.core.env.ConfigurableEnvironment ConfigurableEnvironment}实例时特别有用。
 */
public interface ConfigurableConversionService extends ConversionService, ConverterRegistry {
}
```

### 4.2 实现
##### 4.2.1[GenericConversionService](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/convert/support/GenericConversionService.java)
```java
/**
 * 基于 {@link ConversionService} 的实现，适用于大多数环境。
 * 通过 {@link ConfigurableConversionService} 接口间接实现 {@link ConverterRegistry} 作为注册API。
 */
public class GenericConversionService implements ConfigurableConversionService {
	/**
	 * 不需要转换时使用的一般NO-OP转换器。
	 */
	private static final GenericConverter NO_OP_CONVERTER = new NoOpConverter("NO_OP");

	/**
	 * 当没有转换器可用时用作缓存条目。
	 * 此转换器从不返回。
	 */
	private static final GenericConverter NO_MATCH = new NoOpConverter("NO_MATCH");

	private final Converters converters = new Converters();

	private final Map<ConverterCacheKey, GenericConverter> converterCache = new ConcurrentReferenceHashMap<>(64);
}

```

