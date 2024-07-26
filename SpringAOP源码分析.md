# SpringAOP源码分析

- 通过@EnableAspctJAutoProxy开启注解AOP功能；

```java
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {
```

- 通过@Import注解导入AspectJAutoProxyRegistrar类， 在IOC启动时， 扫描到@Import注解后， 若是ImportBeanDefinitionRegistry类型，扫描完后， ImportBeanDefinitionRegistry#registerBeanDefinition方法；
- 会往往BeanFactory加入AnnotationAwareAspectJAutoProxyCreator的Bean定义；

```java
   @Override
   public void registerBeanDefinitions(
         AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
      AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);
      AnnotationAttributes enableAspectJAutoProxy =
            AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
      if (enableAspectJAutoProxy != null) {
         if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
            AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
         }
          //是否将代理对象暴露出来，当前线程，可拿到该实例；
         if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
            AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
         }
      }
   }
}

@Nullable
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(
      BeanDefinitionRegistry registry, @Nullable Object source) {

   return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
}
```

### AnnotationAwareAspectJAutoProxyCreator

AnnotationAwareAspectJAutoProxyCreator是一个是后置处理, 实现了SmartInstantiationAwareBeanPostProcessor；

![AnnotationAwareAspectJAutoProxyCreator继承关系](assets/\image-20230226165532178.png)

### AnnotationAwareAspectJAutoProxyCreator创建

-  由于AnnotationAwareAspectJAutoProxyCreator也是Bean， 会进行Bean生命周期，AbstractApplicationContext#refresh里， registerBeanPostProcessor过程里， 会进行所有BeanPostProcessor实例化，放入容器；

- AnnotationAwareAspectJAutoProxyCreator实现BeanFactoryAware接口， 需要实现setBeanFactory接口；

  - AbstractAdvisorAutoProxyCreator会创建检索Advisor接口检索器advisorRetrievalHelper， **用来扫描实现Advisor接口的类，并创建Advisor对象；**
  - AnnotationAwareAspectJAutoProxyCreator会创建Advisor工厂, 创建用来处理AspectJ注解的处理器；**用来扫描@Aspect, @Before等AOP注解的类并创建对应的Advisor**

  ```java
  //AnnotationAwareAspectJAutoProxyCreator父类AbstractAdvisorAutoProxyCreator重写了setBeanFactory方法
  @Override
  public void setBeanFactory(BeanFactory beanFactory) {
     super.setBeanFactory(beanFactory);
     initBeanFactory((ConfigurableListableBeanFactory) beanFactory);
  }
  protected void initBeanFactory(ConfigurableListableBeanFactory beanFactory) {
     this.advisorRetrievalHelper = new BeanFactoryAdvisorRetrievalHelperAdapter(beanFactory);
  }
  //AnnotationAwareAspectJAutoProxyCreator重写了initBeanFactory方法；
  @Override
  protected void initBeanFactory(ConfigurableListableBeanFactory beanFactory) {
     super.initBeanFactory(beanFactory);
     if (this.aspectJAdvisorFactory == null) {
        this.aspectJAdvisorFactory = new ReflectiveAspectJAdvisorFactory(beanFactory);
     }
     this.aspectJAdvisorsBuilder =
           new BeanFactoryAspectJAdvisorsBuilderAdapter(beanFactory, this.aspectJAdvisorFactory);
  }
  ```

#### AbstractAdvisorAutoProxyCreator

该类通过创建Advisor检索器， 拥有了扫描并创建实现Advisor接口对象的能力；

```java
public abstract class AbstractAdvisorAutoProxyCreator extends AbstractAutoProxyCreator {
   @Nullable
   private BeanFactoryAdvisorRetrievalHelper advisorRetrievalHelper;
   @Override
   public void setBeanFactory(BeanFactory beanFactory) {
      super.setBeanFactory(beanFactory);
      initBeanFactory((ConfigurableListableBeanFactory) beanFactory);
   }
   protected void initBeanFactory(ConfigurableListableBeanFactory beanFactory) {
      this.advisorRetrievalHelper = new BeanFactoryAdvisorRetrievalHelperAdapter(beanFactory);
   }
    //搜索所有实现了Advisor接口的通知器对象；
	protected List<Advisor> findCandidateAdvisors() {
		return this.advisorRetrievalHelper.findAdvisorBeans();
	}
    
}
```

**DefaultAdvisorAspectJAutoProxyCreator与AnnotationAwareAspectJAutoProxyCreator区别**

相同点：两者都继承了AbstractAdvisorAutoProxyCreator类， 都具有扫描并创建实现Advisor接口对象的能力

不同点：AnnotationAwareAspectJAutoProxyCreator拥有AspectJAdvisorBuilder属性， 拥有扫描并处理AOP注解的能力，因此AnnotationAwareAspectJAutoProxyCreator能力更加强大；



### Advisor的扫描与创建

- Spring流程， AOP注解扫描在Bean的实例化前，而不再Bean生命周期之中；

- AnnotationAwareAspectJAutoProxyCreator实现了InstantiationBeanPostProcessor接口， 实现了postprocessBeforeInstantiation方法；

  ```java
  @Override
  public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
     Object cacheKey = getCacheKey(beanClass, beanName);
  
     if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
        if (this.advisedBeans.containsKey(cacheKey)) {
           return null;
        }
        if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
           this.advisedBeans.put(cacheKey, Boolean.FALSE);
           return null;
        }
     }
     return null;
  }
  ```

  - AbstractAutoProxyCreator#postProcessBeforeInstantiation

    - AspectJAwareAutoProxyCreator#shouldSkip

      - AnnotationAwareAspectJAutoProxyCreator#findCandidateAdvisors

        1. 调用AspectJAwareAutoProxyCreator#findCandidateAdvisors通过Advisor检索器执行扫描并创建实现了Advisor接口的对象
        2. 调用AnnotationAwareAspectJAutoProxyCreator#buildAspectJAdvisors通过AspectJAdvisor创建器执行扫描AOP注解并创建Advisor

        ```javascript
        //AnnotationAwareAspectJAutoProxyCreator#findCandidateAdvisors
        @Override
        protected List<Advisor> findCandidateAdvisors() {
           // 通过Advisor检索器执行扫描并创建实现了Advisor接口的对象
           List<Advisor> advisors = super.findCandidateAdvisors();
           // AspectJAdvisor创建器执行扫描AOP注解并创建Advisor
           if (this.aspectJAdvisorsBuilder != null) {
              advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
           }
           return advisors;
        }
        ```



####  基于接口方式的Advisor创建

AbstractAdvisorAutoProxyCreator#findCandidateAdvisor方法里，调用了Advisor检索器，扫描并创建Advisor;

```java
//AbstractAdvisorAutoProxyCreator#findCandidateAdvisor
protected List<Advisor> findCandidateAdvisors() {
   return this.advisorRetrievalHelper.findAdvisorBeans();
}
```

1. 扫描所有的Advisor;从BeanFactory里获取类型为Advisor的Bean定义；
2. 创建Advisor的Bean对象

```java
public List<Advisor> findAdvisorBeans() {
   //看是否存在缓存；
   String[] advisorNames = this.cachedAdvisorBeanNames;
   
    //扫描所有的Advisor;从BeanFactory里获取类型为Advisor的Bean定义；
   if (advisorNames == null) {
      advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
            this.beanFactory, Advisor.class, true, false);
      this.cachedAdvisorBeanNames = advisorNames;
   }
    //创建Advisor对象
   List<Advisor> advisors = new ArrayList<>();
   for (String name : advisorNames) {
      if (isEligibleBean(name)) {
         if (this.beanFactory.isCurrentlyInCreation(name)) {}
         else {
            try {advisors.add(this.beanFactory.getBean(name, Advisor.class));}
            catch (BeanCreationException ex) {}
         }
      }
   }
   return advisors;
}
```

#### 基于注解Advisor的扫描与创建

1. 从容器种获取类型为Object的Bean名称， 此处获取到所有的bean名称；
2. 遍历bean名称；
3. 判断bean名称对应的bean是否被@Aspect修饰；
4. 若Bean名称被@Aspect注解修饰， 通过Advisor工厂创建Advisor;

```java
public List<Advisor> buildAspectJAdvisors() {
   List<String> aspectNames = this.aspectBeanNames;
   if (aspectNames == null) {
      synchronized (this) {
         aspectNames = this.aspectBeanNames;
         if (aspectNames == null) {
            List<Advisor> advisors = new ArrayList<>();
            aspectNames = new ArrayList<>();
            String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                  this.beanFactory, Object.class, true, false);
            for (String beanName : beanNames) {
               Class<?> beanType = this.beanFactory.getType(beanName, false);
               if (this.advisorFactory.isAspect(beanType)) {
                  aspectNames.add(beanName);
                  AspectMetadata amd = new AspectMetadata(beanType, beanName);
                  if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
                     MetadataAwareAspectInstanceFactory factory =
                           new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
                     List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
					//省略缓存
                     advisors.addAll(classAdvisors);
                  }
               }
            }
            this.aspectBeanNames = aspectNames;
            return advisors;
         }
      }
   }
   //省略缓存..
   return advisors;
}
```

1. 通过getAdvisorMethods获取所有非@Pointcut修饰的方法, 遍历方法生成Advisor;
2. 若方法被Before.class, Around.class, After.class, AfterReturning.class, AfterThrowing.class, Pointcut.clas修饰，则生成PointCut对象；否则返回空；
3. 再创建Advisor对象， 内部包含了增强方法Method, Pointcut,  切面类， 创建Advise对象；



```java
@Override
public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
   Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
   String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
   validate(aspectClass);

   MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
         new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);

   List<Advisor> advisors = new LinkedList<Advisor>();
   for (Method method : getAdvisorMethods(aspectClass)) {
      Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
      if (advisor != null) {
         advisors.add(advisor);
      }
   }

	//省略...
   return advisors;
}

//获取Advisor
@Override
public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
      int declarationOrderInAspect, String aspectName) {
   AspectJExpressionPointcut expressionPointcut = getPointcut(
         candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
   if (expressionPointcut == null) {
      return null;
   }

   return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
         this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
}
    
//创建Advisor对象
public InstantiationModelAwarePointcutAdvisorImpl(AspectJExpressionPointcut declaredPointcut,
      Method aspectJAdviceMethod, AspectJAdvisorFactory aspectJAdvisorFactory,
      MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {

   this.declaredPointcut = declaredPointcut;
   this.declaringClass = aspectJAdviceMethod.getDeclaringClass();
   this.methodName = aspectJAdviceMethod.getName();
   this.parameterTypes = aspectJAdviceMethod.getParameterTypes();
   this.aspectJAdviceMethod = aspectJAdviceMethod;
   this.aspectJAdvisorFactory = aspectJAdvisorFactory;
   this.aspectInstanceFactory = aspectInstanceFactory;
   this.declarationOrder = declarationOrder;
   this.aspectName = aspectName;

   if (aspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {}
   else {
      this.pointcut = this.declaredPointcut;
      this.lazy = false;
      this.instantiatedAdvice = instantiateAdvice(this.declaredPointcut);
   }
}
```

#### 创建Advise

会根据注解的类型，创建不同的Advise

@Before: AspectJMethodBeforeAdvice类型

@After: AspectJAfterAdvice类型

@AfterReturning: AspectJAfterReturningAdvice类型

@AfterThrowing:AspectJAfterThrowingAdvice类型

@Around: AspectJAroundAdvice类型

```java
private Advice instantiateAdvice(AspectJExpressionPointcut pcut) {
   return this.aspectJAdvisorFactory.getAdvice(this.aspectJAdviceMethod, pcut,
         this.aspectInstanceFactory, this.declarationOrder, this.aspectName);
}
```

```java
@Override
public Advice getAdvice(Method candidateAdviceMethod, AspectJExpressionPointcut expressionPointcut,
      MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {

   Class<?> candidateAspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
   validate(candidateAspectClass);

   AspectJAnnotation<?> aspectJAnnotation =
         AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
   if (aspectJAnnotation == null) {
      return null;
   }

   AbstractAspectJAdvice springAdvice;

   switch (aspectJAnnotation.getAnnotationType()) {
      case AtBefore:
         springAdvice = new AspectJMethodBeforeAdvice(
               candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
         break;
      case AtAfter:
         springAdvice = new AspectJAfterAdvice(
               candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
         break;
      case AtAfterReturning:
         springAdvice = new AspectJAfterReturningAdvice(
               candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
         AfterReturning afterReturningAnnotation = (AfterReturning) aspectJAnnotation.getAnnotation();
         if (StringUtils.hasText(afterReturningAnnotation.returning())) {
            springAdvice.setReturningName(afterReturningAnnotation.returning());
         }
         break;
      case AtAfterThrowing:
         springAdvice = new AspectJAfterThrowingAdvice(
               candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
         AfterThrowing afterThrowingAnnotation = (AfterThrowing) aspectJAnnotation.getAnnotation();
         if (StringUtils.hasText(afterThrowingAnnotation.throwing())) {
            springAdvice.setThrowingName(afterThrowingAnnotation.throwing());
         }
         break;
      case AtAround:
         springAdvice = new AspectJAroundAdvice(
               candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
         break;
      case AtPointcut:
         if (logger.isDebugEnabled()) {
            logger.debug("Processing pointcut '" + candidateAdviceMethod.getName() + "'");
         }
         return null;
      default:
         throw new UnsupportedOperationException(
               "Unsupported advice type on method: " + candidateAdviceMethod);
   }

   // Now to configure the advice...
   springAdvice.setAspectName(aspectName);
   springAdvice.setDeclarationOrder(declarationOrder);
   String[] argNames = this.parameterNameDiscoverer.getParameterNames(candidateAdviceMethod);
   if (argNames != null) {
      springAdvice.setArgumentNamesFromStringArray(argNames);
   }
   springAdvice.calculateArgumentBindings();
   return springAdvice;
}
```



### AOP代理对象创建

Spring中， 为与Bean的生命周期解耦， 因此将AOP放在了初始化后；特殊的循环依赖放在了实例化后，依赖注入前， 对应的BeanPostProcessor的**postProcessAfterInitialization**；不管是DefaultAdvisorAutoProxyCreator, 还是AnnotationAwareAspectJAutoProxyCreator， 都继承了AbstractAutoProxyCreator, 该类实现了postProcessBeforeInstantiation初始化前， postProcessAfterInitialization初始化后方法；

总体流程：

1. 判断当前Bean是否需要进行AOP， 通过初筛，精筛后， 且未生成代理对象的bean可执行AOP；
2. 创建代理工厂ProxyFactory,  设置Advisor通知器，生产代理对象， 若是基于接口，则使用JDK动态代理， 否则使用CGLIB代理



```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
   if (beanName != null && this.targetSourcedBeans.contains(beanName)) {
      return bean;
   }
   if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
      return bean;
   }
   if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
      this.advisedBeans.put(cacheKey, Boolean.FALSE);
      return bean;
   }

   // Create proxy if we have advice.
   Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
   if (specificInterceptors != DO_NOT_PROXY) {
      this.advisedBeans.put(cacheKey, Boolean.TRUE);
      Object proxy = createProxy(
            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
      this.proxyTypes.put(cacheKey, proxy.getClass());
      return proxy;
   }

   this.advisedBeans.put(cacheKey, Boolean.FALSE);
   return bean;
}
//利用ProxyFactory生产代理对象
protected Object createProxy(
      Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {
	//省略部分代码
   ProxyFactory proxyFactory = new ProxyFactory();
   proxyFactory.copyFrom(this);

   Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
   proxyFactory.addAdvisors(advisors);
   proxyFactory.setTargetSource(targetSource);
	//省略部分代码
   return proxyFactory.getProxy(getProxyClassLoader());
}
//利用AOpProxy工厂生产创建AopProxy对象， 存在两种AopProxy对象
//第一种：JdkDynamicAopProxy， 对应JDK代理， 可获取并创建JDK动态代理对象
//第二种：ObjenesesiCglibAopProxy， 对应CGLIB代理，可获取并创建CGLIB代理对象
public Object getProxy(ClassLoader classLoader) {
   return createAopProxy().getProxy(classLoader);
}

protected final synchronized AopProxy createAopProxy() {
   return getAopProxyFactory().createAopProxy(this);
}
```

**AopProxy: 对不同种动态代理技术的抽象而成的接口、每种类型的AopProxy代表一种动态代理技术**

若代理的是接口， 则会使用JDK代 理， 否则使用CGLIB；

```java
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {
   @Override
   public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
      if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
         Class<?> targetClass = config.getTargetClass();
         if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
            return new JdkDynamicAopProxy(config);
         }
         return new ObjenesisCglibAopProxy(config);
      }
      else {
         return new JdkDynamicAopProxy(config);
      }
   }
   private boolean hasNoUserSuppliedProxyInterfaces(AdvisedSupport config) {
      Class<?>[] ifcs = config.getProxiedInterfaces();
      return (ifcs.length == 0 || (ifcs.length == 1 && SpringProxy.class.isAssignableFrom(ifcs[0])));
   }

}
```

JDK代理： 创建JDK动态代理的第三个参数为Invocation, 传this代表本身是Invocation类型， 因此执行动态代理时，会调用本身的invoke方法；

```java
//JdkDynamicAopProxy#getProxy
@Override
public Object getProxy(ClassLoader classLoader) {
   if (logger.isDebugEnabled()) {
      logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
   }
   Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
   findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
   return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```

CGLIB代理：创建Enhancer增强器，设置要继承的父类，实现的接口， 拦截处理Callback, 最终创建CGlib代理对象；

```java
@Override
public Object getProxy(ClassLoader classLoader) {
   try {
      Class<?> rootClass = this.advised.getTargetClass();
      Class<?> proxySuperClass = rootClass;
      if (ClassUtils.isCglibProxyClass(rootClass)) {
         proxySuperClass = rootClass.getSuperclass();
         Class<?>[] additionalInterfaces = rootClass.getInterfaces();
         for (Class<?> additionalInterface : additionalInterfaces) {
            this.advised.addInterface(additionalInterface);
         }
      }

      // Configure CGLIB Enhancer...
      Enhancer enhancer = createEnhancer();

      enhancer.setSuperclass(proxySuperClass);
      enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));

      Callback[] callbacks = getCallbacks(rootClass);
      Class<?>[] types = new Class<?>[callbacks.length];
      for (int x = 0; x < types.length; x++) {
         types[x] = callbacks[x].getClass();
      }
      // fixedInterceptorMap only populated at this point, after getCallbacks call above
      enhancer.setCallbackFilter(new ProxyCallbackFilter(
            this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
      enhancer.setCallbackTypes(types);

      // Generate the proxy class and create a proxy instance.
      return createProxyClassAndInstance(enhancer, callbacks);
   }
   catch (Throwable ex) {
   }
}

protected Object createProxyClassAndInstance(Enhancer enhancer, Callback[] callbacks) {
   enhancer.setInterceptDuringConstruction(false);
   enhancer.setCallbacks(callbacks);
   return (this.constructorArgs != null ?
         enhancer.create(this.constructorArgTypes, this.constructorArgs) :
         enhancer.create());
}
```

### AOP代理对象执行逻辑

1. 调用TargetSource#getTarget方法，获取到目标对象；
2. 在使用Proxyfactory创建代理对象之前， 需要往ProxyFactory添加Advisor;
3. 代理对象在执行某个方法时， 会把ProxyFactory中的Advisor拿出来和当前正在执行的方法进行筛选匹配；
4. 把和方法所匹配的Advisor适配成MethodInterceptor；
5. 把和当前方法匹配的MethodInterceptor链， 以及被代理对象，代理对象，代理类，当前method对象，方法参数封装为MethodInvocation对象；
6. 调用MethodInvocation的proceed方法， 开始执行各个MethodInterceptor以及被代理对象的方法；
7. 顺序调用每个MethodInterceptor的invoke方法，并且把MethodInvocation对象传入invoke方法；
8. 调用完MethodInterceptor的invoke方法后，就会调用invokeJoinpoint()方法， 从而执行被代理对象的当前方法；

### 各注解对应的MethodInterceptor

- @Before: AspectJMethodBeforeAdvice,  进行动态代理时，会把AspectJMethodBeforeAdvice转成MethodBeforeAdviceInterceptor;
  - 先执行@Before的增强逻辑
  - 再执行invocation#proceed(), 会执行下一个Interceptor, 如果没有，则执行target对应的目标方法；
- @After：AspectJAfterAdvice，实现Interceptor;
  - 先执行MethodInvocation#proceed()方法，如果没有下一个Interceptor,则执行target的目标方法；
  - 执行@After对应的增强方法；
- @Around:  AspectJAroundAdvice, 直接实现了MethodInterceptor；
  - 直接执行Advice方法，由@Around决定要不要往后面调用；
- @AfterThrowing: AspectJThrowingAdvice, 实现了Interceptor
  - 先执行MethodInvocation#proceed方法， 如果没有下一个Interceptor, 则执行target的目标方法；
  - 如果发生异常，则会执行AfterThrowing对应的异常增强处理方法；
- @AfterReturning: AspectJReturningAdvice，在进行动态代理时会把AspectJAfterReturningAdvice转成**AfterReturningAdviceInterceptor**
  - 先执行MethodInvocation#proceed方法， 如果没有一个Interceptor, 则执行target的目标方法；
  - 执行AfterReturning对应的方法；

![AOP流程](assets/\image-20230301233010286.png)