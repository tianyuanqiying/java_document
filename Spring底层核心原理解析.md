# Spring底层核心原理解析

1. Bean的生命周期底层原理
2. 依赖注入底层原理
3. 初始化底层原理
4. 推断构造方法底层原理
5. AOP底层原理
6. Spring事务底层原理



```java
ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");
UserService userService = (UserService) context.getBean("userService");
```

老版的Spring会见到ClassPathXMlApplicationContext容器、该容器以xml方式读取Bean的信息注入IOC容器；

spring.xml：

```xml
<context:component-scan base-package="com.zhouyu"/>
<bean id="userService" class="com.zhouyu.service.UserService"/>
```

----------------------

而新版的Spring和SpringBoot中，ClassPathXMLApplicationContext已经过时、取代使用的是AnnotationConfigApplicationContext容器, 使用方式@Configuration + @Bean方式；

两者底层有很多共同之处；

```java
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
//ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");
        UserService userService = (UserService) context.getBean("userService");
```

```java
@ComponentScan("com.zhouyu")
public class AppConfig {
 @Bean
 public UserService userService(){
  return new UserService();
 }
}
```

目前，我们基本很少直接使用上面这种方式来用Spring，而是使用Spring MVC，或者Spring Boot，但是它们都是基于上面这种方式的，都需要在内部去创建一个ApplicationContext的，只不过：

1. Spring MVC创建的是**XmlWebApplicationContext**，和**ClassPathXmlApplicationContext**类似，都是基于XML配置的
2. Spring Boot创建的是**AnnotationConfigApplicationContext**

---

## Spring如何创建对象？

上述代码中，创建对象实例的步骤要么是new AnnotationConfigApplicationContext时、要么是getBean时创建；

创建AnnotationConfigApplicationContext传入了AppConfig参数，该参数定义了扫描路径com.zhouyu下的类，如果类被@Service、@Component、@Configuration等注解修饰、 会将这些Bean的描述信息记录下来、放入一个Map中；  这些Bean的描述信息 指定的是 BeanDefinition，即Bean定义； 存放这些BeanDefinition的容器为BeanDefinitionMap；



## Bean的创建过程

1. 利用类的构造方法创建对象(如果有多个构造方法，则会进行**构造方法推断**、即使用哪个构造方法创建对象)
2. 如果类存在@Autowired注解修饰的属性、则从容器中找到并赋值 (**依赖注入**)
3. 依赖注入后，Spring会判断该对象是否实现了BeanNameAware接口、BeanClassLoaderAware接口、BeanFactoryAware接口，如果实现了，就表示当前对象必须实现该接口中所定义的setBeanName()、setBeanClassLoader()、setBeanFactory()方法，那Spring就会调用这些方法并传入相应的参数（**Aware回调**）
4. 判断类是否有使用@PostConstruct注解修饰的方法、存在则执行该方法（**初始化前**）

5. 判断类实现了InitializingBean接口，实现了Spring则会调用重写afterPropertiesSet方法（初始化）
6. 最后，Spring会判断当前对象需不需要进行AOP，如果不需要那么Bean就创建完了，如果需要进行AOP，则会进行动态代理并生成一个代理对象做为Bean（**初始化后**）

Bean对象创建出来后：

1. 如果当前Bean是单例Bean，那么会把该Bean对象存入一个Map<String, Object>，Map的key为beanName，value为Bean对象。这样下次getBean时就可以直接从Map中拿到对应的Bean对象了。（实际上，在Spring源码中，这个Map就是**单例池**）
2. 如果当前Bean是原型Bean，那么后续没有其他动作，不会存入一个Map，下次getBean时会再次执行上述创建过程，得到一个新的Bean对象。



## 推断构造方法

如果有多个构造方法，则会进行**构造方法推断**， spring要使用哪个构造方法进行实例化、

1. 如果类中只有一个构造方法、则Spring就会使用该构造方法进行实例化；
2. 如果类中存在多个构造方法、
   - 如果存在默认构造方法、则Spring就会使用该默认构造方法创建实例；
   - 如果没有默认构造方法、则Spring会进行报错、因为Spring也不知道用户想使用哪个方法进行实例化；



- 若使用有参构造器、需要传入参数、则
  - Spring根据参数类型从容器中，找到对应的Bean实例、
  - 如果找到的多个Bean实例、则会根据参数名称进行匹配，拿到唯一匹配的参数值；
  - 如果没有找到、则报错、无法创建该实例；



## AOP大致流程

1. 找到所有切面、获得Pointcut和@After、@Before方法；
2. 初始化后、判断PointCut是否和当前对象匹配，匹配上则表示需要进行AOP



利用cglib进行AOP的大致流程：

 1. 生成代理类XxxProxy,  代理类继承 被代理类 TargetClass

 2. 代理类重写父类方法

 3. 代理类中存在target对象、该属性为被代理类的类型、 该对象经过实例化、依赖注入、初始化等步骤、

 4. 执行代理类的代理方法时、例如XxxProxy#test

     1. 先执行@Before增强方法；

     2. 调用target#test方法

     3. 再执行@After增强方法；‘

        

UserService代理对象.test()--->执行切面逻辑--->target.test()，注意target对象不是代理对象，而是被代理对象



## Spring事务

业务中加上@Transactional注解、就标志该方法是事务方法、本质也是通过AOP实现的；

当执行@Transactional方法时、

1. 代理对象利用事务管理器创建一个新的连接Connection;
2. 修改事务的自动提交状态为false;
3. 执行target#method()，业务逻辑；
4. 业务逻辑执行完、事务提交、出现异常则回滚；



Spring事务是否会失效的判断标准：**某个加了@Transactional注解的方法被调用时，要判断到底是不是直接被代理对象调用的，如果是则事务会生效，如果不是则失效。**

