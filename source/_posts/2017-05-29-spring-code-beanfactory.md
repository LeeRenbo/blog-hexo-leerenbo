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

