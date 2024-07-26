# Spring整合Mybatis原理

## 整合核心思路

很多框架需要和Spring整合，核心思想就是 将框架的核心组件对象放入Spring容器中，使其成为Bean;

比如Mybatis, Mybatis可以单独使用，而单独使用Mybatis框架需要用到Mybatis所提供的的一些类对象，使用这些对象就可以使用Mybatis提供的功能， 与Spring整合为了把这些对象放入Spring容器成为Bean, 只要成为Bean, Spring项目中就可方便使用Mybatis功能了；

## Mybatis-Spring 1.3 底层源码执行流程

1. 通过@MapperScan指定扫描目录， 且导入MapperScannerRegistrar类

2. MapperScannerRegistrar实现了ImportBeanDefinitionRegistrar接口， Spring加载创建Bean定义时，会调用registerBeanDefinitions方法；

3. 获取@MapperScan注解信息；创建并赋值ClassPathMapperScanner，调用doScan方法扫描Mapper

   - 重写isCandiddateComponent方法，若是独立接口，则返回true;

   - 添加includeFilter, 添加一个TypeFilter，该filter#match返回true;
   - ClasspathBeanDefinitionScanner几处判断Bean方法， 需要重写，否则校验不通过；

4. 调用父类ClasspathBeanDefinitionScanner#scan方法扫描并注册BeanDefinition

5. 再次遍历扫描到的BeanDefinition集合，修改BeanDefinition的属性

- 设置构造方法参数值为接口类的全限定名
  - 需要保存原有接口的信息
- 设置BeanDefinition的类型为MapperFactoryBean类, 该类实现了FactoryBean接口；
  - 接口类是无法实例化，对接口动态代理，生成动态代理对象；
- 设置BeanDefinition的autowireMode自动注入类型为 byType类型；

6. Bean的生命周期， 生成Bean对象， 调用MapperFactoryBean#getObject方法生成动态代理对象；

- MapperFactoryBean的AutowireMode为byType，所以Spring会自动调用set方法，有两个set方法，一个setSqlSessionFactory，一个setSqlSessionTemplate，而这两个方法执行的前提是根据方法参数类型能找到对应的bean，所以Spring容器中要存在SqlSessionFactory类型的bean或者SqlSessionTemplate类型的bean。创建SqlSessionTemplate方法时，会生成sqlSessionProxy代理对象（Mybatis用的）；
- getObject获取SqlSession, 调用SqlSessionTempldate#getMapper方法；
- 生成MapperProxy对象，实现了InvocationHandler, 保存了SqlSessionTemplate对象；
- JDK动态代理生成代理对象， 传入代理接口类， 传入MapperProxy对象；（Spring用的）

7. 生命周期结束后，容器中存在的Mapper对象为代理对象；

### Mapper代理对象执行

以SelectOne为例，查询一条数据库数据。

1. 调用代理对象会调用MapperProxy#invoke方法；

2. 调用sqlSession#selectOne方法，代理对象创建时， sqlSession对象的类型为SqlSessionTempldate，因此会调用SqlSessionTempldate#selectOne方法

3. 调用sqlSessionProxy代理对象的方法， 经过SqlSessionInterceptor拦截器InvocationHandler处理；

   - 获取SqlSession对象， 类型为DefaultSqlSession(Mybatis用的)
     - 若方法上存在@Transactional事务注解，会从ThreadLocal中获取SqlSession, 若获取不到，则创建DefaultSqlSession， 再放入ThreadLocal中； 
       - DefaultSqlSession存在并发问题，因此线程不安全，通过ThreadLocal保证线程安全；
     - 若不存在@Transactional注解，则每次查询都生成DefaultSqlSession对象，解决线程安全问题；

   - 反射调用method方法；
   - 判断是否开启事务，开启则提交事务





### Mybatis一级缓存

Mybatis的一级缓存是基于SqlSession实现的，因此，执行同一SQL时，使用同一个SqlSession对象， 就可利用一级缓存，提高Sql执行效率；

**Mybatis一级缓存失效**

1. 如果业务方法上不存在@Transactional注解， 则每次获取SqlSession都创建一个新的SqlSession对象，必然导致缓存不生效；如果开启@Transactional， 则该事物范围内，多条SQL使用同个SqlSession， 一级缓存会生效；
2. Spring整合Mybatis后，一级缓存失效并不是问题；若不开启事务， 则每条SQL都相当于一个事务， 执行完就提交了。在没有开启Spring事务的时候，SqlSession的一级缓存并不是**失效**了，而是存在的生命周期太短了(一条SQL的执行时间) ； 另外， 若是数据库事务级别为读未提交， 两条相同的SQL查询的数据可能不一样了，此时的一级缓存就是脏数据了。





## Mybatis-Spring 2.0.6版本整合

1. 通过@MapperScan导入MapperScanRegistrar类；
2. Spring启动调用MapperScanRegistrar#registerBeanDefinitions方法
3. 在registerBeanDefinitions中定义了MapperScanConfigurer的BeanDefinition加入BeanFactory:
4. MapperScanConfigurer实现BeanDefinitionRegistryPostProcessor， 启动过程中，调用postProcessBeanDefinitionRegistry方法；
5. 在postProcessBeanDefinitionRegistry中，创建ClasspathMapperScanner对象，扫描并加载Mapper接口；

Spring整合Mybatis之后SQL执行流程：
https://www.processon.com/view/link/6152cc385653bb6791db436c





```java
@MapperScan("com.redis.zset.weibo.test")
```

```java
@Import(MapperScannerRegistrar.class)
public @interface MapperScan {}
```

```java
public class MapperScannerRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware {

  @Override
  public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

    AnnotationAttributes annoAttrs = AnnotationAttributes.fromMap(importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName()));
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);

    Class<? extends Annotation> annotationClass = annoAttrs.getClass("annotationClass");
    if (!Annotation.class.equals(annotationClass)) {
      scanner.setAnnotationClass(annotationClass);
    }

    Class<?> markerInterface = annoAttrs.getClass("markerInterface");
    if (!Class.class.equals(markerInterface)) {
      scanner.setMarkerInterface(markerInterface);
    }
    List<String> basePackages = new ArrayList<String>();
    for (String pkg : annoAttrs.getStringArray("value")) {
      if (StringUtils.hasText(pkg)) {
        basePackages.add(pkg);
      }
    }

    scanner.registerFilters();
    scanner.doScan(StringUtils.toStringArray(basePackages));
  }
```

```java
public class ClassPathMapperScanner extends ClassPathBeanDefinitionScanner {
      @Override
  public Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
      logger.warn("No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.");
    } else {
      processBeanDefinitions(beanDefinitions);
    }

    return beanDefinitions;
  }
    
  private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    for (BeanDefinitionHolder holder : beanDefinitions) {
      definition = (GenericBeanDefinition) holder.getBeanDefinition();

 	definition.getConstructorArgumentValues()
     .addGenericArgumentValue(definition.getBeanClassName()); // issue #59
        definition.setBeanClass(this.mapperFactoryBean.getClass());

        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
      }
    }
  }
	
   public void registerFilters() {
      boolean acceptAllInterfaces = true;
	//...
      if (acceptAllInterfaces) {
        addIncludeFilter(new TypeFilter() {
          @Override
          public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
            return true;
          }
        });
      }
    }


  @Override
  protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
    return beanDefinition.getMetadata().isInterface() && beanDefinition.getMetadata().isIndependent();
  }
 
@Override
  protected boolean checkCandidate(String beanName, BeanDefinition beanDefinition) {
    if (super.checkCandidate(beanName, beanDefinition)) {
      return true;
    } else {
      logger.warn("Skipping MapperFactoryBean with name '" + beanName 
          + "' and '" + beanDefinition.getBeanClassName() + "' mapperInterface"
          + ". Bean already defined with the same name!");
      return false;
    }
  }
}
```

