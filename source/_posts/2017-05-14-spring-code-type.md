---
title: spring 源码解析 3.类型体系
date: 2017-05-14 15:01:48
tags:
- spring container
categories:
- spring
- code
---
## 1.Java反射，类型
![](/assets/img/spring/javaType.png)

## 2.Spring类型解析
![](/assets/img/spring/springConvertType.png)

### 2.1[ResolvableType](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/ResolvableType.java)
```java
/**
 * 封装Java {@link java.lang.reflect.Type}，提供访问{@link #getSuperType() supertypes}，{@link #getInterfaces() interfaces} 和 {@link #getGeneric(int...) generic parameters} 以及最终{@link #resolve() resolve} 到 {@link java.lang.Class}的能力。
 * 可以从 {@link #forField(Field) fields} ，{@link #forMethodParameter(Method, int) method parameters} ，{@link #forMethodReturnType(Method) method returns} 或 {@link #forClass(Class) classes} 中获取 {@code ResolvableTypes}。
 * 这个类上的大多数方法都将返回{@link ResolvableType}，方便导航。 例如：
 * 
 * 
 * private HashMap<Integer, List<String>> myMap;
 *
 * public void example() {
 *     ResolvableType t = ResolvableType.forField(getClass().getDeclaredField("myMap"));
 *     t.getSuperType(); // AbstractMap<Integer, List<String>>
 *     t.asMap(); // Map<Integer, List<String>>
 *     t.getGeneric(0).resolve(); // Integer
 *     t.getGeneric(1).resolve(); // List
 *     t.getGeneric(1); // List<String>
 *     t.resolveGeneric(1, 0); // String
 * }
 */
@SuppressWarnings("serial")
public class ResolvableType implements Serializable {

	/**
     * 没有值可用时返回{@code ResolvableType}。 {@code NONE}优先于{@code null}，以便可以安全地链接多个方法调用。
	 */
	public static final ResolvableType NONE = new ResolvableType(null, null, null, 0);

	private static final ResolvableType[] EMPTY_TYPES_ARRAY = new ResolvableType[0];

	private static final ConcurrentReferenceHashMap<ResolvableType, ResolvableType> cache =
			new ConcurrentReferenceHashMap<>(256);

	/**
     * 正在管理的底层Java类型（只有{@code null}，转化为{@link #NONE}）。
	 */
	private final Type type;

	/**
     * 该类型的可选提供程序。
	 */
	private final TypeProvider typeProvider;

	/**
     * 如果没有解析器可用，则使用{@code VariableResolver}或{@code null}。
	 */
	private final VariableResolver variableResolver;

	/**
     * 数组的组件类型或{@code null}，如果该类型应被推导出来。
	 */
	private final ResolvableType componentType;

	/**
	 * Copy of the resolved value.
     * 解析值的复制
	 */
	private final Class<?> resolved;

	private final Integer hash;

	private ResolvableType superType;

	private ResolvableType[] interfaces;

	private ResolvableType[] generics;
}
```

### [TypeDescriptor](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/convert/TypeDescriptor.javaOrderService)

```java
/**
 * 关于要转换的类型的上下文。
 */
@SuppressWarnings("serial")
public class TypeDescriptor implements Serializable {

	static final Annotation[] EMPTY_ANNOTATION_ARRAY = new Annotation[0];

	private static final Map<Class<?>, TypeDescriptor> commonTypesCache = new HashMap<>(18);

	private static final Class<?>[] CACHED_COMMON_TYPES = {
			boolean.class, Boolean.class, byte.class, Byte.class, char.class, Character.class,
			double.class, Double.class, int.class, Integer.class, long.class, Long.class,
			float.class, Float.class, short.class, Short.class, String.class, Object.class};

	private final Class<?> type;

	private final ResolvableType resolvableType;

	private final AnnotatedElementAdapter annotatedElement;
}
```