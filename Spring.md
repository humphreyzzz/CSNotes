# IoC(Inverse of Control)
控制反转，有时也被称为依赖注入，是一种降低对象之间耦合关系的设计思想。   
依赖注入(DI)和控制反转(IOC)是从不同的角度的描述的同一件事情，就是指通过引入IOC容器，利用依赖关系注入的方式，实现对象之间的解耦。   
可以把IOC容器的工作模式看做是工厂模式的升华，可以把IOC容器看作是一个工厂，这个工厂里要生产的对象都在配置文件中给出定义，然后利用编程语言的的反射编程，根据配置文件中给出的类名生成相应的对象。从实现来看，IOC是把以前在工厂方法里写死的对象生成代码，改变为由配置文件来定义，也就是把工厂和对象生成这两者独立分隔开来，目的就是提高灵活性和可维护性。

## 工作机制
web环境下Spring容器、SpringMVC容器启动过程：

1. 首先，对于一个web应用，其部署在web容器中，web容器提供其一个全局的上下文环境，这个上下文就是ServletContext，其为后面的spring IoC容器提供宿主环境；
2. 其次，在web.xml中会提供有contextLoaderListener（或ContextLoaderServlet）。在web容器启动时，会触发容器初始化事件，此时contextLoaderListener会监听到这个事件，其contextInitialized方法会被调用，在这个方法中，spring会初始化一个启动上下文，这个上下文被称为根上下文，即WebApplicationContext，这是一个接口类，确切的说，其实际的实现类是XmlWebApplicationContext。这个就是spring的IoC容器，其对应的Bean定义的配置由web.xml中的context-param标签指定。在这个IoC容器初始化完毕后，spring容器以WebApplicationContext.ROOTWEBAPPLICATIONCONTEXTATTRIBUTE为属性Key，将其存储到ServletContext中，便于获取；
3. 再次，contextLoaderListener监听器初始化完毕后，开始初始化web.xml中配置的Servlet，这个servlet可以配置多个，以最常见的DispatcherServlet为例（Spring MVC），这个servlet实际上是一个标准的前端控制器，用以转发、匹配、处理每个servlet请求。DispatcherServlet上下文在初始化的时候会建立自己的IoC上下文容器，用以持有spring mvc相关的bean，这个servlet自己持有的上下文默认实现类也是XmlWebApplicationContext。在建立DispatcherServlet自己的IoC上下文时，会利用WebApplicationContext.ROOTWEBAPPLICATIONCONTEXTATTRIBUTE先从ServletContext中获取之前的根上下文(即WebApplicationContext)作为自己上下文的parent上下文（即第2步中初始化的XmlWebApplicationContext作为自己的父容器）。有了这个parent上下文之后，再初始化自己持有的上下文（这个DispatcherServlet初始化自己上下文的工作在其initStrategies方法中可以看到，大概的工作就是初始化处理器映射、视图解析等）。

初始化完毕后，spring以与servlet的名字相关(此处不是简单的以servlet名为Key，而是通过一些转换)的属性为属性Key，也将其存到ServletContext中，以便后续使用。这样每个servlet就持有自己的上下文，即拥有自己独立的bean空间，同时各个servlet共享相同的bean，即根上下文定义的那些bean。

# AOP(Aspect Oriented Programming)
AOP（Aspect Oriented Programming）称为面向切面编程，在程序开发中主要用来解决一些系统层面上的问题，比如日志，事务，权限等待，Struts2的拦截器设计就是基于AOP的思想，是个比较经典的例子。

在不改变原有的逻辑的基础上，增加一些额外的功能。代理也是这个功能，读写分离也能用aop来做。

AOP可以说是OOP（Object Oriented Programming，面向对象编程）的补充和完善。OOP引入封装、继承、多态等概念来建立一种对象层次结构，用于模拟公共行为的一个集合。不过OOP允许开发者定义纵向的关系，但并不适合定义横向的关系，例如日志功能。日志代码往往横向地散布在所有对象层次中，而与它对应的对象的核心功能毫无关系对于其他类型的代码，如安全性、异常处理和透明的持续性也都是如此，这种散布在各处的无关的代码被称为横切（cross cutting），在OOP设计中，它导致了大量代码的重复，而不利于各个模块的重用。

AOP技术恰恰相反，它利用一种称为"横切"的技术，剖解开封装的对象内部，并将那些影响了多个类的公共行为封装到一个可重用模块，并将其命名为"Aspect"，即切面。所谓"切面"，简单说就是那些与业务无关，却为业务模块所共同调用的逻辑或责任封装起来，便于减少系统的重复代码，降低模块之间的耦合度，并有利于未来的可操作性和可维护性。

使用"横切"技术，AOP把软件系统分为两个部分：核心关注点和横切关注点。业务处理的主要流程是核心关注点，与之关系不大的部分是横切关注点。横切关注点的一个特点是，他们经常发生在核心关注点的多处，而各处基本相似，比如权限认证、日志、事物。AOP的作用在于分离系统中的各种关注点，将核心关注点和横切关注点分离开来。

## Advice通知类型介绍：
1. Before:在目标方法被调用之前做增强处理,@Before只需要指定切入点表达式即可

2. AfterReturning:在目标方法正常完成后做增强,@AfterReturning除了指定切入点表达式后，还可以指定一个返回值形参名returning,代表目标方法的返回值

3. AfterThrowing:主要用来处理程序中未处理的异常,@AfterThrowing除了指定切入点表达式后，还可以指定一个throwing的返回值形参名,可以通过该形参名来访问目标方法中所抛出的异常对象

4. After:在目标方法完成之后做增强，无论目标方法时候成功完成。@After可以指定一个切入点表达式

5. Around:环绕通知,在目标方法完成前后做增强处理,环绕通知是最重要的通知类型,像事务,日志等都是环绕通知,注意编程中核心是一个ProceedingJoinPoint



# 注解
## @Configuration
该类等价与XML中配置beans，相当于Ioc容器，它的某个方法头上如果注册了@Bean，就会作为这个Spring容器中的Bean，与xml中配置的bean意思一样。
## @Value
为了简化从properties里取配置，可以使用@Value, 可以properties文件中的配置值。
## @Controller @Service @Repository @Component
1. @controller用来定义控制(dao)层的组件
2. @service用来定义业务层(service)的组件
3. @repository用来定义持久层(domain)的组件
4. @component用来定义不在上述范围内的一般性组件
## @PostConstruct @PreDestory 
实现初始化和销毁bean之前进行的操作，只能有一个方法可以用此注释进行注释，方法不能有参数，返回值必需是void,方法需要是非静态的。   

@PostConstruct：在构造方法和init方法（如果有的话）之间得到调用，且只会执行一次。

@PreDestory：注解的方法在destory()方法调用后得到执行。

引深一点，Spring 容器中的 Bean 是有生命周期的，Spring 允许在 Bean 在初始化完成后以及 Bean 销毁前执行特定的操作，常用的设定方式有以下三种：

1.通过实现 InitializingBean/DisposableBean 接口来定制初始化之后/销毁之前的操作方法；

2.通过bean元素的 init-method/destroy-method属性指定初始化之后 /销毁之前调用的操作方法；

3.在指定方法上加上@PostConstruct 或@PreDestroy注解来制定该方法是在初始化之后还是销毁之前调用

但他们之前并不等价。即使3个方法都用上了，也有先后顺序.

Constructor > @PostConstruct >InitializingBean > init-method

## @Primary
自动装配时当出现多个Bean候选者时，被注解为@Primary的Bean将作为首选者，否则将抛出异常。
## @Lazy(true)
用于指定该Bean是否取消预初始化，用于注解类，延迟初始化。
## @Autowired
Autowired默认先按byType，如果发现找到多个bean，则按照byName方式比对，如果还有多个，则报出异常。
1. 可以手动指定按byName方式注入，使用@Qualifier。
2. 如果要允许null 值，可以设置它的required属性为false，如：@Autowired(required=false) 
## @Resource
默认按byName自动注入，如果找不到再按byType找bean，如果还是找不到则抛异常，无论按byName还是byType如果找到多个，则抛异常。

可以手动指定bean,它有2个属性分别是name和type，使用name属性，则使用byName的自动注入，而使用type属性时则使用byType自动注入：

@Resource(name=”bean名字”) 或 @Resource(type=”bean的class”)

这个注解是属于J2EE的，减少了与spring的耦合。

## @Async
基于@Async标注的方法，称之为异步方法,这个注解用于标注某个方法或某个类里面的所有方法都是需要异步处理的。被注解的方法被调用的时候，会在新线程中执行，而调用它的方法会在原来的线程中执行。
## @Named
@Named和Spring的@Component功能相同。@Named可以有值，如果没有值生成的Bean名称默认和类名相同。比如@Named public class Person 或 @Named(“cc”) public class Person
## @Inject
使用@Inject需要引用javax.inject.jar，它与Spring没有关系，是jsr330规范。

与@Autowired有互换性。
## @Singleton
只要在类上加上这个注解，就可以实现一个单例类，不需要自己手动编写单例实现类。

## @RequestBody
@RequestBody（required=true）:有个默认属性required,默认是true,当body里没内容时抛异常。

## @CrossOrigin
是Cross-Origin ResourceSharing（跨域资源共享）的简写。

作用是解决跨域访问的问题，在Spring4.2以上的版本可直接使用。在类上或方法上添加该注解。

## @RequestParam
作用是提取和解析请求中的参数。@RequestParam支持类型转换，类型转换目前支持所有的基本Java类型
```java
@RequestParam([value=”number”], [required=false])  String number 
```
将请求中参数为number映射到方法的number上。required=false表示该参数不是必需的，请求上可带可不带。

## @PathVariable，@RequestHeader，@CookieValue
1. @PathVariable：处理requet uri部分,当使用@RequestMapping URI template 样式映射时， 即someUrl/{paramId}, 这时的paramId可通过 @Pathvariable注解绑定它传过来的值到方法的参数上。
2. @RequestHeader，@CookieValue: 处理request header部分的注解。

## @Scope
配置bean的作用域。
默认是单例模式，即@Scope(“singleton”),

singleton：单例，即容器里只有一个实例对象。

prototype：多对象，每一次请求都会产生一个新的bean实例，Spring不无法对一个prototype bean的整个生命周期负责，容器在初始化、配置、装饰或者是装配完一个prototype实例后，将它交给客户端，由程序员负责销毁该对象，不管何种作用域，容器都会调用所有对象的初始化生命周期回调方法，而对prototype而言，任何配置好的析构生命周期回调方法都将不会被调用。

request：对每一次HTTP请求都会产生一个新的bean，同时该bean仅在当前HTTP request内有效。

## @ResponseStatus
@ResponseStatus用于修饰一个类或者一个方法，修饰一个类的时候，一般修饰的是一个异常类,当处理器的方法被调用时，@ResponseStatus指定的code和reason会被返回给前端。value属性是http状态码，比如404，500等。reason是错误信息。

当修改类或方法时，只要该类得到调用，那么value和reason都会被添加到response里。

## @RestController
@RestController = @Controller + @ResponseBody。

是2个注解的合并效果，即指定了该controller是组件，又指定方法返回的是String或json类型数据，不会解决成jsp页面，注定不够灵活，如果一个Controller即有SpringMVC返回视图的方法，又有返回json数据的方法即使用@RestController太死板。

灵活的作法是：定义controller的时候，直接使用@Controller，如果需要返回json可以直接在方法中添@ResponseBody

## 元注解，包括@Retention @Target @Document @Inherited四种
元注解是指注解的注解，比如我们看到的ControllerAdvice注解定义如下。
```java
@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Component  
public @interface ControllerAdvice {  
    XXX  
}  
```
1. @Retention: 定义注解的保留策略
2. @Target：定义注解的作用目标
3. @Document：说明该注解将被包含在javadoc中
4. @Inherited：说明子类可以继承父类中的该注解

## RequestMapping
处理映射请求的注解。用于类上，表示类中的所有响应请求的方法都是以该地址作为父路径。有6个属性：
1. value：指定请求的实际地址，指定的地址可以是URI Template 模式；
2. method：指定请求的method类型， GET、POST、PUT、DELETE等；
3. consumes：指定处理请求的提交内容类型（Content-Type），例如@RequestMapping(value = ”/test”, consumes=”application/json”)处理application/json内容类型
4. produces：指定返回的内容类型，仅当request请求头中的(Accept)类型中包含该指定类型才返回；
5. params：指定request中必须包含某些参数值，才让该方法处理；
6. headers：指定request中必须包含某些指定的header值，才能让该方法处理请求。

## @GetMapping @PostMapping
1. @GetMapping(value = “page”)等价于@RequestMapping(value = “page”, method = RequestMethod.GET)
2. @PostMapping(value = “page”)等价于@RequestMapping(value = “page”, method = RequestMethod.POST)

# Bean
## Bean的作用域
作用域限定了Spring Bean的作用范围，在Spring配置文件定义Bean时，通过声明scope配置项，可以灵活定义Bean的作用范围。例如，当你希望每次IOC容器返回的Bean是同一个实例时，可以设置scope为singleton；当你希望每次IOC容器返回的Bean实例是一个新的实例时，可以设置scope为prototype。

scope配置项有5个属性，用于描述不同的作用域。

五种作用域中，request、session和global session三种作用域仅在基于web的应用中使用（不必关心你所采用的是什么web应用框架），只能用在基于web的Spring ApplicationContext环境。

1. singleton

使用该属性定义Bean时，IOC容器仅创建一个Bean实例，IOC容器每次返回的是同一个Bean实例。

singleton是默认的作用域，当定义Bean时，如果没有指定scope配置项，Bean的作用域被默认为singleton。singleton属于单例模式，在整个系统上下文环境中，仅有一个Bean实例。也就是说，在整个系统上下文环境中，你通过Spring IOC获取的都是同一个实例。

2. prototype

使用该属性定义Bean时，IOC容器可以创建多个Bean实例，每次返回的都是一个新的实例。

当一个Bean的作用域被定义prototype时，意味着程序每次从IOC容器获取的Bean都是一个新的实例。因此，对有状态的bean应该使用prototype作用域，而对无状态的bean则应该使用singleton作用域。

3. request

该属性仅对HTTP请求产生作用，使用该属性定义Bean时，每次HTTP请求都会创建一个新的Bean，适用于WebApplicationContext环境。

4. session

该属性仅用于HTTP Session，同一个Session共享一个Bean实例。不同Session使用不同的实例。

5. global-session

该属性仅用于HTTP Session，同session作用域不同的是，所有的Session共享一个Bean实例。

## 生命周期
### 执行过程
1. 根据配置情况调用 Bean 构造方法或工厂方法实例化 Bean。
2. 利用依赖注入完成 Bean 中所有属性值的配置注入。
3. 如果 Bean 实现了 BeanNameAware 接口，则 Spring 调用 Bean 的 setBeanName() 方法传入当前 Bean 的 id 值。
4. 如果 Bean 实现了 BeanFactoryAware 接口，则 Spring 调用 setBeanFactory() 方法传入当前工厂实例的引用。
5. 如果 Bean 实现了 ApplicationContextAware 接口，则 Spring 调用 setApplicationContext() 方法传入当前 ApplicationContext 实例的引用。
6. 如果 BeanFactory 装配了 org.springframework.beans.factory.config.BeanPostProcessor后处理器，将调用 BeanPostProcessor 的 Object postProcessBeforeInitialization(Object bean, String beanName)接口方法对 Bean 进行加工操作。其中入参 bean 是当前正在处理的 Bean，而 beanName 是当前 Bean 的配置名，返回的对象为加工处理后的 Bean。用户可以使用该方法对某些 Bean 进行特殊的处理，甚至改变 Bean 的行为， BeanPostProcessor 在 Spring 框架中占有重要的地位，为容器提供对 Bean 进行后续加工处理的切入点， Spring 容器所提供的各种“神奇功能”（如 AOP，动态代理等）都通过 BeanPostProcessor 实施。
7. 如果 Bean 实现了 InitializingBean 接口，则 Spring 将调用 afterPropertiesSet() 方法。
8. 如果在配置文件中通过 init-method 属性指定了初始化方法，则调用该初始化方法。
9. 如果 BeanPostProcessor 和 Bean 关联，则 Spring 将调用该接口的初始化方法 postProcessAfterInitialization()。此时，Bean 已经可以被应用系统使用了。
10. 如果在 中指定了该 Bean 的作用范围为 scope=“singleton”，则将该 Bean 放入 Spring IoC 的缓存池中，将触发 Spring 对该 Bean 的生命周期管理；如果在 中指定了该 Bean 的作用范围为 scope=“prototype”，则将该 Bean 交给调用者，调用者管理该 Bean 的生命周期，Spring 不再管理该 Bean。
11. 如果 Bean 实现了 DisposableBean 接口，则 Spring 会调用 destory() 方法将 Spring 中的 Bean 销毁；如果在配置文件中通过 destory-method 属性指定了 Bean 的销毁方法，则 Spring 将调用该方法对 Bean 进行销毁。

### Bean方法的分类
* Bean 自身的方法：如调用 Bean 构造函数实例化 Bean，调用 Setter 设置 Bean 的属性值以及通过\<bean\>的 init-method 和 destroy-method 所指定的方法；
* Bean 级生命周期接口方法：如 BeanNameAware、 BeanFactoryAware、 InitializingBean 和 DisposableBean，这些接口方法由 Bean 类直接实现；
* 容器级生命周期接口方法：InstantiationAwareBean PostProcessor 和 BeanPostProcessor 这两个接口实现，一般称它们的实现类为“后处理器”。 后处理器接口一般不由 Bean 本身实现，它们独立于 Bean，实现类以容器附加装置的形式注册到 Spring 容器中并通过接口反射为 Spring 容器预先识别。当Spring 容器创建任何 Bean 的时候，这些后处理器都会发生作用，所以这些后处理器的影响是全局性的。当然，用户可以通过合理地编写后处理器，让其仅对感兴趣Bean 进行加工处理。

ApplicationContext 和 BeanFactory 另一个最大的不同之处在于：ApplicationContext会利用 Java 反射机制自动识别出配置文件中定义的 BeanPostProcessor、 InstantiationAwareBeanPostProcessor 和 BeanFactoryPostProcessor，并自动将它们注册到应用上下文中；而后者需要在代码中通过手工调用 addBeanPostProcessor()方法进行注册。这也是为什么在应用开发时，我们普遍使用 ApplicationContext 而很少使用 BeanFactory 的原因之一。

### 不同对象的加载模式
#### 单例管理的对象   
当scope=”singleton”，即默认情况下，会在启动容器时（即实例化容器时）时实例化。但我们可以指定Bean节点的lazy-init=”true”来延迟初始化bean，这时候，只有在第一次获取bean时才会初始化bean，即第一次请求该bean时才初始化。
#### 非单例管理的对象   
当scope=”prototype”时，容器也会延迟初始化bean，Spring读取xml文件的时候，并不会立刻创建对象，而是在第一次请求该bean时才初始化（如调用getBean方法时）。在第一次请求每一个prototype的bean时，Spring容器都会调用其构造器创建这个对象，然后调用init-method属性值中所指定的方法。对象销毁的时候，Spring容器不会帮我们调用任何方法，因为是非单例，这个类型的对象有很多个，Spring容器一旦把这个对象交给你之后，就不再管理这个对象了。

可以发现，对于作用域为prototype的bean，其destroy方法并没有被调用。如果bean的scope设为prototype时，当容器关闭时，destroy方法不会被调用。对于prototype作用域的bean，有一点非常重要，那就是Spring不能对一个prototype bean的整个生命周期负责：容器在初始化、配置、装饰或者是装配完一个prototype实例后，将它交给客户端，随后就对该prototype实例不闻不问了。不管何种作用域，容器都会调用所有对象的初始化生命周期回调方法。但对prototype而言，任何配置好的析构生命周期回调方法都将不会被调用。清除prototype作用域的对象并释放任何prototype bean所持有的昂贵资源，都是客户端代码的职责（让Spring容器释放被prototype作用域bean占用资源的一种可行方式是，通过使用bean的后置处理器，该处理器持有要被清除的bean的引用）。谈及prototype作用域的bean时，在某些方面你可以将Spring容器的角色看作是Java new操作的替代者，任何迟于该时间点的生命周期事宜都得交由客户端来处理。

Spring容器可以管理singleton作用域下bean的生命周期，在此作用域下，Spring能够精确地知道bean何时被创建，何时初始化完成，以及何时被销毁。而对于prototype作用域的bean，Spring只负责创建，当容器创建了bean的实例后，bean的实例就交给了客户端的代码管理，Spring容器将不再跟踪其生命周期，并且不会管理那些被配置成prototype作用域的bean的生命周期。

## 实例化Bean的方式
1. 使用类构造器实例化(默认的类构造器)
```xml
<bean id=“orderService" class="cn.itcast.OrderServiceBean"/>
```
2. 使用静态工厂方法实例化 
```xml
<bean id="personService" class="cn.itcast.service.OrderFactory" factory-method="createOrder"/>
```
```java
public class OrderFactory {
    // 注意这里的这个方法是 static 的！
    public static OrderServiceBean createOrder(){   
        return new OrderServiceBean();
    }
}
```
3. 使用实例工厂方法实例化: 
```xml
<bean id="personServiceFactory" class="cn.itcast.service.OrderFactory"/>
<bean id="personService" factory-bean="personServiceFactory" factory-method="createOrder"/>
```
```java
public class OrderFactory {
    public OrderServiceBean createOrder(){
        return new OrderServiceBean();
    }
}
```
在BAMS中，工作流引擎activiti的各个组件就是使用此方式实例化的。  
```xml
<bean id="processEngine" class="org.activiti.spring.ProcessEngineFactoryBean">
    <property name="processEngineConfiguration" ref="processEngineConfiguration"/>
</bean>
 
<bean id="repositoryService" factory-bean="processEngine" factory-method="getRepositoryService"/>
<bean id="runtimeService" factory-bean="processEngine" factory-method="getRuntimeService"/>
<bean id="formService" factory-bean="processEngine" factory-method="getFormService"/>
<bean id="identityService" factory-bean="processEngine" factory-method="getIdentityService"/>
<bean id="taskService" factory-bean="processEngine" factory-method="getTaskService"/>
<bean id="historyService" factory-bean="processEngine" factory-method="getHistoryService"/>
<bean id="managementService" factory-bean="processEngine" factory-method="getManagementService"/>
```
使用工厂实例化的Bean跟普通Bean不同，其返回的对象不是指定类的一个实例，其返回的是该FactoryBean的getObject方法所返回的对象。以下为ProcessEngineFactoryBean源码。 
```java
public class ProcessEngineFactoryBean implements FactoryBean<ProcessEngine> ..{
    ...
    protected ProcessEngineImpl processEngine;
     
    public ProcessEngine getObject() throws Exception {
    initializeExpressionManager();
    initializeTransactionExternallyManaged();
 
    if (processEngineConfiguration.getBeans()==null) {
        processEngineConfiguration.setBeans(new SpringBeanFactoryProxyMap(applicationContext));
    }
 
    processEngine = (ProcessEngineImpl) processEngineConfiguration.buildProcessEngine();
 
    return processEngine;
    }
     
    ...
}
```
ProcessEngineFactoryBean最终返回的是processEngine对象，repositoryService、runtimeService、formService等等组件都是通过processEngine里的getXX方法获得的。 

总结：Spring中有三种实例化Bean的方式，类构造器实例化、静态工厂方法实例化及实例工厂方法实例化。两种Bean类型，一种是普通Bean，另一种是工厂Bean，即FactoryBean，这两种Bean都被容器管理，但工厂Bean跟普通Bean不同，其返回的对象不是指定类的一个实例，其返回的是该FactoryBean的getObject方法所返回的对象。在Spring框架内部，有很多地方有FactoryBean的实现类，它们在很多应用如(Spring的AOP、ORM、事务管理)及与其它第三框架(ehCache)集成时都有体现。