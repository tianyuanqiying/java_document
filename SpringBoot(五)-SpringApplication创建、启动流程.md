SpringBoot(五)  - SpringApplication创建、以及启动流程

## 原理解析-SpringApplication创建初始化流程

main方法：

```java
public static void main(String[] args) {
    SpringApplication.run(WeiboApplication.class, args);
}

public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
   return run(new Class<?>[] { primarySource }, args);
}

public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
   return new SpringApplication(primarySources).run(args);
}
```

### SpringApplication创建过程

```java
public SpringApplication(Class<?>... primarySources) {
   this(null, primarySources);
}
```

```java
//    1. 调用getSpringFactoriesInstances 加载META_INF/spring.factories文件
//    2. 从spring.factories中获取配置好的ApplicationContextInitializer
//    3. 从spring.factories中获取配置好的ApplicationListener
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
   this.resourceLoader = resourceLoader;
   Assert.notNull(primarySources, "PrimarySources must not be null");
   this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
   this.webApplicationType = WebApplicationType.deduceFromClasspath();
   setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
   setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
   this.mainApplicationClass = deduceMainApplicationClass();
}
```

#### getSpringFactoriesInstances： 从spring.factories中加载配置好的类

```java
  // ------------------getSpringFactoriesInstances----------------------------
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
   return getSpringFactoriesInstances(type, new Class<?>[] {});
}
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
   ClassLoader classLoader = getClassLoader();
   Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
   List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
   AnnotationAwareOrderComparator.sort(instances);
   return instances;
}

  // ----------------------->SpringFactoriesLoader#loadFactoryNames<--------
public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
   String factoryTypeName = factoryType.getName();
   return loadSpringFactories(classLoader).getOrDefault(factoryTypeName, Collections.emptyList());
}

private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
   MultiValueMap<String, String> result = cache.get(classLoader);
   if (result != null) {
      return result;
   }
   try {
      Enumeration<URL> urls = (classLoader != null ?
            classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
            ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
      result = new LinkedMultiValueMap<>();
      while (urls.hasMoreElements()) {
         URL url = urls.nextElement();
         UrlResource resource = new UrlResource(url);
         Properties properties = PropertiesLoaderUtils.loadProperties(resource);
         for (Map.Entry<?, ?> entry : properties.entrySet()) {
            String factoryTypeName = ((String) entry.getKey()).trim();
            for (String factoryImplementationName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
               result.add(factoryTypeName, factoryImplementationName.trim());
            }
         }
      }
      cache.put(classLoader, result);
      return result;
   }
   catch (IOException ex) {
   }
}
//   结论：
//   1. 首次调用getSpringFactoriesInstances会调用SpringFactoriesLoader#loadFactoryNames，会去加载META_INF/spring.factories文件，
//   将所有的自动配置类都加载出来，放入缓存cache，下次直接从缓存中获取，再根据需要的类型factoryType获取对应的类；
//   2. 因此首次调用SpringFactoriesLoader#loadFactoryNames是在创建SpringApplication实例中调用的。
```

### run方法 ： 创建并加载IOC容器

```java
public ConfigurableApplicationContext run(String... args) {
    //1. 记录应用启动时间
   StopWatch stopWatch = new StopWatch();
   stopWatch.start();
   
   ConfigurableApplicationContext context = null;
   Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
   
  	//2. 设置headless
   configureHeadlessProperty();
    
   //3. 从spring.factories中获取类型为SpringApplicationRunListener的类、并实例化；
   SpringApplicationRunListeners listeners = getRunListeners(args);
    
   //4. 触发遍历调用SpringApplicationRunListener#starting方法
   listeners.starting();
   try {
       
      //5. 保存命令行参数
      ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
      
      //6、准备环境、并遍历listener正在准备环境
      ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
       
      //配置需要忽略的Bean信息
      configureIgnoreBeanInfo(environment);
      
      //7. 打印Banner、 即启动图案
      Banner printedBanner = printBanner(environment);
      
      //8.创建IOC容器、类型为AnnotationConfigServletWebApplicationContext 
      context = createApplicationContext();
       
      //9. 获取SpringBootExceptionReporter；
       //从spring.factories中获取 SpringBootExceptionReporter的类并实例化
      exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
            new Class[] { ConfigurableApplicationContext.class }, context);
       
      //10. 准备容器、 通知listner正在准备容器
      prepareContext(context, environment, listeners, applicationArguments, printedBanner);
      
      //11.加载IOC容器、启动嵌入式Tomcat
      refreshContext(context);
       
      //空方法、作为扩展使用、 
      afterRefresh(context, applicationArguments);
      
      //12、记录启动结束时间
      stopWatch.stop();
      if (this.logStartupInfo) {
         new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
      }
      
      //13、通知listener容器已经启动完成；
      listeners.started(context);
       
      //14、 调用CommandLineRunner、ApplicationRunner#run方法；处理SpringBoot应用加载完成的业务 
      callRunners(context, applicationArguments);
   }
   catch (Throwable ex) {
      //16.启动失败、 通知listener应用启动失败 
      handleRunFailure(context, ex, exceptionReporters, listeners);
      throw new IllegalStateException(ex);
   }

   try {
      //15、通知listener、容器正在运行； 
      listeners.running(context);
   }
   catch (Throwable ex) {
      //16.监听器调用失败、 通知异常器exceptionReporters启动失败；
      handleRunFailure(context, ex, exceptionReporters, null);
      throw new IllegalStateException(ex);
   }
   return context;
}
```

#### 1、 记录应用启动时间

```java
public void start() throws IllegalStateException {
    this.start("");
}

public void start(String taskName) throws IllegalStateException {
    if (this.currentTaskName != null) {
        throw new IllegalStateException("Can't start StopWatch: it's already running");
    } else {
        this.currentTaskName = taskName;
        this.startTimeNanos = System.nanoTime();
    }
}
```

#### 2、设置headless

```java
private static final String SYSTEM_PROPERTY_JAVA_AWT_HEADLESS = "java.awt.headless";

private void configureHeadlessProperty() {
   System.setProperty(SYSTEM_PROPERTY_JAVA_AWT_HEADLESS,
         System.getProperty(SYSTEM_PROPERTY_JAVA_AWT_HEADLESS, Boolean.toString(this.headless)));
}
```

#### 3、获取SpringApplicationRunListener

```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
   Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
   return new SpringApplicationRunListeners(logger,
         getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
}
//getSpringFactoriesInstances :从/META-INF下获取SpringApplicationRunlistener的类并实例化
//再包装为SpringApplicationRunListeners实例；
```

#### 4、 通知Listener正在启动

```java
class SpringApplicationRunListeners {
   private final List<SpringApplicationRunListener> listeners;
   SpringApplicationRunListeners(Log log, Collection<? extends SpringApplicationRunListener> listeners) {
      this.log = log;
      this.listeners = new ArrayList<>(listeners);
   }
   void starting() {
      for (SpringApplicationRunListener listener : this.listeners) {
         listener.starting();
      }
   }   
}    
//遍历所有的SpringApplicationRunListener#starting方法，某些监听器对starting感兴趣、就进行处理
```

#### 5、保存命令行参数

```java
public class DefaultApplicationArguments implements ApplicationArguments {
   private final Source source;
   private final String[] args;
   public DefaultApplicationArguments(String... args) {
      Assert.notNull(args, "Args must not be null");
      this.source = new Source(args);
      this.args = args;
   }
}
//将命令行参数包装为DefaultApplicationArguments保存；
```

#### 6、创建并准备环境信息

```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,  ApplicationArguments applicationArguments) {
   //6.1 创建环境信息
   ConfigurableEnvironment environment = getOrCreateEnvironment();
   
   //6.2 配置环境信息
   configureEnvironment(environment, applicationArguments.getSourceArgs());
   
   //6.3 保存环境信息 
   ConfigurationPropertySources.attach(environment);
   
   //6.4 通知listener，环境准备完成 
   listeners.environmentPrepared(environment);
   bindToSpringApplication(environment);
   if (!this.isCustomEnvironment) {
      environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
            deduceEnvironmentClass());
   }
   ConfigurationPropertySources.attach(environment);
   return environment;
}
//6.1 根据webApplicationType创建环境信息、此处类型为StandardServletEnvironment;
private ConfigurableEnvironment getOrCreateEnvironment() {
   if (this.environment != null) {
      return this.environment;
   }
   switch (this.webApplicationType) {
   case SERVLET:
      return new StandardServletEnvironment();
   case REACTIVE:
      return new StandardReactiveWebEnvironment();
   default:
      return new StandardEnvironment();
   }
}
//6.2 配置环境信息
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
   //添加类型转换器、配置值需要进行类型转换
   if (this.addConversionService) {
      ConversionService conversionService = ApplicationConversionService.getSharedInstance();
      environment.setConversionService((ConfigurableConversionService) conversionService);
   }
   //准备配置外部配置源；
   configurePropertySources(environment, args);
   //准备配置文件信息
   configureProfiles(environment, args);
}

//6.4 触发listener的environmentPrepared的事件、遍历调用listener#environmentPrepared方法； 
void environmentPrepared(ConfigurableEnvironment environment) {
   for (SpringApplicationRunListener listener : this.listeners) {
      listener.environmentPrepared(environment);
   }
}
```

#### 7、打印Banner

```java
private Banner printBanner(ConfigurableEnvironment environment) {
   SpringApplicationBannerPrinter bannerPrinter = new SpringApplicationBannerPrinter(resourceLoader, this.banner);

   return bannerPrinter.print(environment, this.mainApplicationClass, System.out);
}

//SpringApplicationBannerPrinter#print 获取打印器、调用打印方法；
Banner print(Environment environment, Class<?> sourceClass, PrintStream out) {
   //默认返回SpringBootRunner;
   Banner banner = getBanner(environment);
    
   //打印SpringBoot的默认图案
   banner.printBanner(environment, sourceClass, out);
   return new PrintedBanner(banner, sourceClass);
}
```



```java
class SpringBootBanner implements Banner {

   private static final String[] BANNER = { "", "  .   ____          _            __ _ _",
         " /\\\\ / ___'_ __ _ _(_)_ __  __ _ \\ \\ \\ \\", "( ( )\\___ | '_ | '_| | '_ \\/ _` | \\ \\ \\ \\",
         " \\\\/  ___)| |_)| | | | | || (_| |  ) ) ) )", "  '  |____| .__|_| |_|_| |_\\__, | / / / /",
         " =========|_|==============|___/=/_/_/_/" };

   private static final String SPRING_BOOT = " :: Spring Boot :: ";

   private static final int STRAP_LINE_SIZE = 42;

   @Override
   public void printBanner(Environment environment, Class<?> sourceClass, PrintStream printStream) {
      //控制台打印SpringBoot的Banner图案
      for (String line : BANNER) {
         printStream.println(line);
      }
      String version = SpringBootVersion.getVersion();
      version = (version != null) ? " (v" + version + ")" : "";
      StringBuilder padding = new StringBuilder();
      //拼接待打印的版本号信息
      while (padding.length() < STRAP_LINE_SIZE - (version.length() + SPRING_BOOT.length())) {
         padding.append(" ");
      }
		
      //打印版本号
      printStream.println(AnsiOutput.toString(AnsiColor.GREEN, SPRING_BOOT, AnsiColor.DEFAULT, padding.toString(),
            AnsiStyle.FAINT, version));
      printStream.println();
   }
}
```

![SpirngBoot默认Banner图案](assets/\image-20221204105055912.png)

#### 8、创建IOC容器

```java
protected ConfigurableApplicationContext createApplicationContext() {
   Class<?> contextClass = this.applicationContextClass;
   if (contextClass == null) {
      try {
         switch (this.webApplicationType) {
         case SERVLET:
            contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
            break;
         case REACTIVE:
            contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
            break;
         default:
            contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
         }
      }
      catch (ClassNotFoundException ex) {
      }
   }
   return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
//根据应用类型获取对应的容器类型、并实例化；此处为AnnotationConfigServletWebApplicationContext;
```

#### 9、获取SpringBootExceptionReporter；

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
   ClassLoader classLoader = getClassLoader();
   Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
   List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
   return instances;
}
//从/META-INF/spring.factories下获取类型为SpringBootExceptionReporter的类、并进行实例化；
```

#### 10、准备IOC容器、触发Listener的prepareContexted方法

```java
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
      SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
   //设置环境信息；
   context.setEnvironment(environment);
   postProcessApplicationContext(context);
   
   //调用初始化器进行初始化
   applyInitializers(context);
    
   //触发listener的容器准备完成的回调方法
   listeners.contextPrepared(context);

   // Add boot specific singleton beans
   ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
   
   //将命令行参数作为Bean注入容器；
   beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
  
   //将Banner作为Bean注入容器
   if (printedBanner != null) {
      beanFactory.registerSingleton("springBootBanner", printedBanner);
   }
    
   if (beanFactory instanceof DefaultListableBeanFactory) {
      ((DefaultListableBeanFactory) beanFactory)
            .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
   }
 	//设置懒加载初始化后置处理器；
   if (this.lazyInitialization) {
      context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
   }
   //加载资源、此处返回为启动类的Class
   Set<Object> sources = getAllSources();
   Assert.notEmpty(sources, "Sources must not be empty");
   
   //加载启动类作为Bean注入容器
   load(context, sources.toArray(new Object[0]));

   //触发监听器的容器加载完成方法；
   listeners.contextLoaded(context);
}

//遍历ApplicationContextInitializer进行初始化；
protected void applyInitializers(ConfigurableApplicationContext context) {
   for (ApplicationContextInitializer initializer : getInitializers()) {
      Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(initializer.getClass(),
            ApplicationContextInitializer.class);
      Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
      initializer.initialize(context);
   }
}

//遍历调用listener#contextPrepared，代表容器准备完成；某些监听器可能对此事件感兴趣并进行业务处理；
void contextPrepared(ConfigurableApplicationContext context) {
   for (SpringApplicationRunListener listener : this.listeners) {
      listener.contextPrepared(context);
   }
}

//遍历调用listener#contextLoaded、代表容器准备完成、
void contextLoaded(ConfigurableApplicationContext context) {
   for (SpringApplicationRunListener listener : this.listeners) {
      listener.contextLoaded(context);
   }
}
```

#### 11、加载IOC容器、启动嵌入式Tomcat

- 层次调用后、最终会调用AbstractApplicationContext#refresh方法、Spring的IOC容器典型的13个方法、加载应用里所有的Bean;  

- ServletWebApplicationContext重写了onRefresh方法、创建并启动嵌入式Tomcat：

```java
private void refreshContext(ConfigurableApplicationContext context) {
   refresh((ApplicationContext) context);
}

@Deprecated
protected void refresh(ApplicationContext applicationContext) {
   refresh((ConfigurableApplicationContext) applicationContext);
}
protected void refresh(ConfigurableApplicationContext applicationContext) {
   applicationContext.refresh();
}

@Override
public final void refresh() throws BeansException, IllegalStateException {
   try {
      super.refresh();
   }
   catch (RuntimeException ex) 
      throw ex;
   }
}

@Override
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
      // Prepare this context for refreshing.
      prepareRefresh();

      // Tell the subclass to refresh the internal bean factory.
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // Prepare the bean factory for use in this context.
      prepareBeanFactory(beanFactory);

      try {
         // Allows post-processing of the bean factory in context subclasses.
         postProcessBeanFactory(beanFactory);

         // Invoke factory processors registered as beans in the context.
         invokeBeanFactoryPostProcessors(beanFactory);

         // Register bean processors that intercept bean creation.
         registerBeanPostProcessors(beanFactory);

         // Initialize message source for this context.
         initMessageSource();

         // Initialize event multicaster for this context.
         initApplicationEventMulticaster();

         // Initialize other special beans in specific context subclasses.
         onRefresh();

         // Check for listener beans and register them.
         registerListeners();

         // Instantiate all remaining (non-lazy-init) singletons.
         finishBeanFactoryInitialization(beanFactory);

         // Last step: publish corresponding event.
         finishRefresh();
      }

      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }

         // Destroy already created singletons to avoid dangling resources.
         destroyBeans();

         // Reset 'active' flag.
         cancelRefresh(ex);

         // Propagate exception to caller.
         throw ex;
      }

      finally {
         // Reset common introspection caches in Spring's core, since we
         // might not ever need metadata for singleton beans anymore...
         resetCommonCaches();
      }
   }
}
```

#### 12、记录启动结束时间

```java
public void stop() throws IllegalStateException {
        long lastTime = System.nanoTime() - this.startTimeNanos;
        this.totalTimeNanos += lastTime;
        this.lastTaskInfo = new StopWatch.TaskInfo(this.currentTaskName, lastTime);
        if (this.keepTaskList) {
            this.taskList.add(this.lastTaskInfo);
        }
        ++this.taskCount;
        this.currentTaskName = null;
}
```

#### 13、通知listener容器已经启动完成；

```java
void started(ConfigurableApplicationContext context) {
   for (SpringApplicationRunListener listener : this.listeners) {
      listener.started(context);
   }
}
//遍历调用SpringApplicationRunListener#started方法、代表容器已经启动完成；
```

#### 14、 调用CommandLineRunner、ApplicationRunner#run方法

```java
private void callRunners(ApplicationContext context, ApplicationArguments args) {
   List<Object> runners = new ArrayList<>();
   runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
   runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
   AnnotationAwareOrderComparator.sort(runners);
   for (Object runner : new LinkedHashSet<>(runners)) {
      if (runner instanceof ApplicationRunner) {
         //调用ApplicationRunner#run方法
         callRunner((ApplicationRunner) runner, args);
      }
      if (runner instanceof CommandLineRunner#run方法) {
         //调用CommandLineRunner#run方法
         callRunner((CommandLineRunner) runner, args);
      }
   }
}
```

#### 15、通知listener、容器正在运行

```java
void running(ConfigurableApplicationContext context) {
   for (SpringApplicationRunListener listener : this.listeners) {
      listener.running(context);
   }
}
//遍历调用SpringApplicationRunListener#running方法、代表应用正在运行；
```

#### 16.启动失败

```java
//发生异常、启动失败；
//1. 处理异常退出码；
	1.1 发布异常退出码时间context.publishEvent(new ExitCodeEvent(context, exitCode))
    1.2 注册异常退出码registerExitCode
//2. 触发listener#failed启动失败回调方法；
//3. 记录失败异常信息；
private void handleRunFailure(ConfigurableApplicationContext context, Throwable exception,
      Collection<SpringBootExceptionReporter> exceptionReporters, SpringApplicationRunListeners listeners) {
   try {
      try {
         handleExitCode(context, exception);
         if (listeners != null) {
            listeners.failed(context, exception);
         }
      }
      finally {
         reportFailure(exceptionReporters, exception);
         if (context != null) {
            context.close();
         }
      }
   }
   catch (Exception ex) {
      logger.warn("Unable to close ApplicationContext", ex);
   }
   ReflectionUtils.rethrowRuntimeException(exception);
}


void failed(ConfigurableApplicationContext context, Throwable exception) {
   for (SpringApplicationRunListener listener : this.listeners) {
      callFailedListener(listener, context, exception);
   }
}
```

