# Spring启动流程分析

Spring的启动，包括ApplicationContext构建和refresh方法的过程

过程如下：

1. ApplicationContext构建

   - 创建BeanFactory;
   - 注册一些核心BeanPostProcessor
     - AutowiredAnnotationBeanPostProcessor : 处理@Autowired，@Value
     - CommonAnnotationBeanPostProcessor ： 处理@Resource, @PostConstruct，@PreDestroy
     - ConfigurationClassPostProcessor ： 处理@Configuration， 扫描并得到BeanDefinition
     - EventListenerMethodProcessor: 扫描ApplicationListener类；
     - DefaultEventListenerFactory:  生产Listener的工厂
     - ApplicationContextAwareProcessor: 处理该类负责的Aware接口；
     - ContextAnnotationAutowireCandidateResolver： isAutoCandidate判断Bean是否可被注入等；

2. 注册配置类为BeanDefinition

3. refresh方法

   - 准备BeanFactory

   - 扫描并解析配置类， 得到所有的BeanDefinition；
     - 通过调用BeanDefinitionRegisteryPostProcessor，BeanFactoryPostProcessor完成
   - 注册后置处理器；
   - 初始化MessageSource， AppContext拥有国际化功能；
   - 初始化ApplicationEventMulticast, AppContext拥有事件发布功能；
   - 注册监听器
   - 创建非懒加载且单例bean的实例化；
   - 初始化LifeCycleProcessor， 调用LifeCycle#start方法；
   - 发布ContextRefreshEvent事件

   



# BeanFactoryPostProcessor

BeanPostProcessor在Bean的生命周期中对Bean的加工处理; BeanFactoryPostProcessor是针对BeanFactory进行操作，即对BeanFactory的加工处理， 例如允许对BeanDefinition的自定义处理；

```java
@FunctionalInterface
public interface BeanFactoryPostProcessor {
    
   void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

}
```

该接口会在Bean初始化进行调用，只能修改BeanFactory里的东西；

```java
@Component
public class CustomBeanPostProcessor implements BeanFactoryPostProcessor
{
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        BeanDefinition messageSource =
                beanFactory.getBeanDefinition("messageSource");
        messageSource.setAutowireCandidate(false);
        messageSource.setBeanClassName("customMessageSource");
        messageSource.setLazyInit(true);
    }
}
```

可以从BeanFactory中获取并修改BeanDefinition的属性；

而ConfigurableListableBeanFactory中，是没有注册Bean定义能力的, 只能进行修改BeanFactory;

 那么AppContext如何注册BeanDefinition？ DefaultListableBeanFactory不仅实现了BeanFactoryPostProcessor, 还实现了BeanDefinitionRegistryPostProcessor, 该接口负责注册BeanDefinition;

## BeanDefinitionRegistryPostProcessor

该接口是BeanFactoryPostProcessor的扩展接口， 创建完BeanDefinition对象后，该接口提供注册BeanDefinition功能；

```java
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
   void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
}
```

```java
@Component
public class CustomBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {
    public Map<String, BeanDefinition>  beanDefinitionMap = new ConcurrentHashMap<>();
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.genericBeanDefinition().getBeanDefinition();
        beanDefinition.setBeanClass(Pet.class);
        beanDefinition.setBeanClassName("pet123");
        beanDefinition.setLazyInit(true);
        registry.registerBeanDefinition(beanDefinition.getBeanClassName(), beanDefinition);
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        BeanDefinition messageSource = beanFactory.getBeanDefinition("xxx")
        messageSource.setLazyInit(true);
    }
}
```

需要实现两个接口， 可利用postProcessBeanDefinitionRegistry方法注册BeanDefinition;

## refresh()

```java
/**
 * Load or refresh the persistent representation of the configuration, which
 * might be from Java-based configuration, an XML file, a properties file, a
 * relational database schema, or some other format.
 * <p>As this is a startup method, it should destroy already created singletons
 * if it fails, to avoid dangling resources. In other words, after invocation
 * of this method, either all or no singletons at all should be instantiated.
 * @throws BeansException if the bean factory could not be initialized
 * @throws IllegalStateException if already initialized and multiple refresh
 * attempts are not supported
 */
void refresh() throws BeansException, IllegalStateException;
```

加载或者刷新持久化的配置， 可以使基于Java的配置， XML文件， Properties文件等；作为一个启动方法， 如果创建单例失败后，单例应该被销毁；

Spring中，有可以重复刷新和只能刷新一次的两种容器、

1. GenericApplicationContext extends AbstractApplicationContext

   - 只能刷新一次，再次调用refresh方法报错；

   - AnnotationConfigApplicationContext 继承GenericApplicationContext ，因此不可刷新

2. AbstractRefreshableApplicationContext extends AbstractApplicationContext

   - 可重复刷新，再次刷新会把上一次加载的bean销毁，再重新加载；

   - AnnotationConfigWebApplicationContext继承AbstractRefreshableApplicationContext类， 拥有刷新功能， web场景中， nacos动态配置就需要动态刷新；





## refresh()底层原理流程

AnnotationConfigApplicationContext容器创建流程分析、

1. 在调用AnnotationConfigApplicationContext构造方法之前，会调用父类GenericApplicationContext的构造方法，创建DefaultListableBeanFactory对象；

2. 创建AnnotationBeanDefinitionReader对象， 注册内置的后置处理器和类的BeanDefinition;

   - 设置dependencyComparator，AnnotationAwareOrderComparator对象，会获取类的@Order或者Order的排序值，再根据排序值进行排序；
   - 设置autowireCandidateResolver， ContextAnnotationAutowireCandidateResolver对象，用来解析某个Bean能不能进行自动注入；
   - 注册ConfiguationClassPostProcessor类的BeanDefinition， 扫描加载并注册BeanDefinition; 
     - 实现了BeanDefinitionRegistryPostProcessor和BeanFactoryPostProcessor接口
   - 注册AutowiredAnnotationBeanPostProcessor类的BeanDefinition, 处理@Autowired， @Value;
   - 注册CommonAnnotationBeanPostProcessor类的BeanDefinition,处理@Resource, @PostConstruct, @PreDesotry;
   - 注册EventListenerMethodProcessor类的BeanDefinition, 扫描@EventListener注解的类；
     - 实现了BeanFactoryPostProcessor，SmartInitializingSingleton
   - 注册DefaultEventListenerFactory类的BeanDefinition, 生成ApplicationListetener的工厂；

3. 创建ClassPathBeanDefinitionScanner对象，内部设置了扫描的类型，

   - 设置**this.includeFilters = AnnotationTypeFilter(Component.class)**

   -  Spring#refresh方法中会重新创建一个Scanner扫描Bean; 此时创建的Scanner, 通过AppContext#scan方法扫描Bean时，使用此Scanner进行扫描

4. 利用reader注册配置类的BeanDefinition

5. 以下refresh方法

6. prepareRefresh 准备刷新

   - 记录开始时间
   -  initPropertySource()：空实现； 允许子容器加载配置到环境Environment中； 
     - 模板设计模式： 自己空方法， 给子类扩展功能
   - 校验环境， 例如校验环境变量中是否有某个变量的值；

7. obtainFreshBeanFactory： 获取Bean工厂；

   - 判断是否可重复刷新AppContext,即是否可多次调用refresh方法；
   - 获取beanFactory,  此处获取DefaultListableBeanFactory

8. prepareBeanFactory: 准备bean工厂

   - 设置类加载器；

   - 设置Spring的EL表达式解析器；

   - 添加ApplicationContextAwareProcessor后置处理器，负责处理部分Aware的回调

   - 设置忽略的依赖项

     - EnvironmentAware,EmbeddedValueResolverAware, MessageSourceAware
     - ResourceLoaderAware, ApplicationEventMulticasterAware, ApplicationContextAware

     - Spring自动注入，只要存在set方法，就会执行自动注入， 这几个Aware可通过回调完成依赖注入；因此进行忽略，减少启动时间；

   - 指定相关类型对应的依赖对象；

     - ApplicationEventMulticaster, ApplicationContext
     - ResourceLoader, Environment
     - **用一个Map存储，key为类型，value为依赖对象，当处理@Autowired注解依赖时， 先从Map中找，找到则直接获取value依赖对象**

   - **添加ApplicationListenerDetector， 用来处理ApplicationListener接口的Bean;**

   - 添加一些单例bean到单例池

     - "environment"：Environment对象
     - "systemProperties"：System.getProperties()返回的Map对象
     - "systemEnvironment"：System.getenv()返回的Map对象

9. postProcessBeanFactory ： 模板设计模式，扩展接口，空实现，子类继承当前类后可重写扩展功能；

   - 扩展处理BeanFactory, 即Spring扫描注册BeanDefinition可以通过此方法添加，修改BeanFactory内容

10. invokeBeanFactoryPostProcessor ：扫描加载并注册BeanDefinition;

    - 调用BeanDefinitionRegistryPostProcessor, BeanFactoryPostProcessor处理器
    - 核心ConfigurationClassPostProcessor: 解析加载并注册BeanDefinition；

11. registerBeanPostProcessors : 注册后置处理器；

    - 此处的注册实际是分批次，按顺序添加到DefaultListableBeanFactory属性beanPostProcessor集合中；
      - 拿到容器中所有的BeanPostProcessor, 根据 @PriorityOrder ,  @Order, 实现MergerBeanDefinitionBeanPostProcessor,  非Order进行分组；
      - @PriorityOrder ：根据order值排序， 加入beanPostProcessor集合；
      - @Order： 根据Order排序， 加入beanPostProcessor集合；
      - 非Order：  加入beanPostProcessor集合；
      -  实现MergerBeanDefinitionBeanPostProcessor： 排序，加入beanPostProcessor集合, 若存在相同BeanPostProcessor, 会先删除；
      - 再次加入ApplicationListenerDetector， 保证放在最后

12. initMessageSource()：初始化MessageSource， 拥有国际化的能力

    - 若存在名为"messageSource"的BeanDefinition, 则利用该Bean定义进行实例化；
    - 不存在则实例化默认的DelegatingMessageSource事件广播器，并注入单例池

13. initApplicationEventMulticaster():初始化事件广播器， 拥有事件发布能力；

    - 若存在“ApplicationListenerMulticaster”的BeanDefinition, 则利用该Bean定义实例化事件广播器；
    - 不存在则实例化默认的DelegatingMessageSource事件广播器，并注入单例池

14. onRefresh(): 模板扩展方法，空实现；子类继承后，可扩展功能；

15. registerListeners():会从BeanFactory中找出类型为ApplicationListener的Bean名称，添加到ApplicationEventMulticasetor中， 此时未实例化，只记录名称；

16. finishBeanFactoryInitialization(): 完成非懒加载单例bean的实例化；

17. finishRefresh():  发布事件；

    - 初始化lifecycleProcessor并设置beanFactory, 再调用SmartLiftCycle#start回调方法；
      - Spring生命周期的扩展机制
    - 发布ContextRefreshEvent事件，触发contextRefreshed回调；



## invokeBeanFactoryPostProcessor 

执行BeanFactoryPostProcessor后置处理器；

1. 先执行手动添加到AppContext的BeanDefinitionRegistryPostProcessor处理器；
2. 拿到@PriorityOrder标记的BeanDefinitionRegistryPostProcessor，排序，再执行；
3. 拿到@Order标记的BeanDefinitionRegistryPostProcessor，排序， 再执行；
4. 拿到非Order的BeanDefinitionRegistryPostProcessor，排序，再执行；
   - 执行回调过程中，可能再次注册了BeanDefinitionRegistryPostProcessor的Bean定义，因此需要循环执行，直到全部执行完；
5. 执行所有自定义的BeanDefinitionRegistryPostProcessor的postProcessBeanFactory回调方法；
6. 执行手动添加到AppContext的BeanDefinitionRegistryPostProcessor回调方法porstProcessBeanFactory
7. 拿到@PriorityOrder标记的BeanFactoryPostProcessor，排序，再执行；
8. 拿到@Order标记的BeanFactoryPostProcessor，排序，再执行；
9. 拿到非Order的BeanFactoryPostProcessor，执行；







[Spring启动流程详解| ProcessOn免费在线作图,在线流程图,在线思维导图](https://www.processon.com/view/link/5f60a7d71e08531edf26a919)[Spring启动流程详解| ProcessOn免费在线作图,在线流程图,在线思维导图](https://www.processon.com/view/link/5f60a7d71e08531edf26a919)

