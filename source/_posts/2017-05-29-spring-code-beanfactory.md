---
title: spring 源码解析 4.BeanFactory
date: 2017-05-29 12:56:59
tags:
- spring container
categories:
- spring
- code
---
## 1. 概述
BeanFactory为Spring的IoC功能提供了基础，但它直接用于与其他第三方框架的集成，现在大部分Spring用户都具有历史性。 BeanFactory和相关接口（如BeanFactoryAware，InitializingBean，DisposableBean）仍然存在于Spring中，目的是与与Spring集成的大量第三方框架向后兼容。 通常第三方组件不能使用更现代的等同物，例如@PostConstruct或@PreDestroy，以便与JDK 1.4保持兼容，或避免依赖于JSR-250。

## 2. BeanDefinition
### 2.1 类图
![](assets/img/spring/BeanDefinition.png)

### 2.2 接口

##### 2.2.1 [AttributeAccessor](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/AttributeAccessor.java)
```java
/**
 * 定义一个通用规范的接口，用于附加和访问任意对象的元数据。
 */
public interface AttributeAccessor {

	/**
	 * 将{@code name}定义的属性设置为提供的{@code 值}。 如果{@code 值}为{@code null}，则属性为{@link #removeAttribute removed}。
	 * 一般来说，用户应该注意通过使用完全限定的名称防止与其他元数据属性重叠，也许使用类或包名称作为前缀。
	 */
	void setAttribute(String name, Object value);

	/**
	 * 获取由{@code name}标识的属性的值。 如果属性不存在，返回{@code null}。
	 */
	Object getAttribute(String name);

	/**
	 * 删除{@code name}标识的属性并返回其值。 如果找不到{@code name}下的属性，则返回{@code null}。
	 */
	Object removeAttribute(String name);

	/**
	 * 如果{@code name}标识的属性存在，则返回{@code true}。 否则返回{@code false}。
	 */
	boolean hasAttribute(String name);
	String[] attributeNames();
}
```

##### 2.2.2 [BeanMetadataElement](https://github.com/LeeRenbo/spring-framework/blob/master/spring-beans/src/main/java/org/springframework/beans/BeanMetadataElement.java)
```java
/**
 * 携带配置源对象的 bean metadata 元素实现的接口。
 */
public interface BeanMetadataElement {
	/**
	 * 返回此元数据元素的配置源{@code Object}（可能为{@code null}）。
	 */
	Object getSource();
}
```

##### 2.2.3 [BeanDefinition](https://github.com/LeeRenbo/spring-framework/blob/master/spring-beans/src/main/java/org/springframework/beans/factory/config/BeanDefinition.java)
```java
/**
 * BeanDefinition 描述了一个bean实例，它具有属性值，构造函数参数值以及具体实现提供的进一步信息。
 * 这只是一个最小的接口：主要目的是允许{@link BeanFactoryPostProcessor}（如{@link PropertyPlaceholderConfigurer}）来内省和修改属性值和其他bean元数据。
 */
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {

	/**
	 * 标准单例范围的范围标识符：“singleton”。
	 * 请注意，扩展bean工厂可能会支持更多的范围。
	 */
	String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;

	/**
	 * 标准原型范围的范围标识：“prototype”。
	 * 请注意，扩展bean工厂可能会支持更多的范围。
	 */
	String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;

	/**
	 * 角色提示，指出{@code BeanDefinition}是应用程序的主要部分。 通常对应于用户定义的bean。
	 */
	int ROLE_APPLICATION = 0;

	/**
	 * 角色提示，指出 {@code BeanDefinition} 是一些较大配置的支持部分，通常是外部{@link org.springframework.beans.factory.parsing.ComponentDefinition}。 {@code SUPPORT} bean被认为是重要的，以便在查看特定的{@link org.springframework.beans.factory.parsing.ComponentDefinition}时注意到，但在查看应用程序的整体配置时不会。
	 */
	int ROLE_SUPPORT = 1;

	/**
	 * 角色提示，指出{@code BeanDefinition}提供了完全的后台角色，与最终用户无关。 当注册完全是{@link org.springframework.beans.factory.parsing.ComponentDefinition}的内部工作的一部分的bean时，使用此提示。
	 */
	int ROLE_INFRASTRUCTURE = 2;


	// Modifiable attributes

	/**
	 * 设置此bean定义的父定义的名称（如果有）。
	 */
	void setParentName(String parentName);

	/**
	 * 返回此bean定义的父定义的名称（如果有）。
	 */
	String getParentName();

	/**
	 * 指定此bean定义的bean类名称。
	 * 可以在bean工厂后处理期间修改类名称，通常用解析变量替换原始类名。
	 */
	void setBeanClassName(String beanClassName);

	/**
	 * 返回此bean定义的当前bean类名。
	 * 请注意，这不一定是在运行时使用的实际类名，以便子代定义从其父代替覆盖/继承类名。
	 * 此外，这可能只是调用的工厂方法类，或者在调用方法的工厂bean引用的情况下甚至可能为空。
	 * 因此，不要认为这在运行时是确定的bean类型，而是仅在单个bean定义级别使用它来解析目的。
	 */
	String getBeanClassName();

	/**
	 * 覆盖此bean的目标范围，指定一个新的范围名称。
	 */
	void setScope(String scope);

	/**
	 * 返回此bean的当前目标范围的名称，如果尚未知道，则返回{@code null}。
	 */
	String getScope();

	/**
	 * 设置这个bean是否应该被懒惰地初始化。
	 * 如果{@code false}，bean将在启动时由Bean工厂实例化，Bean工厂会执行急速初始化单例。
	 */
	void setLazyInit(boolean lazyInit);

	/**
	 * 返回这个bean是否应该被懒惰地初始化，也就是说在启动时不是急切地被实例化。 
	 * 只适用于singleton bean。
	 */
	boolean isLazyInit();

	/**
	 * 设置该bean依赖于初始化的bean的名称。
	 * bean工厂将保证这些bean首先被初始化。
	 */
	void setDependsOn(String... dependsOn);

	/**
	 * 返回该bean依赖的bean名称。
	 */
	String[] getDependsOn();

	/**
	 * 设置这个bean是否是自动连线到其他bean的候选项。
	 * 请注意，此标志仅用于影响基于类类型的自动装配。
	 * 它不影响名称的显式引用，即使指定的bean未被标记为自动连接候选，它也将被解析。 因此，如果名称匹配，则通过名称自动装配将会注入一个bean。
	 */
	void setAutowireCandidate(boolean autowireCandidate);

	/**
	 * 返回这个bean是否是自动连线到其他bean的候选项。
	 */
	boolean isAutowireCandidate();

	/**
	 * 设置此bean是否是主要的自动连线候选。
	 * 如果这个值是{@code true}，在多个匹配候选者中只有一个bean，则它将作为一个打破者。
	 */
	void setPrimary(boolean primary);

	/**
	 * 返回此bean是否是主要的自动装配候选。
	 */
	boolean isPrimary();

	/**
	 * 指定要使用的工厂bean（如果有）。
	 * 这个为指定的工厂方法的bean的名称。
	 */
	void setFactoryBeanName(String factoryBeanName);

	/**
	 * 返回工厂bean名称，如果有的话。
	 */
	String getFactoryBeanName();

	/**
	 * 指定工厂方法（如果有）。
	 * 该方法将使用构造函数参数进行调用，如果没有指定，则不使用参数。
	 * 该方法将在指定的工厂bean（如果有）或其他方式在本地bean类上作为静态方法调用。
	 */
	void setFactoryMethodName(String factoryMethodName);

	/**
	 * 返回工厂方法（如有）。
	 */
	String getFactoryMethodName();

	/**
	 * 返回此bean的构造函数参数值。
	 * 可以在bean工厂post-processing期间修改返回的实例。
	 */
	ConstructorArgumentValues getConstructorArgumentValues();

	/**
	 * 返回要应用于bean新实例的属性值。
	 * 可以在bean工厂post-processing期间修改返回的实例。
	 */
	MutablePropertyValues getPropertyValues();


	// Read-only attributes

	boolean isSingleton();

	boolean isPrototype();

	boolean isAbstract();

	int getRole();

	/**
	 * 返回这个bean定义的人类可读描述。
	 */
	String getDescription();

	/**
	 * 返回这个bean定义来源的资源的描述（为了在发生错误时显示上下文的目的）。
	 */
	String getResourceDescription();

	/**
	 * 返回原始BeanDefinition，否则返回{@code null}。
	 * 允许检索被装饰的bean定义（如果有）。
	 * 请注意，此方法返回直接鼻祖。 通过鼻祖链迭代，以查找用户定义的原始BeanDefinition。
	 */
	BeanDefinition getOriginatingBeanDefinition();
}
```
##### 2.2.4 [AnnotatedBeanDefinition](https://github.com/LeeRenbo/spring-framework/blob/master/spring-beans/src/main/java/org/springframework/beans/factory/annotation/AnnotatedBeanDefinition.java)
```java
/**
 * 扩展{@link org.springframework.beans.factory.config.BeanDefinition}接口，暴露关于其bean类的{@link org.springframework.core.type.AnnotationMetadata}，而可以不需要加载该类。
 */
public interface AnnotatedBeanDefinition extends BeanDefinition {

	/**
	 * 为此bean定义的bean类获取注释元数据（以及基本类元数据）。
	 */
	AnnotationMetadata getMetadata();

	/**
	 * 获取此bean定义的工厂方法的元数据（如果有）。
	 */
	MethodMetadata getFactoryMethodMetadata();
}
```

### 2.3 实现
##### 2.3.1 [AttributeAccessorSupport](https://github.com/LeeRenbo/spring-framework/blob/master/spring-core/src/main/java/org/springframework/core/AttributeAccessorSupport.java)
```java
/**
 * {@link AttributeAccessor AttributeAccessors}的支持类，提供所有方法的基本实现。 被子类扩展。
 * 如果子类和所有属性值都是{@link Serializable}，则可以使用{@link Serializable}。
 */
@SuppressWarnings("serial")
public abstract class AttributeAccessorSupport implements AttributeAccessor, Serializable {
	/** Map with String keys and Object values */
	private final Map<String, Object> attributes = new LinkedHashMap<>(0);
```

##### 2.3.2 [BeanMetadataAttributeAccessor](https://github.com/LeeRenbo/spring-framework/blob/master/spring-beans/src/main/java/org/springframework/beans/BeanMetadataAttributeAccessor.java)
```java
/**
 * 扩展{@link org.springframework.core.AttributeAccessorSupport}，将属性保存为{@link BeanMetadataAttribute}对象，以便跟踪定义源。
 */
@SuppressWarnings("serial")
public class BeanMetadataAttributeAccessor extends AttributeAccessorSupport implements BeanMetadataElement {

	private Object source;

	public void setSource(Object source) {
		this.source = source;
	}

	/**
	 * 将给定的BeanMetadataAttribute添加到此访问器的一组属性中。
	 */
	public void addMetadataAttribute(BeanMetadataAttribute attribute) {
		super.setAttribute(attribute.getName(), attribute);
	}

	/**
	 * 在这个访问者的属性集中查找给定的BeanMetadataAttribute。
	 */
	public BeanMetadataAttribute getMetadataAttribute(String name) {
		return (BeanMetadataAttribute) super.getAttribute(name);
	}
}
```

```java
/**
 * 持有键值样式属性，是bean定义的一部分。
 * 保留键值对之外的跟踪定义源。
 */
public class BeanMetadataAttribute implements BeanMetadataElement {
	private final String name;
	private final Object value;
	private Object source;
}
```

##### 2.3.3 [AbstractBeanDefinition](https://github.com/LeeRenbo/spring-framework/blob/master/spring-beans/src/main/java/org/springframework/beans/factory/support/AbstractBeanDefinition.java)
```java
/**
 * 具体的，完整的{@link BeanDefinition}类的基类，分为{@link GenericBeanDefinition}，{@link RootBeanDefinition}和{@link ChildBeanDefinition}的常用属性。
 *
 * autowire常量与{@link org.springframework.beans.factory.config.AutowireCapableBeanFactory}接口中定义的常量相匹配。
 */
@SuppressWarnings("serial")
public abstract class AbstractBeanDefinition extends BeanMetadataAttributeAccessor
		implements BeanDefinition, Cloneable {

	/**
	 * 默认范围名称的常量：{@code ""}，相当于单例状态，除非从父bean定义（如果适用）覆盖。
	 */
	public static final String SCOPE_DEFAULT = "";

	public static final int AUTOWIRE_NO = AutowireCapableBeanFactory.AUTOWIRE_NO;
	public static final int AUTOWIRE_BY_NAME = AutowireCapableBeanFactory.AUTOWIRE_BY_NAME;
	public static final int AUTOWIRE_BY_TYPE = AutowireCapableBeanFactory.AUTOWIRE_BY_TYPE;
	public static final int AUTOWIRE_CONSTRUCTOR = AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR;
	@Deprecated
	public static final int AUTOWIRE_AUTODETECT = AutowireCapableBeanFactory.AUTOWIRE_AUTODETECT;

	/**
	 * 根本没有依赖关系。
	 */
	public static final int DEPENDENCY_CHECK_NONE = 0;
	/**
	 * 对象引用的依赖关系检查。
	 */
	public static final int DEPENDENCY_CHECK_OBJECTS = 1;
	/**
	 * 依赖关系检查“简单”属性。
	 */
	public static final int DEPENDENCY_CHECK_SIMPLE = 2;
	/**
	 * 所有属性的依赖关系检查
	 */
	public static final int DEPENDENCY_CHECK_ALL = 3;

	/**
	 * 常量，表示容器应尝试推断出一个bean的 {@link #setDestroyMethodName destroy method name}，而不是显式指定方法名称。 值{@value}专门设计为在方法名称中包括非法的字符，确保不会与具有相同名称的合法命名方法发生冲突。
	 * 目前，在destroy方法推断中检测到的方法名称是“close”和“shutdown”，如果存在于特定的bean类上。
	 */
	public static final String INFER_METHOD = "(inferred)";

	private volatile Object beanClass;

	private String scope = SCOPE_DEFAULT;

	private boolean abstractFlag = false;

	private boolean lazyInit = false;

	private int autowireMode = AUTOWIRE_NO;

	private int dependencyCheck = DEPENDENCY_CHECK_NONE;

    /**
	 * 该bean依赖于初始化的bean的名称。 bean工厂将保证这些bean首先被初始化。
     * 请注意，依赖关系通常通过bean属性或构造函数参数来表示。 对于其他类型的依赖关系（* ugh *）或启动时的数据库准备，此属性应该是必需的。
     */
	private String[] dependsOn;

    /**
	 * 设置这个bean是否是自动连线到其他bean的候选。
	 * 请注意，此标志仅用于影响基于类型的自动装配。 它不会影响名称的显式引用，即使指定的bean未标记为自动连接候选，也将得到解析。 因此，如果名称匹配，则通过名称自动装配将会注入一个bean。
     */
	private boolean autowireCandidate = true;

	private boolean primary = false;

    /**
    * 限定符，用于自动装配候选解决，由限定的类型名称作为key 。
    */
	private final Map<String, AutowireCandidateQualifier> qualifiers = new LinkedHashMap<>(0);

	/**
	 * 指定用于创建bean实例的回调，作为声明性指定的工厂方法的替代方法。
	 * 如果设置了这样的回调，它将覆盖任何其他构造函数或工厂方法元数据。
	 * 然而，bean属性和潜在注释驱动的注射仍然照常使用。
	 */
	private Supplier<?> instanceSupplier;

    /**
	 * 指定是否允许访问非公共构造函数和方法，对于指向这些的外部化元数据的情况。 默认值为{@code true}; 将其切换到{@code false}以允许进行公共访问。
	 * 这适用于构造函数解析，工厂方法解析以及init / destroy方法。 
	 * Bean属性访问器在任何情况下都必须公开，不受此设置的影响。
	 * 请注意，注释驱动的配置仍将访问非公开成员，只要它们已被注释。 此设置仅适用于此bean定义中的外部化元数据。
     */
	private boolean nonPublicAccessAllowed = true;

    /**
	 * 指定是否以宽松模式（{@code true}（默认值为））解析构造函数，或者切换到严格解析
	 * （如果在转换参数时模糊匹配所有的构造函数时抛出异常，而宽大模式将使用 一个与“最接近”类型匹配）。
	 */
	private boolean lenientConstructorResolution = true;

	private String factoryBeanName;

	/**
	 * 指定工厂方法（如果有）。 该方法将使用构造函数参数进行调用，如果没有指定，则不使用参数。
	 * 该方法将在指定的工厂bean（如果有）否则在本地bean类上作为静态方法调用。
	 */
	private String factoryMethodName;

	private ConstructorArgumentValues constructorArgumentValues;

	private MutablePropertyValues propertyValues;

	private MethodOverrides methodOverrides = new MethodOverrides();

	private String initMethodName;

	private String destroyMethodName;

	private boolean enforceInitMethod = true;

	private boolean enforceDestroyMethod = true;

	/**
	 * 设置此bean定义是否为“合成”，即不由应用程序本身定义（例如，通过{@code <aop：config>}创建的基础架构bean，例如自动代理的帮助器）。
	 */
	private boolean synthetic = false;

	private int role = BeanDefinition.ROLE_APPLICATION;

	private String description;

	/**
	 * 这个bean定义来自的资源。
	 */
	private Resource resource;
}
```

```java
/**
 * 确定自动装配候选人的资格。 包含一个或多个此类限定符的bean定义可以对字段或参数上的注释进行精细匹配以进行自动装配。
 */
public class AutowireCandidateQualifier extends BeanMetadataAttributeAccessor {
    /**
     * 将value存在 key 是 VALUE_KEY 的 BeanMetadataAttributeAccessor抽象实现中
     */
	public static String VALUE_KEY = "value";
	private final String typeName;
}
```

```java
/**
 *  {@link org.springframework.beans.factory.config.BeanDefinition}的描述性{@link org.springframework.core.io.Resource}包装器。
 */
class BeanDefinitionResource extends AbstractResource {

	private final BeanDefinition beanDefinition;
}
```

```java
/**
 * 构造函数参数值的持有者，通常作为bean定义的一部分。
 * 支持构造函数参数列表中特定索引的值以及类型中的泛型参数匹配。
 */
public class ConstructorArgumentValues {
	private final Map<Integer, ValueHolder> indexedArgumentValues = new LinkedHashMap<>(0);
	private final List<ValueHolder> genericArgumentValues = new LinkedList<>();
	
	/**
	 * Holder for a constructor argument value, with an optional type
	 * attribute indicating the target type of the actual constructor argument.
	 * 
	 * 构造函数参数值的持有者，可选的type属性指示实际构造函数参数的目标类型。
	 */
	public static class ValueHolder implements BeanMetadataElement {
		private Object value;
		private String type;
		private String name;
		private Object source;
		private boolean converted = false;
		private Object convertedValue;
    }
}
```

```java
/**
 * {@link PropertyValues}接口的默认实现。
 * 允许对属性进行简单的操作，并提供构造函数以支持从Map的深层复制和构造。
 */
@SuppressWarnings("serial")
public class MutablePropertyValues implements PropertyValues, Serializable {
	private final List<PropertyValue> propertyValueList;
	private Set<String> processedProperties;
	private volatile boolean converted = false;
}
```
```java
/**
 * 包含一个或多个{@link PropertyValue}对象的Holder，通常包含特定目标bean的一个更新。
 */
public interface PropertyValues {

	PropertyValue[] getPropertyValues();
	PropertyValue getPropertyValue(String propertyName);
	/**
	 * 返回自上一个PropertyValues之后的更改。 子类也应该覆盖{@code equals}。
	 */
	PropertyValues changesSince(PropertyValues old);
	boolean contains(String propertyName);
	boolean isEmpty();
}
```

```java
/**
 * 对象保存单个bean属性的信息和值。 在这里使用一个对象，而不仅仅是将所有属性存储在按属性名称键入的 map 中，这样可以更有弹性，并以优化的方式处理索引属性等。
 * 请注意，该值不需要是最终必需的类型：{@link BeanWrapper}实现应该处理任何必要的转换，因为该对象不知道将要应用的对象的任何内容。
 */
@SuppressWarnings("serial")
public class PropertyValue extends BeanMetadataAttributeAccessor implements Serializable {
	private final String name;
	private final Object value;
	private boolean optional = false;
	private boolean converted = false;
	private Object convertedValue;
	/** Package-visible field that indicates whether conversion is necessary */
	volatile Boolean conversionNecessary;
	/** Package-visible field for caching the resolved property path tokens */
	transient volatile Object resolvedTokens;
}
```
```java
/**
 * {@code BeanDefinition}属性的简单持有人默认。
 */
public class BeanDefinitionDefaults {
	private boolean lazyInit;
	private int dependencyCheck = AbstractBeanDefinition.DEPENDENCY_CHECK_NONE;
	private int autowireMode = AbstractBeanDefinition.AUTOWIRE_NO;
	private String initMethodName;
	private String destroyMethodName;
}
```

##### 2.3.4 [RootBeanDefinition](https://github.com/LeeRenbo/spring-framework/blob/master/spring-beans/src/main/java/org/springframework/beans/factory/support/RootBeanDefinition.java)
```java
/**
 * 根bean定义代表在运行时在Spring BeanFactory中备份特定bean的合并的bean定义。
 * 它可能已经从多个相互继承原始bean定义创建，通常以{@link GenericBeanDefinition GenericBeanDefinitions}的形式进行创建。
 * 根bean定义本质上是运行时的“统一”bean定义视图。
 *
 * 根bean定义也可用于在配置阶段注册各个bean定义。
 * 但是，从Spring 2.5，以编程方式注册bean定义的首选方式是{@link GenericBeanDefinition}类。
 * GenericBeanDefinition 的优点是允许动态定义父依赖关系，而不是将角色“硬编码”为根bean定义。
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @see GenericBeanDefinition
 * @see ChildBeanDefinition
 */
@SuppressWarnings("serial")
public class RootBeanDefinition extends AbstractBeanDefinition {

	private BeanDefinitionHolder decoratedDefinition;

	/**
	 * 指定{@link AnnotatedElement}定义限定符，而不是目标类或工厂方法。
	 */
	private AnnotatedElement qualifiedElement;

	boolean allowCaching = true;

	boolean isFactoryMethodUnique = false;

	/**
	 * 如果事先知道，请指定此bean定义的含有泛型的目标类型。
	 */
	volatile ResolvableType targetType;

	/** Package-visible field for caching the determined Class of a given bean definition
	 * 缓存给定bean定义的确定类
	 * */
	volatile Class<?> resolvedTargetType;

	/** Package-visible field for caching the return type of a generically typed factory method
	 * 缓存返回类型的一般类型的工厂方法
	 * */
	volatile ResolvableType factoryMethodReturnType;

	/** Common lock for the four constructor fields below
	 * 以下四个构造函数字段的通用锁
	 * */
	final Object constructorArgumentLock = new Object();

	/** Package-visible field for caching the resolved constructor or factory method
	 * 缓存已解决的构造函数或工厂方法
	 * */
	Executable resolvedConstructorOrFactoryMethod;

	/** Package-visible field that marks the constructor arguments as resolved
	 * 将构造函数的参数标记为已解决
	 * */
	boolean constructorArgumentsResolved = false;

	/** Package-visible field for caching fully resolved constructor arguments
	 * 缓存完全解析的构造函数参数
	 * */
	Object[] resolvedConstructorArguments;

	/** Package-visible field for caching partly prepared constructor arguments
	 * 缓存部分准备的构造函数参数
	 * */
	Object[] preparedConstructorArguments;

	/** Common lock for the two post-processing fields below
	 * 以下两个后处理字段的通用锁
	 * */
	final Object postProcessingLock = new Object();

	/** Package-visible field that indicates MergedBeanDefinitionPostProcessor having been applied
	 * 表示已应用MergedBeanDefinitionPostProcessor
	 * */
	boolean postProcessed = false;

	/** Package-visible field that indicates a before-instantiation post-processor having kicked in
	 * 表示已经一个实例化后处理器已经触发
	 * */
	volatile Boolean beforeInstantiationResolved;

	private Set<Member> externallyManagedConfigMembers;

	private Set<String> externallyManagedInitMethods;

	private Set<String> externallyManagedDestroyMethods;
}
```

##### 2.3.5 [RootBeanDefinition](https://github.com/LeeRenbo/spring-framework/blob/master/spring-beans/src/main/java/org/springframework/beans/factory/support/RootBeanDefinition.java)
```java
/**
 * 根bean定义代表融合的bean定义，运行时在Spring BeanFactory中备份特定bean的。
 * 它可能已经从多个相互继承原始bean定义创建，通常以{@link GenericBeanDefinition GenericBeanDefinitions}的形式进行创建。
 * 根bean定义本质上是运行时的“统一”bean定义视图。
 *
 * 根bean定义也可用于在配置阶段注册各个bean定义。
 * 但是，从Spring 2.5，以编程方式注册bean定义的首选方式是{@link GenericBeanDefinition}类。
 * GenericBeanDefinition 的优点是允许动态定义父依赖关系，而不是将角色“硬编码”为根bean定义。
 */
@SuppressWarnings("serial")
public class RootBeanDefinition extends AbstractBeanDefinition {

	private BeanDefinitionHolder decoratedDefinition;

	/**
	 * 指定{@link AnnotatedElement}定义限定符，而不是目标类或工厂方法。
	 */
	private AnnotatedElement qualifiedElement;

	boolean allowCaching = true;

	boolean isFactoryMethodUnique = false;

	/**
	 * 如果事先知道，请指定此bean定义的含有泛型的目标类型。
	 */
	volatile ResolvableType targetType;

	/**
	 * 缓存给定bean定义的确定类
	 */
	volatile Class<?> resolvedTargetType;

	/**
	 * 缓存返回类型的一般类型的工厂方法
	 */
	volatile ResolvableType factoryMethodReturnType;

	/**
	 * 以下四个构造函数字段的通用锁
	 */
	final Object constructorArgumentLock = new Object();

	/**
	 * 缓存已解析的构造函数或工厂方法
	 */
	Executable resolvedConstructorOrFactoryMethod;

	/**
	 * 将构造函数的参数标记为已解决
	 */
	boolean constructorArgumentsResolved = false;

	/**
	 * 缓存完全解析的构造函数参数
	 */
	Object[] resolvedConstructorArguments;

	/**
	 * 缓存部分准备的构造函数参数
	 */
	Object[] preparedConstructorArguments;

	/**
	 * 以下两个后处理字段的通用锁
	 */
	final Object postProcessingLock = new Object();

	/**
	 * 表示已应用MergedBeanDefinitionPostProcessor
	 */
	boolean postProcessed = false;

	/**
	 * Package-visible field that indicates a before-instantiation post-processor having kicked in
	 * 表示已经一个实例化后处理器已经触发
	 */
	volatile Boolean beforeInstantiationResolved;

	private Set<Member> externallyManagedConfigMembers;

	private Set<String> externallyManagedInitMethods;

	private Set<String> externallyManagedDestroyMethods;

}
```

##### 2.3.6 [GenericBeanDefinition](https://github.com/LeeRenbo/spring-framework/blob/master/spring-beans/src/main/java/org/springframework/beans/factory/support/GenericBeanDefinition.java)
```java
/**
 * GenericBeanDefinition is a one-stop shop for standard bean definition purposes.
 * Like any bean definition, it allows for specifying a class plus optionally
 * constructor argument values and property values. Additionally, deriving from a
 * parent bean definition can be flexibly configured through the "parentName" property.
 *
 * <p>In general, use this {@code GenericBeanDefinition} class for the purpose of
 * registering user-visible bean definitions (which a post-processor might operate on,
 * potentially even reconfiguring the parent name). Use {@code RootBeanDefinition} /
 * {@code ChildBeanDefinition} where parent/child relationships happen to be pre-determined.
 *
 * GenericBeanDefinition是标准bean定义的一站式服务。
 * 像任何bean定义一样，它允许指定一个类加上可选的构造函数参数值和属性值。
 * 另外，可以通过“parentName”属性灵活地配置从父bean定义的派生。
 *
 * 一般来说，使用这个{@code GenericBeanDefinition}类用于注册用户可见bean定义（后处理器可能操作bean定义，甚至可能重新配置父名称）。
 * 使用{@code RootBeanDefinition} / {@code ChildBeanDefinition}，其中父/子关系恰好是预先确定的。
 *
 * @author Juergen Hoeller
 * @since 2.5
 * @see #setParentName
 * @see RootBeanDefinition
 * @see ChildBeanDefinition
 */
@SuppressWarnings("serial")
public class GenericBeanDefinition extends AbstractBeanDefinition {

	private String parentName;

}
```

##### 2.3.7 [ChildBeanDefinition](https://github.com/LeeRenbo/spring-framework/blob/master/spring-beans/src/main/java/org/springframework/beans/factory/support/ChildBeanDefinition.java)
```java
/**
 * 继承自父项设置的Bean的Bean定义。 子bean定义与父bean定义有固定的依赖关系。
 *
 * 子bean定义将从父级继承构造函数参数值，属性值和方法覆盖，并添加新值。
 * 如果指定了init方法，destroy方法和/或静态工厂方法，它们将覆盖相应的父设置。
 * 剩余的设置将始终从子定义中取出：depends on, autowire mode, dependency check, singleton, lazy init。
 *
 * 注意： 由于Spring 2.5，以编程方式注册bean定义的首选方式是{@link GenericBeanDefinition}类，它允许通过{@link GenericBeanDefinition＃setParentName}动态定义父依赖关系， 方法。
 * 这有效地取代了大多数用例中的ChildBeanDefinition类。
 */
public class ChildBeanDefinition extends AbstractBeanDefinition {
	private String parentName;

}
```

##### 2.3.8 [ScannedGenericBeanDefinition](https://github.com/LeeRenbo/spring-framework/blob/master/spring-context/src/main/java/org/springframework/context/annotation/ScannedGenericBeanDefinition.java)
```java
/**
 * 扩展{@link org.springframework.beans.factory.support.GenericBeanDefinition}类，基于ASM ClassReader，支持通过{@link AnnotatedBeanDefinition}接口暴露注释元数据。
 * 此类不提早加载bean {@code Class}。 它相当于从“.class”文件本身检索所有相关的元数据，并与ASM ClassReader解析。 它在功能上等同于{@link AnnotatedGenericBeanDefinition#AnnotatedGenericBeanDefinition(AnnotationMetadata)}，但是区分已被扫描的类型bean与以其他方式注册或检测到的那些类型。
 */
public class ScannedGenericBeanDefinition extends GenericBeanDefinition implements AnnotatedBeanDefinition {
	private final AnnotationMetadata metadata;
}
```

##### 2.3.9 [AnnotatedGenericBeanDefinition](https://github.com/LeeRenbo/spring-framework/blob/master/spring-beans/src/main/java/org/springframework/beans/factory/annotation/AnnotatedGenericBeanDefinition.java)
```java
/**
 * 扩展{@link org.springframework.beans.factory.support.GenericBeanDefinition}类，添加了通过{@link AnnotatedBeanDefinition}接口暴露注释元数据的支持。
 * 
 * 此GenericBeanDefinition变体主要用于测试期望在AnnotatedBeanDefinition上操作的代码，例如Spring组件扫描支持中的策略实现（默认定义类为{@link org.springframework.context.annotation.ScannedGenericBeanDefinition} ，它还实现了AnnotatedBeanDefinition接口）。
 */
@SuppressWarnings("serial")
public class AnnotatedGenericBeanDefinition extends GenericBeanDefinition implements AnnotatedBeanDefinition {

	private final AnnotationMetadata metadata;

	private MethodMetadata factoryMethodMetadata;
}
```

## 3.BeanDefinitionReader
### 3.1 类图
### 3.2 接口
##### 3.2.1 [BeanDefinitionReader](https://github.com/LeeRenbo/spring-framework/blob/master/spring-beans/src/main/java/org/springframework/beans/factory/support/BeanDefinitionReader.java)
```java
/**
 * 用于bean定义阅读器的简单接口。
 * 指定具有 Resource 和 String 位置参数的加载方法。
 *
 * 具体的bean定义读取器当然可以为bean定义添加额外的加载和注册方法，特定于它们的bean定义格式。
  *
   请注意，bean定义阅读器不必实现此接口。 它仅作为要遵循标准命名约定的bean定义阅读器的建议。
 */
public interface BeanDefinitionReader {

	/**
     * 返回bean工厂注册bean定义。
     * 工厂通过BeanDefinitionRegistry接口暴露，封装了与bean定义处理相关的方法。
	 */
	BeanDefinitionRegistry getRegistry();

	ResourceLoader getResourceLoader();

	/**
     * 返回类加载器以用于bean类。
     * {@code null}建议不要热切地加载bean类，而是仅仅使用类名注册bean定义，相应的类将在稍后（或从不）解析。
	 */
	ClassLoader getBeanClassLoader();

	/**
     * 返回BeanNameGenerator用于匿名bean（未指定明确的bean名称）。
	 */
	BeanNameGenerator getBeanNameGenerator();


	/**
     * 从指定的资源中加载bean定义。
	 */
	int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException;
	int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException;

	/**
     * 从指定的资源位置加载bean定义。
     * 该位置也可以是一个位置模式，前提是该bean定义阅读器的ResourceLoader是ResourcePatternResolver。
	 */
	int loadBeanDefinitions(String location) throws BeanDefinitionStoreException;
	int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException;

}

```

##### 3.2.2 [AbstractBeanDefinitionReader](https://github.com/LeeRenbo/spring-framework/blob/master/spring-beans/src/main/java/org/springframework/beans/factory/support/AbstractBeanDefinitionReader.java)
```java
/**
 * 实现{@link BeanDefinitionReader}接口的bean定义阅读器的抽象基类。
 * 提供常见的属性，如bean工厂工作，类加载器用于加载bean类。
 */
public abstract class AbstractBeanDefinitionReader implements EnvironmentCapable, BeanDefinitionReader {

	/** Logger available to subclasses */
	protected final Log logger = LogFactory.getLog(getClass());

	private final BeanDefinitionRegistry registry;

	private ResourceLoader resourceLoader;

	private ClassLoader beanClassLoader;

	private Environment environment;

	private BeanNameGenerator beanNameGenerator = new DefaultBeanNameGenerator();
}
```

```java
/**
 * 用于保存bean定义的注册表的接口，例如RootBeanDefinition和ChildBeanDefinition实例。 通常由BeanFactories实现，内部使用AbstractBeanDefinition层次结构。
 * 这是Spring的bean工厂包中唯一封装Bean定义注册的接口。 标准BeanFactory接口仅涵盖对完全配置的工厂实例的访问。
 * Spring的bean定义阅读器期望在此接口的实现上工作。 Spring内核中的已知实现者是DefaultListableBeanFactory和GenericApplicationContext。
 */
public interface BeanDefinitionRegistry extends AliasRegistry {
	/**
	 * 使用此注册表注册新的bean定义。
	 * 必须支持RootBeanDefinition和ChildBeanDefinition。
	 */
	void registerBeanDefinition(String beanName, BeanDefinition beanDefinition) throws BeanDefinitionStoreException;
	
	void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;
	BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;
	boolean containsBeanDefinition(String beanName);
	String[] getBeanDefinitionNames();
	int getBeanDefinitionCount();
	boolean isBeanNameInUse(String beanName);
}

```
##### 3.2.3 [BeanDefinitionDocumentReader](https://github.com/LeeRenbo/spring-framework/blob/master/spring-beans/src/main/java/org/springframework/beans/factory/xml/BeanDefinitionDocumentReader.java)
```java
/**
 * SPI用于解析包含Spring bean定义的XML文档。 由{@link XmlBeanDefinitionReader}用于实际解析DOM文档。
 *
 * 为每个文档实例化来解析：在执行{@code registerBeanDefinitions}方法时，实现可以在实例变量中保留状态; 例如，为文档中的所有bean定义定义的全局设置。
 */
public interface BeanDefinitionDocumentReader {

	/**
	 * 从给定的DOM文档中读取bean定义，并在给定的阅读器上下文中注册它们。
	 */
	void registerBeanDefinitions(Document doc, XmlReaderContext readerContext)
			throws BeanDefinitionStoreException;

}

```
### 3.3 实现
##### 3.3.1 [XmlBeanDefinitionReader](https://github.com/LeeRenbo/spring-framework/blob/master/spring-beans/src/main/java/org/springframework/beans/factory/xml/XmlBeanDefinitionReader.java)
```java
/**
 * 用于XML bean定义的Bean定义阅读器。
 * 将实际的XML文档读取委托给{@link BeanDefinitionDocumentReader}接口的实现。
 *
 * 通常应用于{@link org.springframework.beans.factory.support.DefaultListableBeanFactory}或{@link org.springframework.context.support.GenericApplicationContext}。
 *
 * 此类加载DOM文档并将BeanDefinitionDocumentReader应用于该文档。 文档读取器将向给定的bean工厂注册每个bean定义，并与后者的{@link org.springframework.beans.factory.support.BeanDefinitionRegistry}接口的实现进行交互。
 */
public class XmlBeanDefinitionReader extends AbstractBeanDefinitionReader {

	public static final int VALIDATION_NONE = XmlValidationModeDetector.VALIDATION_NONE;
	public static final int VALIDATION_AUTO = XmlValidationModeDetector.VALIDATION_AUTO;
	public static final int VALIDATION_DTD = XmlValidationModeDetector.VALIDATION_DTD;
	public static final int VALIDATION_XSD = XmlValidationModeDetector.VALIDATION_XSD;

	/**
	 * Constants instance for this class
	 * 此类可用于解析其他类包含定义的 public static final 成员常量
	 */
	private static final Constants constants = new Constants(XmlBeanDefinitionReader.class);

	private int validationMode = VALIDATION_AUTO;

	private boolean namespaceAware = false;

	/**
	 * 负责实际阅读XML bean定义文档的类。
	 * 为每个文档实例化来解析
	 */
	private Class<?> documentReaderClass = DefaultBeanDefinitionDocumentReader.class;

	/**
	 * 默认实现是{@link org.springframework.beans.factory.parsing.FailFastProblemReporter}，表现出快速行为的失败。 外部工具可以提供一个替代实现，整理错误和警告以在工具UI中显示。
	 */
	private ProblemReporter problemReporter = new FailFastProblemReporter();

	/**
	 * EmptyReaderEventListener它丢弃每个事件通知。 外部工具可以提供一个替代实现来监视在BeanFactory中注册的组件。
	 */
	private ReaderEventListener eventListener = new EmptyReaderEventListener();

	/**
	 * 默认实现是{@link NullSourceExtractor}，它只返回{@code null}作为源对象。 这意味着 - 在正常运行时执行期间 - 没有额外的源元数据附加到bean配置元数据。
	 */
	private SourceExtractor sourceExtractor = new NullSourceExtractor();

	/**
	 * 如果没有指定，将通过 {@link #createDefaultNamespaceHandlerResolver()} 创建一个 DefaultNamespaceHandlerResolver 默认实例。
	 */
	private NamespaceHandlerResolver namespaceHandlerResolver;

	/**
	 * 使用JAXP加载{@link Document}实例的{@link DefaultDocumentLoader}。
	 */
	private DocumentLoader documentLoader = new DefaultDocumentLoader();

	/**
	 * 默认情况下，将使用{@link ResourceEntityResolver}。 可以被自定义实体解析覆盖，例如相对于某些特定的基本路径。
	 * 用于解决 xsd dtd 资源的网络加载问题
	 */
	private EntityResolver entityResolver;

	/**
	 * {@code org.xml.sax.ErrorHandler}接口的实现，以便自定义处理XML解析错误和警告。
	 * 如果未设置，则使用默认的SimpleSaxErrorHandler，它仅使用视图类的记录器实例记录警告，并重新创建错误以停止XML转换。
	 */
	private ErrorHandler errorHandler = new SimpleSaxErrorHandler(logger);

	/**
	 * 检测XML流是否使用基于DTD或XSD的验证。
	 */
	private final XmlValidationModeDetector validationModeDetector = new XmlValidationModeDetector();

	private final ThreadLocal<Set<EncodedResource>> resourcesCurrentlyBeingLoaded =
			new NamedThreadLocal<>("XML bean definition resources currently being loaded");

	@Override
	public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
		return loadBeanDefinitions(new EncodedResource(resource));
	}

	public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isInfoEnabled()) {
			logger.info("Loading XML bean definitions from " + encodedResource.getResource());
		}

		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
		if (currentResources == null) {
			currentResources = new HashSet<>(4);
			this.resourcesCurrentlyBeingLoaded.set(currentResources);
		}
		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		try {
			InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) {
					inputSource.setEncoding(encodedResource.getEncoding());
				}
				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
			}
			finally {
				inputStream.close();
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"IOException parsing XML document from " + encodedResource.getResource(), ex);
		}
		finally {
			currentResources.remove(encodedResource);
			if (currentResources.isEmpty()) {
				this.resourcesCurrentlyBeingLoaded.remove();
			}
		}
	}

	public int loadBeanDefinitions(InputSource inputSource) throws BeanDefinitionStoreException {
		return loadBeanDefinitions(inputSource, "resource loaded through SAX InputSource");
	}

	public int loadBeanDefinitions(InputSource inputSource, String resourceDescription)
			throws BeanDefinitionStoreException {
		return doLoadBeanDefinitions(inputSource, new DescriptiveResource(resourceDescription));
	}


	/**
	 * Actually load bean definitions from the specified XML file.
	 * @param inputSource the SAX InputSource to read from
	 * @param resource the resource descriptor for the XML file
	 * @return the number of bean definitions found
	 * @throws BeanDefinitionStoreException in case of loading or parsing errors
	 * @see #doLoadDocument
	 * @see #registerBeanDefinitions
	 */
	protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
			Document doc = doLoadDocument(inputSource, resource);
			return registerBeanDefinitions(doc, resource);
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		catch (SAXParseException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
		}
		catch (SAXException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"XML document from " + resource + " is invalid", ex);
		}
		catch (ParserConfigurationException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Parser configuration exception parsing XML from " + resource, ex);
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"IOException parsing XML document from " + resource, ex);
		}
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Unexpected exception parsing XML document from " + resource, ex);
		}
	}

	/**
	 * Actually load the specified document using the configured DocumentLoader.
	 * 实际上使用配置的DocumentLoader加载指定的文档
	 */
	protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
		return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
				getValidationModeForResource(resource), isNamespaceAware());
	}


	/**
	 * Register the bean definitions contained in the given DOM document.
	 * Called by {@code loadBeanDefinitions}.
	 * <p>Creates a new instance of the parser class and invokes
	 * {@code registerBeanDefinitions} on it.
	 *
	 * 注册给定的DOM文档中包含的bean定义。 由{@code loadBeanDefinitions}调用。
	 * 创建解析器类的新实例，并调用{@code registerBeanDefinitions}。
	 *
	 * @param doc the DOM document
	 * @param resource the resource descriptor (for context information)
	 * @return the number of bean definitions found
	 * @throws BeanDefinitionStoreException in case of parsing errors
	 * @see #loadBeanDefinitions
	 * @see #setDocumentReaderClass
	 * @see BeanDefinitionDocumentReader#registerBeanDefinitions
	 */
	public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		int countBefore = getRegistry().getBeanDefinitionCount();
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}

	/**
	 * Create the {@link BeanDefinitionDocumentReader} to use for actually
	 * reading bean definitions from an XML document.
	 * <p>The default implementation instantiates the specified "documentReaderClass".
	 * @see #setDocumentReaderClass
	 */
	protected BeanDefinitionDocumentReader createBeanDefinitionDocumentReader() {
		return BeanDefinitionDocumentReader.class.cast(BeanUtils.instantiateClass(this.documentReaderClass));
	}

	/**
	 * Create the {@link XmlReaderContext} to pass over to the document reader.
	 */
	public XmlReaderContext createReaderContext(Resource resource) {
		return new XmlReaderContext(resource, this.problemReporter, this.eventListener,
				this.sourceExtractor, this, getNamespaceHandlerResolver());
	}
}

```
##### 3.3.2 [DefaultBeanDefinitionDocumentReader]()