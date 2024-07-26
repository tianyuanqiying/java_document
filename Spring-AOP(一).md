# Spring  AOP

## ProxyFactory

SpirngAOP是基于动态代理实现， 使用JDK，Cglib两种方式；ProxyFactory进行封装，表示创建代理对象的工厂，使用起来更方便；

```java
UserService target = new UserService();
ProxyFactory proxyFactory = new ProxyFactory();
proxyFactory.setTarget(target);
proxyFactory.addAdvice(new MethodInterceptor() {
@Override
public Object invoke(MethodInvocation invocation) throws Throwable {
System.out.println("before...");
Object result = invocation.proceed();
System.out.println("after...");
return result;
}
});
UserInterface userService = (UserInterface) proxyFactory.getProxy();
userService.test();
```

通过ProxyFactory，就无须关注CGLIB还是JDK代理了， ProxyFactory做了这个任务， 如果代理类实现了接口， 则使用JDK动态代理， 否则使用CGLIB;

## Advice的分类

1. Before Advice：方法之前执行 
2. After returning advice：方法return后执行 
3.  After throwing advice：方法抛异常后执行 
4. After (finally) advice：方法执行完finally之后执行，这是最后的，比return更后 
5. Around advice：这是功能最强大的Advice，可以自定义执行顺序

## Advisor

Advisor包含一个Pointcout和一个Advice；Pointcout判断类是否需要被代理；

```java
UserService target = new UserService();
ProxyFactory proxyFactory = new ProxyFactory();
proxyFactory.setTarget(target);
proxyFactory.addAdvisor(new PointcutAdvisor() {
@Override
public Pointcut getPointcut() {
return new StaticMethodMatcherPointcut() {
@Override
public boolean matches(Method method, Class<?> targetClass) {
return method.getName().equals("testAbc");
}
};
}
@Override
public Advice getAdvice() {
return new MethodInterceptor() {
@Override
public Object invoke(MethodInvocation invocation) throws Throwable {
System.out.println("before...");
Object result = invocation.proceed();
System.out.println("after...");
return result;
}
};
}
@Override
public boolean isPerInstance() {
return false;
}
});
UserInterface userService = (UserInterface) proxyFactory.getProxy();
userService.test();

```



## 创建代理对象的方式

使用Spring时， 不会直接使用ProxyFactory,   Spring也会完成创建代理对象操作， 但是需要告诉Spring, 哪些类需要被代理，增强方法是什么；

### ProxyFactoryBean

**指定被代理对象， 指定增强逻辑；利用FactoryBean的特性，间接将UserService代理对象作为Bean;**

如果业务中，需要针对某个类作增强，则可以使用此方法；(不实用)

```java
    @Bean
    public ProxyFactoryBean userServiceProxy() {
        UserService userService = new UserService();
        ProxyFactoryBean proxyFactoryBean = new ProxyFactoryBean();
        proxyFactoryBean.setTarget(userService);
        proxyFactoryBean.setInterceptorNames("zhouyuAroundAdvise", "zhouyuAroundAdvise");
        return proxyFactoryBean;
    }
```

### BeanNameAutoProxyCreator

可指定多个Bean的名字，对bean进行代理；是一个后置处理器

```java
@Bean
public BeanNameAutoProxyCreator autoProxyCreator() {
    BeanNameAutoProxyCreator beanNameAutoProxyCreator = new BeanNameAutoProxyCreator();
    beanNameAutoProxyCreator.setBeanNames("user*","device*");
    beanNameAutoProxyCreator.setInterceptorNames("methodBeforeAdvice");
    return  beanNameAutoProxyCreator;
}
```

可批量进行AOP， 指定多个Advice,  容器中需要存在Advice；

 缺点：根据beanName进行代理，bean的所有方法都会进行动态代理，粒度大；



### NameMatchMethodPointcutAdvisor

可指定被代理方法， 指定增强逻辑， 若bean中在MappedName的方法名，会对bean进行代理；

```java
@EnableAspectJAutoProx注解， 则需要加@Bean
public NameMatchMethodPointcutAdvisor nameMatchMethodPointcutAdvisor() {
    NameMatchMethodPointcutAdvisor nameMatchMethodPointcutAdvisor = new NameMatchMethodPointcutAdvisor();
    nameMatchMethodPointcutAdvisor.setAdvice(methodBeforeAdvice());;
    nameMatchMethodPointcutAdvisor.setMappedNames("test", "insertOne");
    return nameMatchMethodPointcutAdvisor;
}

// 需要开启自动执行AOP功能，以下2选1；
//若是开启了AOP， @EnableAspectJAutoProxy, Spring会自动执行AOP流程；
//若是没有开启@EnableAspectJAutoProx注解， 则需要往容器中加入DefaultAdvisorAutoProxyCreator对象， 是后置处理器， 这样Spring会自动执行AOP代理；
@Bean
public DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator() {
    DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator = new DefaultAdvisorAutoProxyCreator();
    return defaultAdvisorAutoProxyCreator;
}
```

这里还需要定义Advise, Advisor, DefaultAdvisorAutoProxyCreator,  可利用注解方式， 将定义Bean的过程省略掉；

Aspect与Advisor关系： 一个Aspect可包含多个Advisor,  一个Advisor也可以是一个Aspect (NameMatchMethodPointcutAdvisor); 

### 注解版AOP

- 注解版AOP中，需开启@EnableAspectJAutoProxy

- 通过Aspect管理PointCut和Advisor；
- 通过@Pointcut决定被增强的方法； 通过@Before, @After等方法定义增强逻辑； 最终会将注解方法转换为Advisor，放入ProxyFactory；

```java
@Aspect
@Component
public class AopAspect {
    @Pointcut("execution(* com.redis.zset.weibo.aop.proxy.UserService.*(..))")
    public void pointCut(){};

    @Before(value = "pointCut()")
    public void methodBefore(JoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().getName();
        System.out.println("执行目标方法【"+methodName+"】的<前置通知>,入参"+ Arrays.asList(joinPoint.getArgs()));
    }
}
```

### SpringAOP与AspectJ的关系

@Pointcut, @Before, @After等注解来源与AspectJ， Spring AOP利用AspectJ这些核心注解，想用@Before、@Around等注解，是需要单独引入aspecj相关jar包的；

利用了AspectJ解析pointcut的能力， pointcut匹配包括初筛，精筛两个过程， AspecJ存在此功能， Spring也将这个功能利用了；



### AOP概念

Aspect: 切面； @Aspect标记的类，包含一个pointCut, 多个Advisor; 

Advice: 通知；  前置，后者，环绕增强方法；

PointCut: 切点；  用来判断哪些方法需要被进行代理的方法；

Advisor: 通知器； 包含一个PointCut, 一个Advice；  

target: 目标对象； 被代理的对象；

proxy: 代理对象； 经过动态代理后，生成的对象；

joinpoint: 连接点；  需要被增强的方法；

Weaving: 织入； 创建代理对象的动作， AspectJ是编译器创建，SpringAOP是运行时创建；



### Advice在Spring AOP中对应API

1. @Before
2. @AfterReturning
3. @AfterThrowing
4. @After
5. @Around



我们前面也提到过，Spring自己也提供了类似的执行实际的实现类：

1. 接口MethodBeforeAdvice，继承了接口BeforeAdvice
2. 接口AfterReturningAdvice
3. 接口ThrowsAdvice
4. 接口AfterAdvice
5. 接口MethodInterceptor



Spring会把五个注解解析为对应的Advice类：

1. @Before：AspectJMethodBeforeAdvice，实际上就是一个MethodBeforeAdvice
2. @AfterReturning：AspectJAfterReturningAdvice，实际上就是一个AfterReturningAdvice
3. @AfterThrowing：AspectJAfterThrowingAdvice，实际上就是一个MethodInterceptor
4. @After：AspectJAfterAdvice，实际上就是一个MethodInterceptor
5. @Around：AspectJAroundAdvice，实际上就是一个MethodInterceptor



### TargetSource

- AOP框架提供的接口； 用来获取AOP代理对象所代理的真正对象， 默认是静态的， 则会返回同一个对象；
- SingletonTargetSource： 创建单例的动态代理对象时， 会将target目标对象包装为SingletonTargetSource对象，包装模式；

- 在依赖注入阶段， 若是属性被@Lazy注解标记， 赋值对象为TargetSource的代理对象， 当调用@Lazy标记的属性时， 会调用getTarget方法，从容器中获取依赖对象；

```java
protected Object buildLazyResolutionProxy(final DependencyDescriptor descriptor, final @Nullable String beanName) {
   BeanFactory beanFactory = getBeanFactory();
   final DefaultListableBeanFactory dlbf = (DefaultListableBeanFactory) beanFactory;
   TargetSource ts = new TargetSource() {
      @Override
      public Object getTarget() {
         Set<String> autowiredBeanNames = (beanName != null ? new LinkedHashSet<>(1) : null);
         Object target = dlbf.doResolveDependency(descriptor, beanName, autowiredBeanNames, null);
         if (autowiredBeanNames != null) {
            for (String autowiredBeanName : autowiredBeanNames) {
               if (dlbf.containsBean(autowiredBeanName)) {
                  dlbf.registerDependentBean(autowiredBeanName, beanName);
               }
            }
         }
         return target;
      }
   };

   ProxyFactory pf = new ProxyFactory();
   pf.setTargetSource(ts);
   Class<?> dependencyType = descriptor.getDependencyType();
   if (dependencyType.isInterface()) {
      pf.addInterface(dependencyType);
   }
   return pf.getProxy(dlbf.getBeanClassLoader());
}
```



