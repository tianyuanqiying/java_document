                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                

# Spring依赖注入(上)

## Spring依赖注入方式

首先分两种：手动注入、自动注入；

**手动注入**

XML中定义Bean时，就是手动注入，因为是**程序员手动给某个属性指定了值**。

```xml
<bean name="userService" class="com.luban.service.UserService">
 <property name="orderService" ref="orderService"/>
</bean>
```

```xml
<bean name="userService" class="com.luban.service.UserService">
 <constructor-arg index="0" ref="orderService"/>
</bean>
```

前者set注入， 后者构造器注入；

手动注入的底层也就是分为两种：set方法注入；构造方法注入；

**自动注入**

XML中bean标签的autowired属性、 @Autowired注解属性；

### XML的autowire自动注入

在XML中，我们可以在定义一个Bean时去指定这个Bean的自动注入模式：

1. byType
2. byName
3. constructor
4. default
5. no

```xml
<bean id="userService" class="com.luban.service.UserService" autowire="byType"/>
```

这样意味着Spring会给所有属性进行赋值，只要属性存在setter方法；

在创建Bean过程中，Spring会解析当前类的所有方法，每个方法对应一个PropertyDescriptor对象；

PropertyDescriptor类如下

```java
public class PropertyDescriptor extends FeatureDescriptor {
	//属性类型引用：如果是getter方法，则为返回值类型，如果是setter方法，则对应唯一参数类型
    private Reference<? extends Class<?>> propertyTypeRef;
    //getter方法的引用
    private final MethodRef readMethodRef = new MethodRef();
    
    //setter方法的引用
    private final MethodRef writeMethodRef = new MethodRef();
    //
    private Reference<? extends Class<?>> propertyEditorClassRef;

    private boolean bound;
    private boolean constrained;
	//并不是方法的名字，而是方法名处理的名字
    //如果是getXXX,则name=XXX;
    //如果是setXXX,则name=XXX;
    //如果是isXXX,则name=XXX;
    private String baseName;
	//setter方法名称
    private String writeMethodName;
    //getter方法名称
    private String readMethodName;
}
```

#### Spring byName注入的流程

1. 找到所有setter方法对应的XXX部分名称；
2. 根据XXX部分的名字获取bean对象；

```java
protected void autowireByName(
      String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
    //拿到所有setter方法对应的XXX名称；
   String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
   for (String propertyName : propertyNames) {
      if (containsBean(propertyName)) {
          //根据XXX获取bean对象；
         Object bean = getBean(propertyName);
          //加入属性键值对pvs; 表示 propertyName属性对应的值为bean
         pvs.add(propertyName, bean);
      }
   }
}
protected String[] unsatisfiedNonSimpleProperties(AbstractBeanDefinition mbd, BeanWrapper bw) {
   Set<String> result = new TreeSet<>();
   PropertyValues pvs = mbd.getPropertyValues();
   //获取PropertyDescriptor属性描述器
   PropertyDescriptor[] pds = bw.getPropertyDescriptors();
   for (PropertyDescriptor pd : pds) {
      //如果有setter方法， 属性键值对pvs中不存在该值，且是对象类型、则是依赖注入方法；
      if (pd.getWriteMethod() != null && !isExcludedFromDependencyCheck(pd) && !pvs.contains(pd.getName()) &&
            !BeanUtils.isSimpleProperty(pd.getPropertyType())) {
         result.add(pd.getName());
      }
   }
   return StringUtils.toStringArray(result);
}
```

#### Spring byType自动注入流程

1. 拿到setter方法解析后对应的XXX属性名称集合
2. 拿到XXX对应的setter方法对应的PropertyDescriptor描述器；
3. 获取setter方法描述器的方法参数
4. 根据setter方法获取唯一参数的类型，根据该类型从容器中获取bean;如果存在多个，则报错；

```java
protected void autowireByType(
      String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
   TypeConverter converter = getCustomTypeConverter();
   Set<String> autowiredBeanNames = new LinkedHashSet<>(4);
   //1. 拿到setter方法解析后对应的XXX属性名称集合
   String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
   for (String propertyName : propertyNames) {
       	//2. 获取属性描述器；
         PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
		//细节：不支持类型为Object的byType自动注入；
         if (Object.class != pd.getPropertyType()) {
            //3. 获取setter方法对应的方法参数；
            MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
            boolean eager = !(bw.getWrappedInstance() instanceof PriorityOrdered);
            //包装方法参数
            DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
            //4. 解析依赖，容器根据类型获取Bean对象， 多个则报错；
            Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
            
            //加入属性键值对， 代表propertyName属性对应的值为autowiredArgument
            if (autowiredArgument != null) {
               pvs.add(propertyName, autowiredArgument);
            }
            autowiredBeanNames.clear();
         }
      }
   }
}
```



constructor：不需要set方法，当某个bean通过构造方法注入时， spring利用构造方法中的参数信息从Spring容器中获取Bean对象，找到之后作为参数传给构造方法，从而实例化Bean对象，完成属性赋值；

构造方法注入相当于byType + byName， 普通的byType找到多个Bean会报错，而构造器注入通过byType找到多个Bean后，会根据参数名称进行确定；



no：表示关闭autowire

default：表示默认值，我们一直演示的某个bean的autowire，而也可以直接在<beans>标签中设置autowire，如果设置了，那么<bean>标签中设置的autowire如果为default，那么则会用<beans>标签中设置的autowire。



**为什么平时都是用@Autowired，而不是用XML的自动注入方法？**

@Autowired相当于xml的自动注入，但是xml的自动注入粒度比较大， 以整个类为单元，对整个类的属性进行自动注入，只要有setter方法，就会执行自动注入逻辑。

有些时候，我们不需要自动注入的属性，因为存在setter, 也执行了自动注入流程，因为粒度比较大；

而@Autowired只针对某个方法，属性，构造方法，粒度更小；@Autowired无法区分byName, byType,**@Autowired先根据byType， 如果找到多个，则再进行byName;**



## @Autowired自动注入

@Autowired注解可以写在：

1. 属性上：先根据**属性类型**去找Bean，如果找到多个再根据**属性名**确定一个
2. 构造方法上：先根据方法**参数类型**去找Bean，如果找到多个再根据**参数名**确定一个
3. set方法上：先根据方法**参数类型**去找Bean，如果找到多个再根据**参数名**确定一个

底层对应的是：

1. 属性注入
2. set方法注入
3. 构造方法注入

### 寻找注入点

在容器启动过程中，Spring会往容器中提前注入AutowiredAnnotationBeanPostProcessor的BeanDefinition, 以及提前创建后置处理器对象；

AutowiredAnnotationBeanPostProcessor实现了**MergedBeanDefinitionPostProcessor**接口，重写了postProcessMergeBeanDefinition方法，在该方法中，扫描所有的注入点，并放入缓存；

实现了**InstantiationAwareBeanPostProcessor**接口，重写了postProcessProperties， 在该方法中，执行依赖注入；

 #### AutowiredAnnotationBeanPostProcessor创建

```java
public AutowiredAnnotationBeanPostProcessor() {
   this.autowiredAnnotationTypes.add(Autowired.class);
   this.autowiredAnnotationTypes.add(Value.class);
   try {
      this.autowiredAnnotationTypes.add((Class<? extends Annotation>)
            ClassUtils.forName("javax.inject.Inject", AutowiredAnnotationBeanPostProcessor.class.getClassLoader()));
   }
   catch (ClassNotFoundException ex) {   }
}
```

创建时、指定了@Autowired, @Value, @Inject注解， 表示只要被这些注解修饰，就会被扫描到；

#### 扫描注入点

1. 遍历Bean的所有Field属性；
2. 获取字段Field上的@Autowired, @Value, @Inject注解，只要存在一个注解，就是注入点；
3. 若Field属性是static静态的，则不进行注入；
4. 获取@Autowired注解的required属性的值, 其他两个注解无此属性；
5. 包装Field属性和required为AutowiredFieldElement对象，加入集合currElements
6. 遍历Bean的所有方法
7. 若方法Method为桥接方法，则获取原方法Method
8. 获取方法Method上的@Autowired, @Value, @Inject注解， 存在一个，则为注入点；
9. 若方法Method是静态的，则不进行注入，跳过该方法，
10. 获取@Autowired注解的required属性值；
11. 包装方法Method和required属性为AutowiredMethodElement实例，加入集合currElements
12. 遍历完当前类后，会遍历父类的属性和方法，直到没有父类。
13. 最终将currElements包装为InjectionMatadata对象，作为当前Bean的注入点对象并缓存；

```java
@Override
public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
   InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
   metadata.checkConfigMembers(beanDefinition);
}

private InjectionMetadata findAutowiringMetadata(String beanName, Class<?> clazz, @Nullable PropertyValues pvs) {
   String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());
   InjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);
   if (InjectionMetadata.needsRefresh(metadata, clazz)) {
      synchronized (this.injectionMetadataCache) {
         metadata = this.injectionMetadataCache.get(cacheKey);
         if (InjectionMetadata.needsRefresh(metadata, clazz)) {
            if (metadata != null) {
               metadata.clear(pvs);
            }
            metadata = buildAutowiringMetadata(clazz);
            this.injectionMetadataCache.put(cacheKey, metadata);
         }
      }
   }
   return metadata;
}
```



```java
private InjectionMetadata buildAutowiringMetadata(final Class<?> clazz) {
   if (!AnnotationUtils.isCandidateClass(clazz, this.autowiredAnnotationTypes)) {
      return InjectionMetadata.EMPTY;
   }

   List<InjectionMetadata.InjectedElement> elements = new ArrayList<>();
   Class<?> targetClass = clazz;

   do {
      final List<InjectionMetadata.InjectedElement> currElements = new ArrayList<>();
		//处理Field
      ReflectionUtils.doWithLocalFields(targetClass, field -> {
         MergedAnnotation<?> ann = findAutowiredAnnotation(field);
         if (ann != null) {
            if (Modifier.isStatic(field.getModifiers())) {
               if (logger.isInfoEnabled()) {
                  logger.info("Autowired annotation is not supported on static fields: " + field);
               }
               return;
            }
            boolean required = determineRequiredStatus(ann);
            currElements.add(new AutowiredFieldElement(field, required));
         }
      });
	  //处理Method
      ReflectionUtils.doWithLocalMethods(targetClass, method -> {
         Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
         if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
            return;
         }
         MergedAnnotation<?> ann = findAutowiredAnnotation(bridgedMethod);
         if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
            if (Modifier.isStatic(method.getModifiers())) {
               if (logger.isInfoEnabled()) {
                  logger.info("Autowired annotation is not supported on static methods: " + method);
               }
               return;
            }
            if (method.getParameterCount() == 0) {
               if (logger.isInfoEnabled()) {
               }
            }
            boolean required = determineRequiredStatus(ann);
            PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
            currElements.add(new AutowiredMethodElement(method, required, pd));
         }
      });

      elements.addAll(0, currElements);
      targetClass = targetClass.getSuperclass();
   }
   while (targetClass != null && targetClass != Object.class);

   return InjectionMetadata.forElements(elements, clazz);
}
```



### static的字段或方法为什么不支持(细节)

```java
@Component
@Scope("prototype")
public class OrderService {
}
```

```java
@Component
@Scope("prototype")
public class UserService  {
 @Autowired
 private static OrderService orderService;
}
```

OrderService、UserService都是原型Bean, 若支持静态属性的注入；

1. UserService userService1 = context.getBean("userService")
2. UserService userService2 = context.getBean("userService")

userService1的orderService注入完后，orderService属性对象为A，  对于UserService类，orderService静态属性不应该发生变化，永远是A； 

而获取userService2中，会再次注入orderService属性值为B， 导致静态属性发生变化；userService1的orderService属性值为B；

### 桥接方法(非重点)

```java
public interface UserInterface<T> {
 void setOrderService(T t);
}
```

```java
@Component
public class UserService implements UserInterface<OrderService> {

 private OrderService orderService;

 @Override
 @Autowired
 public void setOrderService(OrderService orderService) {
  this.orderService = orderService;
 }

 public void test() {
  System.out.println("test123");
 }

}
```

UserService对应的字节码为：

```
// class version 52.0 (52)
// access flags 0x21
// signature Ljava/lang/Object;Lcom/zhouyu/service/UserInterface<Lcom/zhouyu/service/OrderService;>;
// declaration: com/zhouyu/service/UserService implements com.zhouyu.service.UserInterface<com.zhouyu.service.OrderService>
public class com/zhouyu/service/UserService implements com/zhouyu/service/UserInterface {

  // compiled from: UserService.java

  @Lorg/springframework/stereotype/Component;()

  // access flags 0x2
  private Lcom/zhouyu/service/OrderService; orderService

  // access flags 0x1
  public <init>()V
   L0
    LINENUMBER 12 L0
    ALOAD 0
    INVOKESPECIAL java/lang/Object.<init> ()V
    RETURN
   L1
    LOCALVARIABLE this Lcom/zhouyu/service/UserService; L0 L1 0
    MAXSTACK = 1
    MAXLOCALS = 1

  // access flags 0x1
  public setOrderService(Lcom/zhouyu/service/OrderService;)V
  @Lorg/springframework/beans/factory/annotation/Autowired;()
   L0
    LINENUMBER 19 L0
    ALOAD 0
    ALOAD 1
    PUTFIELD com/zhouyu/service/UserService.orderService : Lcom/zhouyu/service/OrderService;
   L1
    LINENUMBER 20 L1
    RETURN
   L2
    LOCALVARIABLE this Lcom/zhouyu/service/UserService; L0 L2 0
    LOCALVARIABLE orderService Lcom/zhouyu/service/OrderService; L0 L2 1
    MAXSTACK = 2
    MAXLOCALS = 2

  // access flags 0x1
  public test()V
   L0
    LINENUMBER 23 L0
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    LDC "test123"
    INVOKEVIRTUAL java/io/PrintStream.println (Ljava/lang/String;)V
   L1
    LINENUMBER 24 L1
    RETURN
   L2
    LOCALVARIABLE this Lcom/zhouyu/service/UserService; L0 L2 0
    MAXSTACK = 2
    MAXLOCALS = 1

  // access flags 0x1041
  public synthetic bridge setOrderService(Ljava/lang/Object;)V
  @Lorg/springframework/beans/factory/annotation/Autowired;()
   L0
    LINENUMBER 11 L0
    ALOAD 0
    ALOAD 1
    CHECKCAST com/zhouyu/service/OrderService
    INVOKEVIRTUAL com/zhouyu/service/UserService.setOrderService (Lcom/zhouyu/service/OrderService;)V
    RETURN
   L1
    LOCALVARIABLE this Lcom/zhouyu/service/UserService; L0 L1 0
    MAXSTACK = 2
    MAXLOCALS = 2
}
```

可以看到在UserSerivce的字节码中有两个setOrderService方法：

1. public setOrderService(Lcom/zhouyu/service/OrderService;)V
2. public synthetic bridge setOrderService(Ljava/lang/Object;)V

并且都是存在@Autowired注解的。

所以在Spring中需要处理这种情况，当遍历到桥接方法时，得找到原方法。



## 注入点进行注入

AutowiredAnnotation实现了InstantiationAwareBeanPostProcessor接口，重写postProcessProperties方法，实现了自动注入；

### 字段注入

1. 遍历所有的AutowiredFieldElement对象；
2. AutowiredFieldElement中获取Field对象和required， 包装为DependentDescriptor对象；
3. 调用beanFactory#resolveDependency方法，传入DependentDescriptor对象，找到对应的bean对象；
4. 将DependentDescriptor对象包装和结果对象beanName包装为shortcutDependency对象，作为缓存；如果是原型bean, 下次创建该Bean，直接拿缓存结果对象的BeanName, 去beanFactory获取bean对象，不用再进行查找；
5. 利用反射给字段Field属性赋值；

```java
/**
 * Class representing injection information about an annotated field.
 */
private class AutowiredFieldElement extends InjectionMetadata.InjectedElement {
   private final boolean required;
   private volatile boolean cached;
   @Nullable
   private volatile Object cachedFieldValue;

   public AutowiredFieldElement(Field field, boolean required) {
      super(field, null);
      this.required = required;
   }

   @Override
   protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
      Field field = (Field) this.member;
      Object value;
      if (this.cached) {
         value = resolvedCachedArgument(beanName, this.cachedFieldValue);
      }
      else {
          //包装Field, required
         DependencyDescriptor desc = new DependencyDescriptor(field, this.required);
         desc.setContainingClass(bean.getClass());
         Set<String> autowiredBeanNames = new LinkedHashSet<>(1);
         Assert.state(beanFactory != null, "No BeanFactory available");
         TypeConverter typeConverter = beanFactory.getTypeConverter();
         try {
            //找到依赖的bean队形；
            value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
         }
         catch (BeansException ex) {
            throw new UnsatisfiedDependencyException(null, beanName, new InjectionPoint(field), ex);
         }
         
         synchronized (this) {
            if (!this.cached) {
               Object cachedFieldValue = null;
               if (value != null || this.required) {
                  cachedFieldValue = desc;
                  registerDependentBeans(beanName, autowiredBeanNames);
                  if (autowiredBeanNames.size() == 1) {
                     String autowiredBeanName = autowiredBeanNames.iterator().next();
                     //缓存结果
                     if (beanFactory.containsBean(autowiredBeanName) &&
                           beanFactory.isTypeMatch(autowiredBeanName, field.getType())) {
                        cachedFieldValue = new ShortcutDependencyDescriptor(
                              desc, autowiredBeanName, field.getType());
                     }
                  }
               }
               this.cachedFieldValue = cachedFieldValue;
               this.cached = true;
            }
         }
      }
      //反射设置结果
      if (value != null) {
         ReflectionUtils.makeAccessible(field);
         field.set(bean, value);
      }
   }
}
```

### Set方法注入

1. 遍历所有的AutowiredMethodElement对象；
2. 遍历放的参数， 每个参数包装为MethodParamerter对象；
3. MethodParameter包装为DependencyDescriptor对象；
4. 调用beanFactory#resolveDependency，传入DependencyDescriptor对象，查找依赖，找到当前方法参数所匹配的Bean对象；
5. 将**DependencyDescriptor对象**和所找到的**结果对象beanName**封装成一个**ShortcutDependencyDescriptor对象**作为缓存，比如如果当前Bean是原型Bean，那么下次再来创建该Bean时，就可以直接拿缓存的结果对象beanName去BeanFactory中去那bean对象了，不用再次进行查找了
6. 用反射将找到的所有结果对象传给当前方法，并执行。

```java
private class AutowiredMethodElement extends InjectionMetadata.InjectedElement {
   private final boolean required;
   private volatile boolean cached;
   @Nullable
   private volatile Object[] cachedMethodArguments;

   @Override
   protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {

      Method method = (Method) this.member;
      Object[] arguments;
      if (this.cached) {
         // Shortcut for avoiding synchronization...
         arguments = resolveCachedArguments(beanName);
      }
      else {
         int argumentCount = method.getParameterCount();
         arguments = new Object[argumentCount];
         DependencyDescriptor[] descriptors = new DependencyDescriptor[argumentCount];
         Set<String> autowiredBeans = new LinkedHashSet<>(argumentCount);
         TypeConverter typeConverter = beanFactory.getTypeConverter();
         for (int i = 0; i < arguments.length; i++) {
            //获取方法的第I个参数
            MethodParameter methodParam = new MethodParameter(method, i);
            //包装为DependencyDescriptor对象
            DependencyDescriptor currDesc = new DependencyDescriptor(methodParam, this.required);
            currDesc.setContainingClass(bean.getClass());
            descriptors[i] = currDesc;
            try {
                //获取参数对应的依赖bean
               Object arg = beanFactory.resolveDependency(currDesc, beanName, autowiredBeans, typeConverter);
               if (arg == null && !this.required) {
                  arguments = null;
                  break;
               }
               arguments[i] = arg;
            }
            catch (BeansException ex) {}
         }
          //缓存结果
         synchronized (this) {
            if (!this.cached) {
               if (arguments != null) {
                  DependencyDescriptor[] cachedMethodArguments = Arrays.copyOf(descriptors, arguments.length);
                  registerDependentBeans(beanName, autowiredBeans);
                  if (autowiredBeans.size() == argumentCount) {
                     Iterator<String> it = autowiredBeans.iterator();
                     Class<?>[] paramTypes = method.getParameterTypes();
                     for (int i = 0; i < paramTypes.length; i++) {
                        String autowiredBeanName = it.next();
                        if (beanFactory.containsBean(autowiredBeanName) &&
                              beanFactory.isTypeMatch(autowiredBeanName, paramTypes[i])) {
                           cachedMethodArguments[i] = new ShortcutDependencyDescriptor(
                                 descriptors[i], autowiredBeanName, paramTypes[i]);
                        }
                     }
                  }
                  this.cachedMethodArguments = cachedMethodArguments;
               }
               else {
                  this.cachedMethodArguments = null;
               }
               this.cached = true;
            }
         }
      }
      //反射设置结果
      if (arguments != null) {
            ReflectionUtils.makeAccessible(method);
            method.invoke(bean, arguments);
      }
   }

   @Nullable
   private Object[] resolveCachedArguments(@Nullable String beanName) {
      Object[] cachedMethodArguments = this.cachedMethodArguments;
      if (cachedMethodArguments == null) {
         return null;
      }
      Object[] arguments = new Object[cachedMethodArguments.length];
      for (int i = 0; i < arguments.length; i++) {
         arguments[i] = resolvedCachedArgument(beanName, cachedMethodArguments[i]);
      }
      return arguments;
   }
}
```

## @Resource 扫描与注入

- @Resouce注解的扫描依赖CommonAnnotationBeanPostProcessor后置处理器，该类实现了InstantiationAwareBeanPostProcessor后置处理器接口， **postProcessProperties执行依赖注入；**继承了InitDestroyAnnotationBeanPostProcessor类，该类实现了MergeBeanDefinitionBeanPostProcessor后置处理器， **postProcessMergeBeanDefinition执行了扫描注入点；**
- 该类不仅只有依赖注入功能， 也**负责扫描并执行@PostConstruct, @ProDestory注解的逻辑**



```java
//在加载类的时候，会给resourceAnnotationTypes资源注解类型集合里加入@Resource注解， @WebServiceRef注解；
static {
   webServiceRefClass = loadAnnotationType("javax.xml.ws.WebServiceRef");
   resourceAnnotationTypes.add(Resource.class);
   if (webServiceRefClass != null) {
      resourceAnnotationTypes.add(webServiceRefClass);
   }
}
```



#### 扫描注入点

```java
@Override
public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
	//扫描@PostConstruct， @PreDestory注解修饰的方法；
    super.postProcessMergedBeanDefinition(beanDefinition, beanType, beanName);
    //扫描@Resouce注解的注入点；
   InjectionMetadata metadata = findResourceMetadata(beanName, beanType, null);
   metadata.checkConfigMembers(beanDefinition);
}
```

1. 遍历类的所有属性；
2. 属性Field上是否存在@WebServiceRef， @EJB， @Resource注解，
3. 若Field是静态属性，则抛出异常IllegalStateException；
4. 若属性的类型是忽略类型ignoredResourceTypes中的一种，则不进行注入；否则包装Field创建ResourceElement对象，放入注入点集合currElements；
5. 遍历类的所有方法；
6. 若方法Method为桥接方法，则跳过；
7. 方法Method上不存在@Resource，则跳过；
8. 存在@Resource， 如果是静态方法，则抛出异常IllegalStateException；
9.  存在@Reource， 如果方法参数不为1， 则抛出异常IllegalStateException；
10. 存在@Resource， 判断方法参数类型是否为忽略类型的一种，是则跳过， 否则包装Method创建ResourceElement对象
11. 遍历完当前类后，会遍历父类的属性和方法，直到没有父类(知道父类为Object类)。
12. 将currElements集合包装为InjectionMetadata对象，并缓存；



#### 注入点注入

1. 若属性上存在@Resource，设置该属性Field可访问
2. getResourceToInject获取依赖Bean对象；
3. 反射给属性Field赋值；
4. 若方法参数存在@Resource， 设置该方法Method可访问；
5. getResourceToInject获取依赖Bean对象；
6. 反射调用方法赋值；

```java
//InjectionMetadata#inject
protected void inject(Object target, @Nullable String requestingBeanName, @Nullable PropertyValues pvs)
      throws Throwable {

   if (this.isField) {
      Field field = (Field) this.member;
      ReflectionUtils.makeAccessible(field);
      field.set(target, getResourceToInject(target, requestingBeanName));
   }
   else {
      if (checkPropertySkipping(pvs)) {
         return;
      }
      try {
         Method method = (Method) this.member;
         ReflectionUtils.makeAccessible(method);
         method.invoke(target, getResourceToInject(target, requestingBeanName));
      }
      catch (InvocationTargetException ex) {
         throw ex.getTargetException();
      }
   }
}
```

checkPropertySkipping： 判断是否跳过，若是想定义某个依赖的值， 通过给Bean的BeanDefinition的pvs添加目标依赖名，依赖的值，就可以跳过自动注入；



getResourceToInject：

1. 若为懒加载依赖注入，则返回AOP动态代理对象；调用时再获取target目标对象；
2. 若非懒加载依赖注入，调用getResource返回依赖bean对象；

```java
private class ResourceElement extends LookupElement {

   private final boolean lazyLookup;

   public ResourceElement(Member member, AnnotatedElement ae, @Nullable PropertyDescriptor pd) {
      super(member, pd);
      Resource resource = ae.getAnnotation(Resource.class);
      //获取@Resource的name属性值；
      String resourceName = resource.name();
      Class<?> resourceType = resource.type();
      //若@Resource值为空， isDefaultName为true;
      this.isDefaultName = !StringUtils.hasLength(resourceName);
      if (this.isDefaultName) {
         //若@Resource值为空，则采用默认值， 使用Field的属性名称作为名称；
         resourceName = this.member.getName();
         if (this.member instanceof Method && resourceName.startsWith("set") && resourceName.length() > 3) {
            resourceName = Introspector.decapitalize(resourceName.substring(3));
         }
      }
       //...
      this.name = (resourceName != null ? resourceName : "");
      Lazy lazy = ae.getAnnotation(Lazy.class);
      this.lazyLookup = (lazy != null && lazy.value());
   }

   @Override
   protected Object getResourceToInject(Object target, @Nullable String requestingBeanName) {
      return (this.lazyLookup ? buildLazyResourceProxy(this, requestingBeanName) :
            getResource(this, requestingBeanName));
   }
}
```

```java
//CommonAnnotationBeanPostProcessor#getResource
protected Object getResource(LookupElement element, @Nullable String requestingBeanName)
      throws NoSuchBeanDefinitionException {
   return autowireResource(this.resourceFactory, element, requestingBeanName);
}
```

CommonAnnotationBeanPostProcessor#autowireResouce:

- 获取@Resource的name值，若没有，则是默认值，isDefaultName也为true;

1. 若@Resource的name没有设置， 则isDefaultName为true；

   - 从BeanFactory#resolveDependency中获取依赖bean对象， 若没有，则报错；
     - 即name属性不设置、流程与@Autowired一样，byType+byName;

2. 若@Resource的name有设置, 则isDefaultName为false;

   - 根据name值从BeanFactory中获取bean对象，再判断是否为依赖的类型
   - 是则返回；不是则利用TypeConverter类型转换器进行转换，若结果为空或转换异常，则报错；

   - 即name属性有设置， 流程与@Autowired不一致， byName + byType;

```java
protected Object autowireResource(BeanFactory factory, LookupElement element, @Nullable String requestingBeanName)
      throws NoSuchBeanDefinitionException {

   Object resource;
   Set<String> autowiredBeanNames;
   String name = element.name;

   if (factory instanceof AutowireCapableBeanFactory) {
      AutowireCapableBeanFactory beanFactory = (AutowireCapableBeanFactory) factory;
      DependencyDescriptor descriptor = element.getDependencyDescriptor();
      if (this.fallbackToDefaultTypeMatch && element.isDefaultName && !factory.containsBean(name)) {
         autowiredBeanNames = new LinkedHashSet<>();
         resource = beanFactory.resolveDependency(descriptor, requestingBeanName, autowiredBeanNames, null);
         if (resource == null) {
            throw new NoSuchBeanDefinitionException(element.getLookupType(), "No resolvable resource object");
         }
      }
      else {
         resource = beanFactory.resolveBeanByName(name, descriptor);
         autowiredBeanNames = Collections.singleton(name);
      }
   }
   //...
   return resource;
}
```

```java
//DefaultListableBeanFactory#doGetBean, name为@Resource的name值， requireType为依赖类型；
protected <T> T doGetBean(
      String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly) throws BeansException {
   Object bean;

   Object sharedInstance = getSingleton(beanName);
   if (sharedInstance != null && args == null) {}
   else {
      try {
         RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
         checkMergedBeanDefinition(mbd, beanName, args);
         if (mbd.isSingleton()) {
            sharedInstance = getSingleton(beanName, () -> {
               try {
                  return createBean(beanName, mbd, args);
               }
               catch (BeansException ex) {}
            });
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
         }
      }
      catch (BeansException ex) {}
   }

   //如果传了requiredType， bean和requiredType不一致， 则会尝试类型转换；类型转换失败，则报错；
   if (requiredType != null && !requiredType.isInstance(bean)) {
      try {
         T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
         if (convertedBean ==
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
         }
         return convertedBean;
      }
      catch (TypeMismatchException ex) {
         throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
      }
   }
   return (T) bean;
}
```

