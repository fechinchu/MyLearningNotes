# Spring源码

# 1.Spring之核心模块

![image-20210219174016566](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210219174016566.png)

* Spring核心模块
  * spring-core:依赖注入IOC与DI的最基本实现;
  * spring-beans:Bean工厂与Bean的装配;
  * spring-context:定义基础的Spring的Context上下文即容器
  * spring-context-support:对Spring IOC容器的扩展支持,以及IOC子容器;
  * spring-context-indexer:Spring的类管理组件和Classpath扫描;
  * spring-expression:Spring表达式语言;
* Spring切面编程
  * spring-aop:面向切面编程的应用模块,整合Asm,CGLIb,JDKProxy;
  * spring-aspects:集成AspectJ;
  * spring-instrument:动态Class Loading模块;
* Spring数据访问与集成
  * spring-jdbc:Spring提供的JDBC抽象框架的主要实现模块,用于简化Spring JDBC操作;
  * spring-tx:Spring JDBC事务控制实现模块;
  * spring-orm:主要集成Hibernate,Java Persistence API(JPA)和Java Data Objects(JDO);
  * spring-oxm:将java对象映射成XML数据,或者将XML数据映射成java对象;
  * spring-jms:Java Messaging Service,能够发送和接收消息;
* Spring Web
  * spring-web:提供了最基础Web支持,主要建立于核心容器之上,通过Servlet或者Listeners来初始化IOC容器;
  * spring-webmvc:实现了Spring MVC的web应用;
  * spring-websocket:主要是与Web前端的全双工通讯的协议;
  * spring-webflux:一个新的非堵塞函数式Reactive Web框架,可以用来建立异步的,非阻塞,事件驱动的服务;
* Spring通信报文
  * spring-messaging:从Spring4开始新加入的一个模块,主要为Spring框架集成一些基础的报文传送应用;

* Spring集成测试
  * spring-test:主要为测试提供;
* Spring集成兼容
  * spring-framework-bom:Bill of Materials:解决Spring的不同模块依赖版本不同问题;

## 1.1.Spring各模块之间的依赖关系

![image-20210220094411854](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210220094411854.png)

## 1.2.SpringMVC实现的基本思路

![image-20210223172435617](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210223172435617.png)

# 2.Spring IOC

> 1. 对象和对象的关系如何表示?
>
>    xml/properties;
>
> 2. 描述对象关系的文件存放在哪里?
>
>    classpath/network/filesystem/servletContext
>
> 3. 如何统一配置文件的标准?
>
>    BeanDefinition
>
> 4. 如何对不同的配置文件进行解析?
>
>    策略模式

## 2.1.BeanFactory

Spring中的Bean的创建是工厂模式,这一系列的Bean工厂,即IOC容器.ApplicationContext是Spring提供的一个高级IOC容器,它除了能够提供IOC容器的基本功能,还为用户提供了如下附加服务.

1. 支持信息源.可以实现国际化(实现MessageSource接口);
2. 访问资源(实现ResourcePatternResovler接口);
3. 支持应用事件(实现ApplicaitionEventPublisher接口);

BeanFactory和ApplicationContext类之间的继承关系图如下:

![image-20210312095114358](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210312095114358.png)

![image-20210312110449685](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210312110449685.png)

## 2.2.BeanDefinition

Spring IOC容器管理我们定义的各种Bean对象及其相互关系,Bean对象在Spring实现中是以BeanDefinition来描述的,其继承体系如下:

![image-20210312100445238](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210312100445238.png)

## 2.3.BeanDefinitionReader

Bean的解析主要就是对Spring配置文件的解析,这个解析过程要通过BeanDefinitionReader来完成;

![image-20210312100928166](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210312100928166.png)

## 2.4.Spring的定位,加载,注册流程

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210312101602100.png" alt="image-20210312101602100" style="zoom: 33%;" />

![image-20210224154205777](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210224154205777.png)

1. 寻找IOC容器初始化入口;
2. 定位配置文件的`setConfigLocations()`方法;
3. AbstractApplicationContext的`refresh()`方法;
4. AbstractApplicationContext的`obtainFreshBeanFactory()`方法;
5. AbstractRefreshableApplicationContext的`loadBeanDefinitions()`方法;
6. AbstractBeandDefinitionReader读取Bean的配置资源;
7. 资源加载器获取要读入的资源;
8. XmlBeanDefinitionReader加载Bean配置资源;
9. DocumentLoader将Bean配置资源转换为Document对象;
10. XmlBeanDefinitionReader解析载入的Bean配置资源文件;
11. DefaultBeanDefinitionDocumentReader对Bean定义的Document对象解析;
12. BeanDefinitionParserDelegate解析Bean配置资源文件中的`<bean>`元素;
13. 解析`<property>`元素;
14. 解析`<property>`元素的子元素;
15. 解析`<list>`元素;
16. 解析过后的BeanDefinition在IOC容器中的注册;
17. DefaultListableBeanFactory向IOC容器注册解析后的BeanDefinition;

![image-20210312144647610](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210312144647610.png)

# 3.Spring DI

## 3.1.依赖注入的关键类

* IOC
  * BeanFactory;
  * AbstractBeanFactory;
* 初始化获得BeanDefinition
  * SimpleInstantiationStrategy;
* DI
  * AbstractPropertyAccessor;
* 真正实例化
  * BeanWrapper;

## 3.2.依赖注入发生时间

* 用户第一次调用`getBean()`方法时;
* 用户在配置文件中将`<bean>`元素配置了`lazy-init=false`属性,即让容器在解析Bean定义时就触发注入;

## 3.3.依赖注入执行细节

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210312155425059.png" alt="image-20210312155425059" style="zoom: 33%;" />

![image-20210312172515903](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210312172515903.png)

## 3.4.Tips:BeanFactory和FactoryBean

* **FactoryBean**:Spring内部实现一种规范,它是以&开头作为BeanName,Spring中所有的容器都是FactoryBean,因为容器本身也由容器管理,root来创建,都是单例放在IOC容器中;

* **BeanFactory**:Bean工厂的顶层规范,只是定义了方法;

# 4.Spring AOP

## 4.1.概念

1. 切面(Aspect)面向规则,具有相同规则的方法的集合体;
2. 通知(Advice)回调;
3. 切入点(Pointcut)需要代理的具体方法;
4. 目标对象(Target Object)被代理的对象;
5. AOP代理(AOP Proxy)主要两种方式:JDK,CGLib;
6. 前置通知(Before Advice)在Pointcut之前调用,织入的方法;
7. 后置通知(After Advice)在Pointcut之后调用,织入的方法;
8. 返回后置通知(After Return Advice)返回值为非void,织入的方法;
9. 环绕通知(Around Advice)只要触发调用,织入的方法;
10. 异常通知(After Throwing Advice)Pointcut抛出异常,织入的方法;

## 4.2.主要流程

![image-20210313144758945](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210313144758945.png)

![image-20210315134508439](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210315134508439.png)

`AbstractAutowireCapableBeanFactory.initializeBean()`方法:

![image-20210313161340443](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210313161340443.png)

![image-20210313163324976](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210313163324976.png)

# 5.SpringMVC

## 5.1.Spring MVC请求处理流程

 ![image-20210315134925176](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210315134925176.png)

![image-20210315162037563](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210315162037563.png)

## 5.2.Spring MVC九大组件

![image-20210315145313230](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210315145313230.png)

# 6.Spring IOC DI AOP MVC简略说明

![image-20210323164432343](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210323164432343.png)

# 7.Spring JDBC

## 7.1.事务

事务是访问并可能更新数据库中各项数据的一个程序执行单元(Unit);

特点:事务是恢复和并发控制的基本单位,事务应该具有4个属性:

原子性,一致性,隔离性,持久性,这四个属性通常称为ACID特性;

## 7.2.事务传播属性

![image-20210323183520963](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210323183520963.png)

## 7.3.事务隔离级别

![image-20210323184028628](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210323184028628.png)

* 脏读:一个事务对数据进行了增删改,但未提交,另一事务可以读取到未提交的数据,如果第一个事务这时候回滚了,那么第二个事务就读到了脏数据;
* 不可重复读:一个事务中发生了两次读操作,第一个读操作和第二次操作之间,另外一个事务对数据进行了修改,这时候两次读取的数据是不一致的;
* 幻读:第一个事务对一定范围的数据进行批量修改,第二个事务在这个范围内增加一条数据,这时候第一个事务就会丢失对新增数据的修改.

## 7.4.Spring 事务隔离级别

![image-20210323185016792](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210323185016792.png)

# 8.Spring面试题

## 8.1.BeanFactory和ApplicationContext有什么区别?

ApplicationContext是BeanFactory子接口;

1. IOC容器中的Bean监控,生命周期(PostProcessor);

   PostProcessor;

   ApplicationEvent;

   ContextRefreshedEvent;

   ContextStartedEvent;

   ContextStopedEvent;

   ContextClosedEvent;

   RequestHandlerEvent;

2. 支持国际化;

3. 扩展了统一资源文件读取方式URL(ClassPathXmlApplicationContext,FileSystemApplicationContext,XmlWebApplicationContext,AnnotationConfigApplicationContext);

## 8.2.Spring Bean的生命周期

1. InitalizingBean和DisposableBean;
2. Aware接口;
3. 配置Bean的时候init/destory;
4. @PostConstruct和@PreDestory注解方式;

## 8.3.Spring Bean各作用域之间的区别

1. 什么时候用什么时候创建,用完就销毁(prototype);
2. 容器启动创建,容器销毁就销毁(singleton);
3. 用户创建请求时候创建,用户请求完用完就销毁(request);
4. 用户创建请求时候创建,用户请求完用完就销毁(session);
5. global-session (context);

## 8.4.Spring中的Bean是线程安全的吗?

跟我们写的代码有关系,跟Spring无关;

## 8.5.Spring中用到哪些设计模式?

## 8.6.Spring如何处理循环依赖?

用缓存机制来解决循环依赖问题

`instantiateBean();`

`populateBean();`

![image-20210325094352314](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210325094352314.png)

在DefaultSingletonBeanRegistry中,有三级缓存;

* singletonObjects一级缓存,用于保存实例化,注入,初始化完成的bean实例;
* earlySingletonObjects二级缓存,用于保存实例化完成的bean实例;
* singletonFactories三级缓存,用于保存bean创建的工厂,以便后面扩展有机会创建代理对象;

~~~java
@Service
public class TestService1 {

    @Autowired
    private TestService2 testService2;

    public void test1() {
    }
}

@Service
public class TestService2 {

    @Autowired
    private TestService1 testService1;

    public void test2() {
    }
}
~~~

![image-20210325094902546](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210325094902546.png)

