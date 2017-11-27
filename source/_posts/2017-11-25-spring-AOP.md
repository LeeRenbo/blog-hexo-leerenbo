---
title: AOP
date: 2017-11-25 21:21:52
tags:
---
# 5.1.介绍

面向切面编程（AOP）通过提供另一种思考程序结构的方式来补充面向对象编程（OOP）。 OOP中模块化的关键单元是类，而AOP中模块化的单元是aspect。 Aspects可以使关注的模块化，例如跨越多种类型和对象的事务管理。 


# 5.1.1 概念

Aspect: 横切多个类的模块。 事务管理是企业Java应用程序中横切关注的一个很好的例子。 在Spring AOP中，切面可以使用类（基于xml的方法）或@Aspect注解（@AspectJ风格）标注的通用类。

Join point: 一个程序的执行，如方法的执行或异常的处理过程中的一个点。 在Spring AOP中，连接点总是代表一个方法的执行。

Advice: 在切面的一个特定采取行动的连接点。 不同类型的建议包括“周围”，“之前”和“之后”的建议。 （建议类型将在下面讨论。）许多AOP框架，包括Spring，都将建议建模为拦截器，在连接点周围维护一个拦截器链。

Pointcut: 一个匹配连接点的判断。 将Advice与Join point用表达式相关联，并在切入点匹配的任何连接点（例如，执行具有特定名称的方法）上运行。 与切入点表达式匹配的连接点的概念是AOP的核心，Spring默认使用AspectJ切入点表达式语言。

Introduction: 代表类型声明额外方法或字段。 Spring AOP允许您向任何建议的对象引入新的接口（和相应的实现）。 例如，您可以使用 introduction 来使bean实现一个IsModified接口，以简化缓存。 （在AspectJ社区中，introduction被称为一个inter-type declaration。）

Target object: 对象被一个或多个方面建议。 也被称为被建议对象。 由于Spring AOP是使用运行时代理实现的，因此该对象将始终是代理对象。

AOP proxy: 一个由AOP框架创建的对象，用于实现切面合约（建议方法执行等等）。 在Spring框架中，AOP代理将是JDK动态代理或CGLIB代理。

Weaving: 链接切面与其他应用程序类型或对象，创建建议的对象。 这可以在编译时（例如使用AspectJ编译器），加载时间或运行时完成。 像其他纯Java AOP框架一样，Spring AOP在运行时执行编织。

advice 类型
Before advice: 在连接点之前执行的建议，但无法阻止执行流程继续到连接点（除非抛出异常）。

After returning advice: 连接点正常完成后要执行的建议：例如，如果方法返回而不抛出异常。

After throwing advice: 如果方法通过抛出异常退出，则要执行的建议。

After (finally) advice: 无论加入点退出的方式（正常或异常退回），要执行的建议。

Around advice: 围绕连接点（如方法调用）的建议。 这是最强大的建议。 周围的建议可以在方法调用之前和之后执行自定义行为。 它还负责选择是否继续加入点，还是通过返回自己的返回值或引发异常来缩短建议的方法执行。

建议使用能够实现所需行为的功能最低的 advice 类型.
pointcuts 是AOP的关键，能独立于对象继承关系来建议方法。

# 5.1.2 Spring AOP的功能和目标

Spring AOP是用纯Java实现的。 不需要特殊的编译过程。 Spring AOP不需要控制类加载器层次结构，因此适用于Servlet容器或应用程序服务器。

Spring AOP目前仅支持方法执行连接点（建议Spring bean的方法的执行）。 虽然可以在不破坏核心Spring AOP API的情况下添加对属性拦截的支持，但没实现属性拦截。 如果您需要建议属性访问和更新连接点，请考虑使用诸如AspectJ之类的语言。

Spring AOP的AOP方法与其他大多数AOP框架不同。 目标不是提供最完整的AOP实现（尽管Spring AOP是相当有能力的）; 而是提供AOP实现和Spring IoC之间的紧密集成，以帮助解决企业应用程序中的常见问题。

因此，例如，Spring框架的AOP功能通常与Spring IoC容器一起使用。 切面使用正常的bean定义语法进行配置（尽管这允许强大的“自动代理”功能）：这是与其他AOP实现的关键区别。 有些事情你不能用Spring AOP轻松或有效地完成，比如建议非常细粒度的对象（比如域对象）：在这种情况下，AspectJ是最好的选择。 但是，我们的经验是，Spring AOP为适用于AOP的企业Java应用程序中的大多数问题提供了极好的解决方案。

Spring AOP将永远不会与AspectJ竞争提供全面的AOP解决方案。 我们认为像Spring AOP这样的基于代理的框架和像AspectJ这样的全面的框架都是有价值的，而且它们是互补的，而不是竞争。 Spring将Spring AOP和IoC与AspectJ无缝集成，以便在一致的基于Spring的应用程序体系结构中满足AOP的所有用途。 此集成不影响Spring AOP API或AOP Alliance API：Spring AOP保持向后兼容。 有关Spring AOP API的讨论，请参阅以下章节。

# 5.1.3 AOP代理
Spring AOP 代理默认使用标准JDK动态代理。 这使得任何接口（或一组接口）都可以被代理。

Spring AOP也可以使用CGLIB代理。 这是代理 类而不是接口的必要条件。 如果业务对象没有实现接口，则默认使用CGLIB。 面向接口编程而不是类是个好习惯; 业务类通常会实现一个或多个业务接口。 可以强制使用CGLIB，在那些需要建议未在接口中声明的方法的情况下（希望很少），或者需要将代理对象作为具体类型传递给方法的情况。

掌握Spring AOP是基于代理的事实是很重要的。 请参阅了解AOP代理，以彻底检查此实现细节的实际含义。

# 5.2. @AspectJ support
[https://www.eclipse.org/aspectj/](https://www.eclipse.org/aspectj/)
# 5.2.1 @Configuration @EnableAspectJAutoProxy
# 5.2.2 @Aspect
# 5.2.3 @Pointcut

execution - 用于匹配方法执行的连接点，在使用Spring AOP时，你会使用的主要切入点指示符
execution(modifiers-pattern? ret-type-pattern declaring-type-pattern?name-pattern(param-pattern) throws-pattern?)
modifiers-pattern 作用域
ret-type-pattern 返回类型(必填) * 匹配所有类型。只有当方法返回给定类型时，完全限定类型名称才会匹配。
declaring-type-pattern 如果指定类型匹配，需要使用后缀 . 来加入name-pattern。 ..*包及子包
name-pattern * 作为全部或部分方法名称
param-pattern ()无参数； (..)任意参数； (*)一个任意类型参数； 

within - 限定匹配某些路径类型的连接点

this - 限定匹配类型的连接点，常用于绑定形式。advice中可以获取 proxy 对象。

target - 限定匹配类型的连接点，常用于绑定形式。advice中可以获取 target 对象。

args - 限定匹配参数类型的连接点，常用于绑定形式。advice中可以获取 方法参数。
args(java.io.Serializable) 区别 execution(* *(java.io.Serializable))。args，只要有参数是Serializable就匹配。execution只有一个参数且是Serializable才匹配。

@target - 限定匹配方法有指定 annotation 的连接点，常用于绑定形式。advice中可以获取 annotation 对象。

@args - 限定匹配参数有 annotation 的连接点，常用于绑定形式。advice中可以获取 annotation 对象。

@within - limits matching to join points within types that have the given annotation (the execution of methods declared in types with the given annotation when using Spring AOP)

@annotation - 限定匹配方法上有 annotation 的连接点，常用于绑定形式。advice中可以获取 annotation 对象。

bean - 仅在Spring AOP中支持

**由于Spring的AOP框架的基于代理的性质，目标对象内的调用根本没有被拦截**。对于JDK代理，只有代理上的公共接口方法调用才能被拦截。使用CGLIB，代理上的public和protected方法调用将被拦截，如果需要的话，甚至包package-visible方法。但是，通过代理的常见交互应始终通过 public 签名进行设计。

请注意，pointcut 定义通常与所有截取的方法匹配。如果一个切入点严格意义上的 public-only，即使在可能存在 non-public 交互 CGLIB代理场景中，需要相应地定义。

**如果拦截需求包含方法调用，甚至包含目标类中的构造函数，请考虑使用Spring驱动的本机AspectJ编织，而不是Spring的基于代理的AOP框架**。这构成了不同特征的AOP使用方式，所以在做出决定之前一定要先熟悉编织。

切入点表达式可以使用'&&'，'||' 和'!'组合。用更小的命名组件构建更复杂的切入点表达式是一种最佳做法。 

当按名称引用切入点时，将应用普通的Java可见性规则（您可以看到相同类型的私有切入点，层次结构中受保护的切入点，任何位置的公共切入点等）。 可见性不影响切入点匹配。

在使用企业应用程序时，您经常要从几个方面参考应用程序的模块和特定的一组操作。 我们建议定义一个“SystemArchitecture”方面来捕获常见的切入点表达式。

pointcuts优化，AspectJ在编译阶段会优化匹配。匹配分为静态匹配和动态匹配。动态匹配只有在代码运行阶段才能判定是否匹配。并以DNF形式进行组织排序，将开销小的评估放在前面。这意味着你无需关心pointcut的排序问题。

pointcuts分三类：
- kinded    选择一种特定类型的连接点。例如：execution, get, set, call, handler
- scoping   选择一组感兴趣的连接点（可能有多种）。例如：within, withincode
- context   匹配（和可选地绑定）基于上下文。 例如：this, target, @annotation
一个写得好的切入点应至少包括前两种类型（kinded和scoping），如果希望基于连接点context进行匹配，则可以同事包含上下文标识符，或者将该上下文绑定以用于建议。 只提供一个指定的指示符或仅指定一个上下文指示符将会起作用，但是会由于所有额外的处理和分析而影响编织性能（使用时间和内存）。 范围标识符的匹配速度非常快，而且它们的使用方式意味着AspectJ可以很快地解除不应该进一步处理的连接点组 - 这就是为什么一个好的切入点应该总是包含一个可能的情况。

# 5.2.4 Declaring advice
```java
        @Before("execution(* com.xyz.myapp.dao.*.*(..))")
        public void doAccessCheck() {
        }
```

```java
        @AfterReturning(
                pointcut="com.xyz.myapp.SystemArchitecture.dataAccessOperation()",
                returning="retVal")
        public void doAccessCheck(Object retVal) {
        }
```
returning与advice方法中的参数类型，一起参与匹配。

```java
        @AfterThrowing(
                pointcut="com.xyz.myapp.SystemArchitecture.dataAccessOperation()",
                throwing="ex")
        public void doRecoveryActions(DataAccessException ex) {
        }

```
throwing与advice方法中的参数类型，一起参与匹配。

```java
        @After("com.xyz.myapp.SystemArchitecture.dataAccessOperation()")
        public void doReleaseLock() {
        }
```

```java
        @Around("com.xyz.myapp.SystemArchitecture.businessService()")
        public Object doBasicProfiling(ProceedingJoinPoint pjp) throws Throwable {
                // start stopwatch
                Object retVal = pjp.proceed();
                // stop stopwatch
                return retVal;
        }

```
第一个参数必须是ProceedingJoinPoint类型。
Spring AOP 于 AspectJ 的proceed参数绑定不一样。

任何Advice方法的第一个参数都可以是JoinPoint类型（@Around是子类型ProceedingJoinPoint）

```java
@Before("com.xyz.myapp.SystemArchitecture.dataAccessOperation() && args(account,..)")
public void validateAccount(Account account) {
        // ...
}
```
```java
@Pointcut("com.xyz.myapp.SystemArchitecture.dataAccessOperation() && args(account,..)")
private void accountDataAccessOperation(Account account) {}

@Before("accountDataAccessOperation(account)")
public void validateAccount(Account account) {
        // ...
}

```
使用args可以绑定参数与参数类型
The proxy object ( this), target object ( target), and annotations ( @within, @target, @annotation, @args)  可以用同样的方式绑定

可使用 args 限定泛型 T 类型。但不支持泛型集合Collection<T>。你必须指定Advice中的参数为Collection<?>类型。然后在方法中自己判断。

参数名称绑定。如果有多个参数需要使用argNames来指定参数名称。旧版的java反射无法获取方法中的参数名称。


