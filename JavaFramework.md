# Spring基础知识

## 什么是Spring

Spring是个包含一系列功能的合集，如快速开发的Spring Boot，支持微服务的Spring Cloud，支持认证与鉴权的Spring Security，Web框架Spring MVC。IOC与AOP依然是核心。

## Spring MVC流程

1. 发送请求——>DispatcherServlet拦截器拿到交给HandlerMapping
2. 依次调用配置的拦截器，最后找到配置好的业务代码Handler并执行业务方法
3. 包装成ModelAndView返回给ViewResolver解析器渲染页面

## 解决循环依赖

无参数构造器、字段注入

## Bean的生命周期

1. Spring对Bean进行实例化。
2. Spring将值和Bean的引用注入进Bean对应的属性中。
3. 容器通过Aware接口把容器信息注入Bean。
4. BeanPostProcessor。进行进一步的构造，会在InitialzationBean前后执行对应方法，当前正在初始化的bean对象会被传递进来，我们就可以对这个bean作任何处理。
5. InitializingBean。这一阶段也可以在bean正式构造完成前增加我们自定义的逻辑，但它与前置处理不同，由于该函数并不会把当前bean对象传进来，因此在这一步没办法处理对象本身，只能增加一些额外的逻辑。
6. DisposableBean。Bean将一直驻留在应用上下文中给应用使用，直到应用上下文被销毁，如果Bean实现了接口，Spring将调用它的destory方法。

## Bean的作用域

* singleton：单例模式，Spring IoC容器中只会存在一个共享的Bean实例，无论有多少个Bean引用它，始终指向同一对象。
* prototype：原型模式，每次通过Spring容器获取prototype定义的bean时，容器都将创建一个新的Bean实例，每个Bean实例都有自己的属性和状态。
* request：在一次Http请求中，容器会返回该Bean的同一实例。而对不同的Http请求则会产生新的Bean，而且该bean仅在当前Http Request内有效。
* session：在一次Http Session中，容器会返回该Bean的同一实例。而对不同的Session请求则会创建新的实例，该bean实例仅在当前Session内有效。
* global Session：在一个全局的Http Session中，容器会返回该Bean的同一个实例，仅在使用portlet context时有效。

## IOC（DI）

* 控制反转

由 Spring IOC 容器来负责对象的生命周期和对象之间的关系。IoC 容器控制对象的创建，依赖对象的获取被反转了没有 IoC 的时候我们都是在自己对象中主动去创建被依赖的对象，这是正转。但是有了 IoC 后，所依赖的对象直接由 IoC 容器创建后注入到被注入的对象中，依赖的对象由原来的主动获取变成被动接受，所以是反转。

* 依赖注入

组件之间依赖关系由容器在运行期决定，由容器动态的将某个依赖关系注入到组件之中，提升组件重用的频率、灵活、可扩展。
通过依赖注入机制，我们只需要通过简单的配置，而无需任何代码就可指定目标需要的资源，完成自身的业务逻辑，而不需要关心具体的资源来自何处，由谁实现。

注入方式：构造器注入、setter 方法注入、接口方式注入

## Spring AOP

### 介绍

面向切面的编程，是一种编程技术，是OOP（面向对象编程）的补充和完善。OOP的执行是一种从上往下的流程，并没有从左到右的关系。因此在OOP编程中，会有大量的重复代码。而AOP则是将这些与业务无关的重复代码抽取出来，然后再嵌入到业务代码当中。常见的应用有：权限管理、日志、事务管理等。

### 实现方式

实现AOP的技术，主要分为两大类：一是采用动态代理技术，利用截取消息的方式，对该消息进行装饰，以取代原有对象行为的执行；二是采用静态织入的方式，引入特定的语法创建“方面”，从而使得编译器可以在编译期间织入有关“方面”的代码。Spring AOP实现用的是动态代理的方式。

### Spring AOP使用的动态代理原理

jdk反射：通过反射机制生成代理类的字节码文件，调用具体方法前调用InvokeHandler来处理cglib工具：利用asm开源包，对代理对象类的class文件加载进来，通过修改其字节码生成子类来处理。

1. 如果目标对象实现了接口，默认情况下会采用JDK的动态代理实现AOP
2. 如果目标对象实现了接口，可以强制使用CGLIB实现AOP
3. 如果目标对象没有实现了接口，必须采用CGLIB库，spring会自动在JDK动态代理和CGLIB之间转换

# Spring面试问题

## 区分 BeanFactory 和 ApplicationContext

| BeanFactory                | ApplicationContext       |
| -------------------------- | ------------------------ |
| 它使用懒加载               | 它使用即时加载           |
| 它使用语法显式提供资源对象 | 它自己创建和管理资源对象 |
| 不支持国际化               | 支持国际化               |
| 不支持基于依赖的注解       | 支持基于依赖的注解       |

BeanFactory和ApplicationContext的优缺点分析：

BeanFactory的优缺点：

* 优点：应用启动的时候占用资源很少，对资源要求较高的应用，比较有优势；
* 缺点：运行速度会相对来说慢一些。而且有可能会出现空指针异常的错误，而且通过Bean工厂创建的Bean生命周期会简单一些。

ApplicationContext的优缺点：

* 优点：所有的Bean在启动的时候都进行了加载，系统运行的速度快；在系统启动的时候，可以发现系统中的配置问题。
* 缺点：把费时的操作放到系统启动中完成，所有的对象都可以预加载，缺点就是内存占用较大。

## 区分构造函数注入和 setter 注入

| 构造函数注入               | setter 注入                |
| -------------------------- | -------------------------- |
| 没有部分注入               | 有部分注入                 |
| 不会覆盖 setter 属性       | 会覆盖 setter 属性         |
| 任意修改都会创建一个新实例 | 任意修改不会创建一个新实例 |
| 适用于设置很多属性         | 适用于设置少量属性         |

## spring 提供了哪些配置方式

* 基于 xml 配置

bean 所需的依赖项和服务在 XML 格式的配置文件中指定。这些配置文件通常包含许多 bean 定义和特定于应用程序的配置选项。它们通常以 bean 标签开头。例如：

```
<bean id="studentbean" class="org.edureka.firstSpring.StudentBean">
 <property name="name" value="Edureka"></property>
</bean>
```

* 基于注解配置

您可以通过在相关的类，方法或字段声明上使用注解，将 bean 配置为组件类本身，而不是使用 XML 来描述 bean 装配。默认情况下，Spring 容器中未打开注解装配。因此，您需要在使用它之前在 Spring 配置文件中启用它。例如：

```
<beans>
<context:annotation-config/>
<!-- bean definitions go here -->
</beans>
```

* 基于 Java API 配置

Spring 的 Java 配置是通过使用 @Bean 和 @Configuration 来实现。

1. @Bean 注解扮演与 `<bean />` 元素相同的角色。
2. @Configuration 类允许通过简单地调用同一个类中的其他 @Bean 方法来定义 bean 间依赖关系。

例如：

```java
@Configuration
public class StudentConfig {
    @Bean
    public StudentBean myStudent() {
        return new StudentBean();
    }
}
```

## Spring 中的 bean 的作用域有哪些

* singleton : 唯一 bean 实例，Spring 中的 bean 默认都是单例的。
* prototype : 每次请求都会创建一个新的 bean 实例。
* request : 每一次HTTP请求都会产生一个新的bean，该bean仅在当前HTTP request内有效。
* session : ：在一个HTTP Session中，一个Bean定义对应一个实例。该作用域仅在基于web的Spring ApplicationContext情形下有效。
* global-session： 全局session作用域，仅仅在基于portlet的web应用中才有意义，Spring5已经没有了。Portlet是能够生成语义代码(例如：HTML)片段的小型Java Web插件。它们基于portlet容器，可以像servlet一样处理HTTP请求。但是，与 servlet 不同，每个 portlet 都有不同的会话。

## 将一个类声明为Spring的 bean 的注解有哪些

我们一般使用 @Autowired 注解自动装配 bean，要想把类标识成可用于 @Autowired 注解自动装配的 bean 的类,采用以下注解可实现：

* @Component ：通用的注解，可标注任意类为 Spring 组件。如果一个Bean不知道属于哪个层，可以使用@Component 注解标注。 8 @Repository : 对应持久层即 Dao 层，主要用于数据库相关操作。
* @Service : 对应服务层，主要涉及一些复杂的逻辑，需要用到 Dao层。
* @Controller : 对应 Spring MVC 控制层，主要用户接受用户请求并调用 Service 层返回数据给前端页面。

## spring 支持几种 bean scope

Spring bean 支持 5 种 scope：

* **Singleton** - 每个 Spring IoC 容器仅有一个单实例。
* **Prototype** - 每次请求都会产生一个新的实例。
* **Request** - 每一次 HTTP 请求都会产生一个新的实例，并且该 bean 仅在当前 HTTP 请求内有效。
* **Session** - 每一次 HTTP 请求都会产生一个新的 bean，同时该 bean 仅在当前 HTTP session 内有效。
* **Global-session** - 类似于标准的 HTTP Session 作用域，不过它仅仅在基于 portlet 的 web 应用中才有意义。Portlet 规范定义了全局 Session 的概念，它被所有构成某个 portlet web 应用的各种不同的 portlet 所共享。在 global session 作用域中定义的 bean 被限定于全局 portlet Session 的生命周期范围内。如果你在 web 中使用 global session 作用域来标识 bean，那么 web 会自动当成 session 类型来使用。

仅当用户使用支持 Web 的 ApplicationContext 时，最后三个才可用。

## Spring 中的 bean 生命周期

Bean的生命周期是由容器来管理的。主要在创建和销毁两个时期。

[![](https://camo.githubusercontent.com/99c0998afcbe4020167c8dad954102f590fe1e18da5b3e906fd6ace61da453e6/687474703a2f2f626c6f672d696d672e636f6f6c73656e2e636e2f696d672f313538333637353039303634315f35312e706e67)](https://camo.githubusercontent.com/99c0998afcbe4020167c8dad954102f590fe1e18da5b3e906fd6ace61da453e6/687474703a2f2f626c6f672d696d672e636f6f6c73656e2e636e2f696d672f313538333637353039303634315f35312e706e67)

### 创建过程

1，实例化bean对象，以及设置bean属性。
2，如果通过Aware接口声明了依赖关系，则会注入Bean对容器基础设施层面的依赖，Aware接口是为了感知到自身的一些属性。容器管理的Bean一般不需要知道容器的状态和直接使用容器。但是在某些情况下是需要在Bean中对IOC容器进行操作的。这时候需要在bean中设置对容器的感知。SpringIOC容器也提供了该功能，它是通过特定的Aware接口来完成的。 比如BeanNameAware接口，可以知道自己在容器中的名字。 如果这个Bean已经实现了BeanFactoryAware接口，可以用这个方式来获取其它Bean。 （如果Bean实现了BeanNameAware接口，调用setBeanName()方法，传入Bean的名字。 如果Bean实现了BeanClassLoaderAware接口，调用setBeanClassLoader()方法，传入ClassLoader对象的实例。 如果Bean实现了BeanFactoryAware接口，调用setBeanFactory()方法，传入BeanFactory对象的实例。）
3，紧接着会调用BeanPostProcess的前置初始化方法postProcessBeforeInitialization，主要作用是在Spring完成实例化之后，初始化之前，对Spring容器实例化的Bean添加自定义的处理逻辑。有点类似于AOP。 4，如果实现了BeanFactoryPostProcessor接口的afterPropertiesSet方法，做一些属性被设定后的自定义的事情。 5，调用Bean自身定义的init方法，去做一些初始化相关的工作。 6，调用BeanPostProcess的后置初始化方法，postProcessAfterInitialization去做一些bean初始化之后的自定义工作。 7，完成以上创建之后就可以在应用里使用这个Bean了。

### 销毁过程

当Bean不再用到，便要销毁 1，若实现了DisposableBean接口，则会调用destroy方法； 2，若配置了destry-method属性，则会调用其配置的销毁方法；

## 什么是 spring 的内部 bean

只有将 bean 用作另一个 bean 的属性时，才能将 bean 声明为内部 bean。为了定义 bean，Spring 的基于 XML 的配置元数据在 `<property>` 或 `<constructor-arg>` 中提供了 `<bean>` 元素的使用。内部 bean 总是匿名的，它们总是作为原型。

例如，假设我们有一个 Student 类，其中引用了 Person 类。这里我们将只创建一个 Person 类实例并在 Student 中使用它。

Student.java

```java
public class Student {
    private Person person;
    //Setters and Getters
}
public class Person {
    private String name;
    private String address;
    //Setters and Getters
}
```

bean.xml

```
<bean id=“StudentBean" class="com.edureka.Student">
    <property name="person">
        <!--This is inner bean -->
        <bean class="com.edureka.Person">
            <property name="name" value=“Scott"></property>
            <property name="address" value=“Bangalore"></property>
        </bean>
    </property>
</bean>
```

## 什么是 spring 装配

当 bean 在 Spring 容器中组合在一起时，它被称为装配或 bean 装配。 Spring 容器需要知道需要什么 bean 以及容器应该如何使用依赖注入来将 bean 绑定在一起，同时装配 bean。

Spring 容器能够自动装配 bean。也就是说，可以通过检查 BeanFactory 的内容让 Spring 自动解析 bean 的协作者。

自动装配的不同模式：

* **no** - 这是默认设置，表示没有自动装配。应使用显式 bean 引用进行装配。
* **byName** - 它根据 bean 的名称注入对象依赖项。它匹配并装配其属性与 XML 文件中由相同名称定义的 bean。
* **byType** - 它根据类型注入对象依赖项。如果属性的类型与 XML 文件中的一个 bean 名称匹配，则匹配并装配属性。
* **构造函数** - 它通过调用类的构造函数来注入依赖项。它有大量的参数。
* **autodetect** - 首先容器尝试通过构造函数使用 autowire 装配，如果不能，则尝试通过 byType 自动装配。

## 自动装配有什么局限

* 覆盖的可能性 - 您始终可以使用 `<constructor-arg>` 和 `<property>` 设置指定依赖项，这将覆盖自动装配。
* 基本元数据类型 - 简单属性（如原数据类型，字符串和类）无法自动装配。
* 令人困惑的性质 - 总是喜欢使用明确的装配，因为自动装配不太精确。

## Spring中出现同名bean怎么办

* 同一个配置文件内同名的Bean，以最上面定义的为准。
* 不同配置文件中存在同名Bean，后解析的配置文件会覆盖先解析的配置文件。
* 同文件中ComponentScan和@Bean出现同名Bean。同文件下@Bean的会生效，@ComponentScan扫描进来不会生效。通过@ComponentScan扫描进来的优先级是最低的，原因就是它扫描进来的Bean定义是最先被注册的。

## Spring 怎么解决循环依赖问题

spring对循环依赖的处理有三种情况： 

* 构造器的循环依赖：这种依赖spring是处理不了的，直 接抛出BeanCurrentlylnCreationException异常。
* 单例模式下的setter循环依赖：通过“三级缓存”处理循环依赖。
* 非单例循环依赖：无法处理。

下面分析单例模式下的setter循环依赖如何解决

[![img](https://camo.githubusercontent.com/53efc926f02b47165ce21f20b3827372e7e7d877b38c73b5408a980663442293/687474703a2f2f626c6f672d696d672e636f6f6c73656e2e636e2f696d672f313538343736313431333334315f31322e706e67)](https://camo.githubusercontent.com/53efc926f02b47165ce21f20b3827372e7e7d877b38c73b5408a980663442293/687474703a2f2f626c6f672d696d672e636f6f6c73656e2e636e2f696d672f313538343736313431333334315f31322e706e67)

Spring的单例对象的初始化主要分为三步：

* createBeanInstance：实例化，其实也就是调用对象的构造方法实例化对象。
* populateBean：填充属性，这一步主要是多bean的依赖属性进行填充。
* initializeBean：调用spring xml中的init 方法。

从上面讲述的单例bean初始化步骤我们可以知道，循环依赖主要发生在第一、第二部。也就是构造器循环依赖和field循环依赖。

[![](https://camo.githubusercontent.com/5cf0aeb4b8eac9eae7313f5449d8a4014082e58df39ddc5ae8e6bd894d9c93b3/687474703a2f2f626c6f672d696d672e636f6f6c73656e2e636e2f696d672f313538343735383330393631365f31302e706e67)](https://camo.githubusercontent.com/5cf0aeb4b8eac9eae7313f5449d8a4014082e58df39ddc5ae8e6bd894d9c93b3/687474703a2f2f626c6f672d696d672e636f6f6c73656e2e636e2f696d672f313538343735383330393631365f31302e706e67)

举例：A的某个field或者setter依赖了B的实例对象，同时B的某个field或者setter依赖了A的实例对象”这种循环依赖的情况。

A首先完成了初始化的第一步（createBeanINstance实例化），并且将自己提前曝光到singletonFactories中。

此时进行初始化的第二步，发现自己依赖对象B，此时就尝试去get(B)，发现B还没有被create，所以走create流程，B在初始化第一步的时候发现自己依赖了对象A，于是尝试get(A)，尝试一级缓存singletonObjects(肯定没有，因为A还没初始化完全)，尝试二级缓存earlySingletonObjects（也没有），尝试三级缓存singletonFactories，由于A通过ObjectFactory将自己提前曝光了，所以B能够通过

ObjectFactory.getObject拿到A对象(虽然A还没有初始化完全，但是总比没有好呀)，B拿到A对象后顺利完成了初始化阶段1、2、3，完全初始化之后将自己放入到一级缓存singletonObjects中。

此时返回A中，A此时能拿到B的对象顺利完成自己的初始化阶段2、3，最终A也完成了初始化，进去了一级缓存singletonObjects中，而且更加幸运的是，由于B拿到了A的对象引用，所以B现在hold住的A对象完成了初始化。

## Spring 中的单例 bean 的线程安全问题

当多个用户同时请求一个服务时，容器会给每一个请求分配一个线程，这时多个线程会并发执行该请求对应的业务逻辑（成员方法），此时就要注意了，如果该处理逻辑中有对单例状态的修改（体现为该单例的成员属性），则必须考虑线程同步问题。 **线程安全问题都是由全局变量及静态变量引起的。** 若每个线程中对全局变量、静态变量只有读操作，而无写操作，一般来说，这个全局变量是线程安全的；若有多个线程同时执行写操作，一般都需要考虑线程同步，否则就可能影响线程安全。

**无状态bean和有状态bean**

* 有状态就是有数据存储功能。有状态对象(Stateful Bean)，就是有实例变量的对象，可以保存数据，是非线程安全的。在不同方法调用间不保留任何状态。
* 无状态就是一次操作，不能保存数据。无状态对象(Stateless Bean)，就是没有实例变量的对象 .不能保存数据，是不变类，是线程安全的。

在spring中无状态的Bean适合用不变模式，就是单例模式，这样可以共享实例提高性能。有状态的Bean在多线程环境下不安全，适合用Prototype原型模式。 Spring使用ThreadLocal解决线程安全问题。如果你的Bean有多种状态的话（比如 View Model 对象），就需要自行保证线程安全 。

## AOP 有哪些实现方式

实现 AOP 的技术，主要分为两大类：

* 静态代理 - 指使用 AOP 框架提供的命令进行编译，从而在编译阶段就可生成 AOP 代理类，因此也称为编译时增强。
  * 编译时编织（特殊编译器实现）。
  * 类加载时编织（特殊的类加载器实现）。
* 动态代理 - 在运行时在内存中“临时”生成 AOP 动态代理类，因此也被称为运行时增强。
  * `JDK` 动态代理：通过反射来接收被代理的类，并且要求被代理的类必须实现一个接口 。JDK 动态代理的核心是 InvocationHandler 接口和 Proxy 类 。
  * `CGLIB`动态代理： 如果目标类没有实现接口，那么 `Spring AOP` 会选择使用 `CGLIB` 来动态代理目标类 。`CGLIB` （ Code Generation Library ），是一个代码生成的类库，可以在运行时动态的生成某个类的子类，注意， `CGLIB` 是通过继承的方式做的动态代理，因此如果某个类被标记为 `final` ，那么它是无法使用 `CGLIB` 做动态代理的。

## Spring AOP and AspectJ AOP 有什么区别

Spring AOP 基于动态代理方式实现；AspectJ 基于静态代理方式实现。 Spring AOP 仅支持方法级别的 PointCut；提供了完全的 AOP 支持，它还支持属性级别的 PointCut。

## Spring 框架中用到了哪些设计模式

**工厂设计模式** : Spring使用工厂模式通过 `BeanFactory`、`ApplicationContext` 创建 bean 对象。

**代理设计模式** : Spring AOP 功能的实现。

**单例设计模式** : Spring 中的 Bean 默认都是单例的。

**模板方法模式** : Spring 中 `jdbcTemplate`、`hibernateTemplate` 等以 Template 结尾的对数据库操作的类，它们就使用到了模板模式。

**包装器设计模式** : 我们的项目需要连接多个数据库，而且不同的客户在每次访问中根据需要会去访问不同的数据库。这种模式让我们可以根据客户的需求能够动态切换不同的数据源。

**观察者模式:** Spring 事件驱动模型就是观察者模式很经典的一个应用。

**适配器模式** :Spring AOP 的增强或通知(Advice)使用到了适配器模式、spring MVC 中也是用到了适配器模式适配 `Controller`。

## Spring 事务实现方式有哪些

* 编程式事务管理：这意味着你可以通过编程的方式管理事务，这种方式带来了很大的灵活性，但很难维护。
* 声明式事务管理：这种方式意味着你可以将事务管理和业务代码分离。你只需要通过注解或者XML配置管理事务。

## Spring框架的事务管理有哪些优点

* 它提供了跨不同事务api（如JTA、JDBC、Hibernate、JPA和JDO）的一致编程模型。
* 它为编程事务管理提供了比JTA等许多复杂事务API更简单的API。
* 它支持声明式事务管理。
* 它很好地集成了Spring的各种数据访问抽象。

## Spring事务定义的传播规则

* PROPAGATION_REQUIRED: 支持当前事务，如果当前没有事务，就新建一个事务。这是最常见的选择。
* PROPAGATION_SUPPORTS: 支持当前事务，如果当前没有事务，就以非事务方式执行。
* PROPAGATION_MANDATORY: 支持当前事务，如果当前没有事务，就抛出异常。
* PROPAGATION_REQUIRES_NEW: 新建事务，如果当前存在事务，把当前事务挂起。
* PROPAGATION_NOT_SUPPORTED: 以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。
* PROPAGATION_NEVER: 以非事务方式执行，如果当前存在事务，则抛出异常。
* PROPAGATION_NESTED:如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则进行与PROPAGATION_REQUIRED类似的操作。

## SpringMVC 工作原理了解吗

**原理如下图所示：**

[![img](https://camo.githubusercontent.com/6143a35ece8cc1efdbadedc0e5e0ed28793972c7a41f7994050447e22bce081b/687474703a2f2f626c6f672d696d672e636f6f6c73656e2e636e2f696d672f5370696e674d56432d50726f636573732e6a7067)](https://camo.githubusercontent.com/6143a35ece8cc1efdbadedc0e5e0ed28793972c7a41f7994050447e22bce081b/687474703a2f2f626c6f672d696d672e636f6f6c73656e2e636e2f696d672f5370696e674d56432d50726f636573732e6a7067)

上图的一个笔误的小问题：Spring MVC 的入口函数也就是前端控制器 `DispatcherServlet` 的作用是接收请求，响应结果。

**流程说明（重要）：**

1. 客户端（浏览器）发送请求，直接请求到 `DispatcherServlet`。
2. `DispatcherServlet` 根据请求信息调用 `HandlerMapping`，解析请求对应的 `Handler`。
3. 解析到对应的 `Handler`（也就是我们平常说的 `Controller` 控制器）后，开始由 `HandlerAdapter` 适配器处理。
4. `HandlerAdapter` 会根据 `Handler`来调用真正的处理器开处理请求，并处理相应的业务逻辑。
5. 处理器处理完业务后，会返回一个 `ModelAndView` 对象，`Model` 是返回的数据对象，`View` 是个逻辑上的 `View`。
6. `ViewResolver` 会根据逻辑 `View` 查找实际的 `View`。
7. `DispaterServlet` 把返回的 `Model` 传给 `View`（视图渲染）。
8. 把 `View` 返回给请求者（浏览器）

## 简单介绍 Spring MVC 的核心组件

那么接下来就简单介绍一下 `DispatcherServlet` 和九大组件（按使用顺序排序的）：

| 组件                        | 说明                                                                                                                                                                                                                                                                                                                                                            |
| :-------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| DispatcherServlet           | Spring MVC 的核心组件，是请求的入口，负责协调各个组件工作                                                                                                                                                                                                                                                                                                       |
| MultipartResolver           | 内容类型(`Content-Type` )为 `multipart/*` 的请求的解析器，例如解析处理文件上传的请求，便于获取参数信息以及上传的文件                                                                                                                                                                                                                                        |
| HandlerMapping              | 请求的处理器匹配器，负责为请求找到合适的 `HandlerExecutionChain` 处理器执行链，包含处理器（`handler`）和拦截器们（`interceptors`）                                                                                                                                                                                                                        |
| HandlerAdapter              | 处理器的适配器。因为处理器 `handler` 的类型是 Object 类型，需要有一个调用者来实现 `handler` 是怎么被执行。Spring 中的处理器的实现多变，比如用户处理器可以实现 Controller 接口、HttpRequestHandler 接口，也可以用 `@RequestMapping` 注解将方法作为一个处理器等，这就导致 Spring MVC 无法直接执行这个处理器。所以这里需要一个处理器适配器，由它去执行处理器 |
| HandlerExceptionResolver    | 处理器异常解析器，将处理器（`handler` ）执行时发生的异常，解析( 转换 )成对应的 ModelAndView 结果                                                                                                                                                                                                                                                              |
| RequestToViewNameTranslator | 视图名称转换器，用于解析出请求的默认视图名                                                                                                                                                                                                                                                                                                                      |
| LocaleResolver              | 本地化（国际化）解析器，提供国际化支持                                                                                                                                                                                                                                                                                                                          |
| ThemeResolver               | 主题解析器，提供可设置应用整体样式风格的支持                                                                                                                                                                                                                                                                                                                    |
| ViewResolver                | 视图解析器，根据视图名和国际化，获得最终的视图 View 对象                                                                                                                                                                                                                                                                                                        |
| FlashMapManager             | FlashMap 管理器，负责重定向时，保存参数至临时存储（默认 Session）                                                                                                                                                                                                                                                                                               |

Spring MVC 对各个组件的职责划分的比较清晰。`DispatcherServlet` 负责协调，其他组件则各自做分内之事，互不干扰。

## @Controller 注解有什么用

`@Controller` 注解标记一个类为 Spring Web MVC **控制器** Controller。Spring MVC 会将扫描到该注解的类，然后扫描这个类下面带有 `@RequestMapping` 注解的方法，根据注解信息，为这个方法生成一个对应的**处理器**对象，在上面的 HandlerMapping 和 HandlerAdapter组件中讲到过。

当然，除了添加 `@Controller` 注解这种方式以外，你还可以实现 Spring MVC 提供的 `Controller` 或者 `HttpRequestHandler` 接口，对应的实现类也会被作为一个**处理器**对象。

## @RequestMapping 注解有什么用

`@RequestMapping` 注解，在上面已经讲过了，配置**处理器**的 HTTP 请求方法，URI等信息，这样才能将请求和方法进行映射。这个注解可以作用于类上面，也可以作用于方法上面，在类上面一般是配置这个**控制器**的 URI 前缀。

## @RestController 和 @Controller 有什么区别

`@RestController` 注解，在 `@Controller` 基础上，增加了 `@ResponseBody` 注解，更加适合目前前后端分离的架构下，提供 Restful API ，返回例如 JSON 数据格式。当然，返回什么样的数据格式，根据客户端的 `ACCEPT` 请求头来决定。

## @RequestMapping 和 @GetMapping 注解的不同之处在哪里

1. `@RequestMapping`：可注解在类和方法上；`@GetMapping` 仅可注册在方法上。
2. `@RequestMapping`：可进行 GET、POST、PUT、DELETE 等请求方法；`@GetMapping` 是 `@RequestMapping` 的 GET 请求方法的特例，目的是为了提高清晰度。

## @RequestParam 和 @PathVariable 两个注解的区别

两个注解都用于方法参数，获取参数值的方式不同，`@RequestParam` 注解的参数从请求携带的参数中获取，而 `@PathVariable` 注解从请求的 URI 中获取。

## 返回 JSON 格式使用什么注解

可以使用 **`@ResponseBody`** 注解，或者使用包含 `@ResponseBody` 注解的 **`@RestController`** 注解。

当然，还是需要配合相应的支持 JSON 格式化的 HttpMessageConverter 实现类。例如，Spring MVC 默认使用 MappingJackson2HttpMessageConverter。

## 什么是springmvc拦截器以及如何使用它

Spring的处理程序映射机制包括处理程序拦截器，当你希望将特定功能应用于某些请求时，例如，检查用户主题时，这些拦截器非常有用。拦截器必须实现org.springframework.web.servlet包的HandlerInterceptor。此接口定义了三种方法：

* preHandle：在执行实际处理程序之前调用。
* postHandle：在执行完实际程序之后调用。
* afterCompletion：在完成请求后调用。

## Spring MVC 和 Struts2 的异同

**入口**不同

* Spring MVC 的入门是一个 Servlet  **控制器** 。
* Struts2 入门是一个 Filter  **过滤器** 。

**配置映射**不同，

* Spring MVC 是基于**方法**开发，传递参数是通过 **方法形参** ，一般设置为 **单例** 。
* Struts2 是基于**类**开发，传递参数是通过 **类的属性** ，只能设计为 **多例** 。

**视图**不同

* Spring MVC 通过参数解析器是将 Request 对象内容进行解析成方法形参，将响应数据和页面封装成 **ModelAndView** 对象，最后又将模型数据通过 **Request** 对象传输到页面。其中，如果视图使用 JSP 时，默认使用 **JSTL** 。
* Struts2 采用**值栈**存储请求和响应的数据，通过 **OGNL** 存取数据。

## REST 代表着什么

REST 代表着抽象状态转移，它是根据 HTTP 协议从客户端发送数据到服务端，例如：服务端的一本书可以以 XML 或 JSON 格式传递到客户端。

可以看看 [REST API design and development](http://bit.ly/2zIGzWK) ，知乎上的 [《怎样用通俗的语言解释 REST，以及 RESTful？》](https://www.zhihu.com/question/28557115)了解。

## 什么是安全的 REST 操作

REST 接口是通过 HTTP 方法完成操作

* 一些 HTTP 操作是安全的，如 GET 和 HEAD ，它不能在服务端修改资源。
* 换句话说，PUT、POST 和 DELETE 是不安全的，因为他们能修改服务端的资源。

所以，是否安全的界限，在于**是否修改**服务端的资源。

## REST API 是无状态的吗

 **是的** ，REST API 应该是无状态的，因为它是基于 HTTP 的，它也是无状态的。

REST API 中的请求应该包含处理它所需的所有细节。它**不应该**依赖于以前或下一个请求或服务器端维护的一些数据，例如会话。

## REST安全吗? 你能做什么来保护它

安全是一个宽泛的术语。它可能意味着消息的安全性，这是通过认证和授权提供的加密或访问限制提供的。

REST 通常不是安全的，需要开发人员自己实现安全机制。

## 为什么要用SpringBoot

在使用Spring框架进行开发的过程中，需要配置很多Spring框架包的依赖，如spring-core、spring-bean、spring-context等，而这些配置通常都是重复添加的，而且需要做很多框架使用及环境参数的重复配置，如开启注解、配置日志等。Spring Boot致力于弱化这些不必要的操作，提供默认配置，当然这些默认配置是可以按需修改的，快速搭建、开发和运行Spring应用。

以下是使用SpringBoot的一些好处：

* 自动配置，使用基于类路径和应用程序上下文的智能默认值，当然也可以根据需要重写它们以满足开发人员的需求。
* 创建Spring Boot Starter 项目时，可以选择选择需要的功能，Spring Boot将为你管理依赖关系。
* SpringBoot项目可以打包成jar文件。可以使用Java-jar命令从命令行将应用程序作为独立的Java应用程序运行。
* 在开发web应用程序时，springboot会配置一个嵌入式Tomcat服务器，以便它可以作为独立的应用程序运行。（Tomcat是默认的，当然你也可以配置Jetty或Undertow）
* SpringBoot包括许多有用的非功能特性（例如安全和健康检查）。

## Spring Boot中如何实现对不同环境的属性配置文件的支持

Spring Boot支持不同环境的属性配置文件切换，通过创建application-{profile}.properties文件，其中{profile}是具体的环境标识名称，例如：application-dev.properties用于开发环境，application-test.properties用于测试环境，application-uat.properties用于uat环境。如果要想使用application-dev.properties文件，则在application.properties文件中添加spring.profiles.active=dev。

如果要想使用application-test.properties文件，则在application.properties文件中添加spring.profiles.active=test。

## Spring Boot 的核心注解是哪个？它主要由哪几个注解组成的

启动类上面的注解是@SpringBootApplication，它也是 Spring Boot 的核心注解，主要组合包含了以下 3 个注解：

@SpringBootConfiguration：组合了 @Configuration 注解，实现配置文件的功能。

@EnableAutoConfiguration：打开自动配置的功能，也可以关闭某个自动配置的选项，如关闭数据源自动配置功能： @SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })。

@ComponentScan：Spring组件扫描。

## 你如何理解 Spring Boot 中的 Starters

Starters可以理解为启动器，它包含了一系列可以集成到应用里面的依赖包，你可以一站式集成 Spring 及其他技术，而不需要到处找示例代码和依赖包。如你想使用 Spring JPA 访问数据库，只要加入 spring-boot-starter-data-jpa 启动器依赖就能使用了。

Starters包含了许多项目中需要用到的依赖，它们能快速持续的运行，都是一系列得到支持的管理传递性依赖。

## Spring Boot Starter 的工作原理是什么

Spring Boot 在启动的时候会干这几件事情：

* Spring Boot 在启动时会去依赖的 Starter 包中寻找 resources/META-INF/spring.factories 文件，然后根据文件中配置的 Jar 包去扫描项目所依赖的 Jar 包。
* 根据 spring.factories 配置加载 AutoConfigure 类。
* 根据 @Conditional 注解的条件，进行自动配置并将 Bean 注入 Spring Context。

总结一下，其实就是 Spring Boot 在启动的时候，按照约定去读取 Spring Boot Starter 的配置信息，再根据配置信息对资源进行初始化，并注入到 Spring 容器中。这样 Spring Boot 启动完毕后，就已经准备好了一切资源，使用过程中直接注入对应 Bean 资源即可。

## 保护 Spring Boot 应用有哪些方法

* 在生产中使用HTTPS
* 使用Snyk检查你的依赖关系
* 升级到最新版本
* 启用CSRF保护
* 使用内容安全策略防止XSS攻击

## Spring 、Spring Boot 和 Spring Cloud 的关系

Spring 最初最核心的两大核心功能 Spring Ioc 和 Spring Aop 成就了 Spring，Spring 在这两大核心的功能上不断的发展，才有了 Spring 事务、Spring Mvc 等一系列伟大的产品，最终成就了 Spring 帝国，到了后期 Spring 几乎可以解决企业开发中的所有问题。

Spring Boot 是在强大的 Spring 帝国生态基础上面发展而来，发明 Spring Boot 不是为了取代 Spring ,是为了让人们更容易的使用 Spring 。

Spring Cloud 是一系列框架的有序集合。它利用 Spring Boot 的开发便利性巧妙地简化了分布式系统基础设施的开发，如服务发现注册、配置中心、消息总线、负载均衡、断路器、数据监控等，都可以用 Spring Boot 的开发风格做到一键启动和部署。

Spring Cloud 是为了解决微服务架构中服务治理而提供的一系列功能的开发框架，并且 Spring Cloud 是完全基于 Spring Boot 而开发，Spring Cloud 利用 Spring Boot 特性整合了开源行业中优秀的组件，整体对外提供了一套在微服务架构中服务治理的解决方案。

用一组不太合理的包含关系来表达它们之间的关系。

Spring ioc/aop > Spring > Spring Boot > Spring Cloud

# Mybatis面试问题

## MyBatis是什么

* Mybatis是一个半ORM（对象关系映射）框架，它内部封装了JDBC，加载驱动、创建连接、创建statement等繁杂的过程，开发者开发时只需要关注如何编写SQL语句，可以严格控制sql执行性能，灵活度高。
* 作为一个半ORM框架，MyBatis 可以使用 XML 或注解来配置和映射原生信息，将 POJO映射成数据库中的记录，避免了几乎所有的 JDBC 代码和手动设置参数以及获取结果集。
* 通过xml 文件或注解的方式将要执行的各种 statement 配置起来，并通过java对象和 statement中sql的动态参数进行映射生成最终执行的sql语句，最后由mybatis框架执行sql并将结果映射为java对象并返回。（从执行sql到返回result的过程）。
* 由于MyBatis专注于SQL本身，灵活度高，所以比较适合对性能的要求很高，或者需求变化较多的项目，如互联网项目。

## Mybaits的优缺点

优点

* 基于SQL语句编程，相当灵活，不会对应用程序或者数据库的现有设计造成任何影响，SQL写在XML里，解除sql与程序代码的耦合，便于统一管理；提供XML标签，支持编写动态SQL语句，并可重用。
* 与JDBC相比，减少了50%以上的代码量，消除了JDBC大量冗余的代码，不需要手动开关连接。
* 很好的与各种数据库兼容（因为MyBatis使用JDBC来连接数据库，所以只要JDBC支持的数据库MyBatis都支持）。
* 能够与Spring很好的集成。
* 提供映射标签，支持对象与数据库的ORM字段关系映射；提供对象关系映射标签，支持对象关系组件维护。

缺点

* SQL语句的编写工作量较大，尤其当字段多、关联表多时，对开发人员编写SQL语句的功底有一定要求。
* SQL语句依赖于数据库，导致数据库移植性差，不能随意更换数据库。

## 为什么说Mybatis是半自动ORM映射工具，它与全自动的区别在哪里

Hibernate属于全自动ORM映射工具，使用Hibernate查询关联对象或者关联集合对象时，可以根据对象关系模型直接获取，所以它是全自动的。

而Mybatis在查询关联对象或关联集合对象时，需要手动编写sql来完成，所以，称之为半自动ORM映射工具。

## Hibernate 和 MyBatis 的区别

**相同点**

都是对jdbc的封装，都是持久层的框架，都用于dao层的开发。

**不同点**

* 映射关系

MyBatis 是一个半自动映射的框架，配置Java对象与sql语句执行结果的对应关系，多表关联关系配置简单。

Hibernate 是一个全表映射的框架，配置Java对象与数据库表的对应关系，多表关联关系配置复杂。

* SQL优化和移植性

Hibernate 对SQL语句封装，提供了日志、缓存、级联（级联比 MyBatis 强大）等特性，此外还提供 HQL（Hibernate Query Language）操作数据库，数据库无关性支持好，但会多消耗性能。如果项目需要支持多种数据库，代码开发量少，但SQL语句优化困难。 MyBatis 需要手动编写 SQL，支持动态 SQL、处理列表、动态生成表名、支持存储过程。开发工作量相对大些。直接使用SQL语句操作数据库，不支持数据库无关性，但sql语句优化容易。

* 开发难易程度和学习成本

Hibernate 是重量级框架，学习使用门槛高，适合于需求相对稳定，中小型的项目，比如：办公自动化系统。

MyBatis 是轻量级框架，学习使用门槛低，适合于需求变化频繁，大型的项目，比如：互联网电子商务系统。

**总结**

MyBatis 是一个小巧、方便、高效、简单、直接、半自动化的持久层框架。

Hibernate 是一个强大、方便、高效、复杂、间接、全自动化的持久层框架。

## JDBC编程有哪些不足之处，MyBatis是如何解决这些问题的

* 数据库链接创建、释放频繁造成系统资源浪费从而影响系统性能，如果使用数据库链接池可解决此问题。

解决：在SqlMapConfig.xml中配置数据链接池，使用连接池管理数据库链接。

* Sql语句写在代码中造成代码不易维护，实际应用sql变化的可能较大，sql变动需要改变java代码。

解决：将Sql语句配置在XXXXmapper.xml文件中与java代码分离。

* 向sql语句传参数麻烦，因为sql语句的where条件不一定，可能多也可能少，占位符需要和参数一一对应。

解决： Mybatis自动将java对象映射至sql语句。

* 对结果集解析麻烦，sql变化导致解析代码变化，且解析前需要遍历，如果能将数据库记录封装成pojo对象解析比较方便。

解决：Mybatis自动将sql执行结果映射至java对象。

## #{}和${}的区别

* #{}是占位符，预编译处理；${}是拼接符，字符串替换，没有预编译处理。
* Mybatis在处理#{}时，#{}传入参数是以字符串传入，会将SQL中的#{}替换为?号，调用PreparedStatement的set方法来赋值。
* Mybatis在处理时 ， 是 原 值 传 入 ， 就 是 把 {}时，是原值传入，就是把时，是原值传入，就是把{}替换成变量的值，相当于JDBC中的Statement编译。
* 变量替换后，#{} 对应的变量自动加上单引号 ‘’；变量替换后，${} 对应的变量不会加上单引号 ‘’。
* #{} 可以有效的防止SQL注入，提高系统安全性；${} 不能防止SQL 注入。
* #{} 的变量替换是在DBMS 中；${} 的变量替换是在 DBMS 外。

## 通常一个Xml映射文件，都会写一个Dao接口与之对应，那么这个Dao接口的工作原理是什么。Dao接口里的方法、参数不同时，方法能重载吗

Dao接口即Mapper接口。接口的全限名就是映射文件中的namespace的值；接口的方法名，就是映射文件中Mapper的Statement的id值；接口方法内的参数，就是传递给sql的参数。Mapper接口是没有实现类的，当调用接口方法时，接口全限名+方法名的拼接字符串作为key值，可唯一定位一个MapperStatement。

Dao接口里的方法，是不能重载的，因为是全限名+方法名的保存和寻找策略。

Dao接口的工作原理是JDK动态代理，Mybatis运行时会使用JDK动态代理为Dao接口生成代理proxy对象，代理对象proxy会拦截接口方法，转而执行MappedStatement所代表的sql，然后将sql执行结果返回。

## 在Mapper中如何传递多个参数

* 若Dao层函数有多个参数，那么其对应的xml中，#{0}代表接收的是Dao层中的第一个参数，#{1}代表Dao中的第二个参数，以此类推。
* 使用@Param注解：在Dao层的参数中前加@Param注解,注解内的参数名为传递到Mapper中的参数名。
* 多个参数封装成Map，以HashMap的形式传递到Mapper中。

## Mybatis动态sql有什么用，执行原理是什么，有哪些动态sql

Mybatis动态sql可以在xml映射文件内，以标签的形式编写动态sql，执行原理是根据表达式的值完成逻辑判断，并动态拼接sql的功能。

Mybatis提供了9种动态sql标签：trim、where、set、foreach、if、choose、when、otherwise、bind

## xml映射文件中，不同的xml映射文件id是否可以重复

不同的xml映射文件，如果配置了namespace，那么id可以重复；如果没有配置namespace，那么id不能重复。

原因是namespace+id是作为Map<String,MapperStatement>的key使用的，如果没有namespace，就剩下id，那么id重复会导致数据互相覆盖。有了namespace，自然id就可以重复，namespace不同，namespace+id自然也不同。

## Mybatis实现一对一有几种方式，具体是怎么操作的

有联合查询和嵌套查询两种方式。

联合查询是几个表联合查询，通过在resultMap里面配置association节点配置一对一的类就可以完成。

嵌套查询是先查一个表，根据这个表里面的结果的外键id，再去另外一个表里面查询数据，也是通过association配置，但另外一个表的查询是通过select配置的。

## Mybatis实现一对多有几种方式，具体是怎么操作的

有联合查询和嵌套查询两种方式。

联合查询是几个表联合查询，只查询一次，通过在resultMap里面的collection节点配置一对多的类就可以完成。

嵌套查询是先查一个表，根据这个表里面的结果的外键id，再去另外一个表里面查询数据，也是通过collection，但另外一个表的查询是通过select配置的。

## Mybatis的一级、二级缓存

* 一级缓存：基于PerpetualCache的HashMap本地缓存，其存储作用域为Session，当Session flush或close之后，该Session中的所有Cache就将清空，默认打开一级缓存。
* 二级缓存与一级缓存机制相同，默认也是采用PerpetualCache，HashMap存储，不同在于其存储作用域为Mapper（namespace），并且可自定义存储源，如Ehcache。默认打不开二级缓存，要开启二级缓存，使用二级缓存属性类需要实现Serializable序列化接口（可用来保存对象的状态），可在它的映射文件中配置。

对于缓存数据更新机制，当某一个作用域（一级缓存Session/二级缓存Namespace）进行了增/删/改操作后，默认该作用域下所有select中的缓存将被clear。

## 使用MyBatis的Mapper接口调用时有哪些要求

* Mapper接口方法名和mapper.xml中定义的每个sql的id相同。
* Mapper接口方法的输入参数类型和mapper.xml中定义的每个sql的parameterType类型相同。
* Mapper接口方法的输出参数类型和mapper.xml中定义的每个sql的resultType的类型相同。
* Mapper.xml文件中的namespace即是mapper接口的类路径。
