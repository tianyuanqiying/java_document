# Spring Security

## SpringSecurity配置使用

```java
	@Configuration
@EnableGlobalMethodSecurity(jsr250Enabled = true, prePostEnabled = true)
public class WebSecurityAnnotationConfig extends WebSecurityConfigurerAdapter{
    @Autowired
    UserServiceInDB userServiceInDB;
    @Autowired
    PasswordEncoder passwordEncoder;

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userServiceInDB);
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        //自定义登录页面、登录成功，登录失败； /user/index必须是POST请求
        http.formLogin().loginPage("/login.html").loginProcessingUrl("/user/login")
                .successForwardUrl("/user/index") //重定向到index.html, 因此登录成功，无需放行
                .failureForwardUrl("/user/toerror"); //重定向到error.html, 因此登录失败，需要放行/error.html，否则因为没有权限，再次调到登录页面
     http.authorizeRequests().antMatchers("/user/login","/login.html","/error.html").permitAll();
        //非登录的URL需要认证
        http.authorizeRequests().anyRequest().authenticated();

        http.csrf().disable();
    }

    @Override
    public void init(WebSecurity web) throws Exception {
        super.init(web);
    }

    @Override
    public void configure(WebSecurity web) throws Exception {
        super.configure(web);
    }
}
```

## 配置解密

### WEB配置解密



```xml
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

由于SpringBoot默认继承了springsecurity配置、其配置可在autoconfigure的spring.factories查看

```properties
org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration,\
org.springframework.boot.autoconfigure.security.servlet.UserDetailsServiceAutoConfiguration,\
org.springframework.boot.autoconfigure.security.servlet.SecurityFilterAutoConfiguration,\
```

SecurityAutoConfiguration作用： 通过@Bean注册认证事件发布器、 通过@Import导入SpringBootWebSecurityConfiguration， WebSecurityEnablerConfiguration，SecurityDataConfiguration

- DefaultAuthenticationEventPublisher ：认证事件发布器；
- **SpringBootWebSecurityConfiguration ： 若容器中不存在WebSecurityConfigurerAdapter类型的Bean,则注册默认实现DefaultConfigurerAdapter的Bean， 若我们自定义WebSecurityConfigurerAdapter, 则该默认配置不生效；**
- WebSecurityEnablerConfiguration:  主要是标记了@EnableWebSecurity；
  - @EnableWebSecurity ：SpringSecurity核心注解，通过@Import导入WebSecurityConfiguration，SpringWebMvcImportSelector，OAuth2ImportSelector； 注解上标记@EnableGlobalAuthentication注解；
    - **WebSecurityConfiguration** ：SpringSecurity核心配置类，在Bean的依赖注入阶段调用setFilterChainProxySecurityConfigurer方法；
      
      - DelegatingApplicationListener ：代理监听器、内部的监听器集合为空、没用到；
      - **Filter** ：注入Filter、Bean名称为**springSecurityFilterChain**
      - SecurityExpressionHandler：依赖**springSecurityFilterChain**，与角色前缀ROLE_有关；
      - WebInvocationPrivilegeEvaluator：依赖**springSecurityFilterChain**， 与权限角色有关；
      - **setFilterChainProxySecurityConfigurer** ： 标记了@Autowired， 配置类bean实例化依赖注入阶段执行该方法；
        - 实例化WebSecurity对象，并通过ApplicationContextAware的setApplicationContext进行初始化；
        - **给WebSecurity设置SecurityConfigurer对象,  若没有自定义SecurityConfigurer，则使用DefaultConfigurerAdapter， 有则使用自定义的SecurityConfigurer**
    - SpringWebMvcImportSelector： 通过selectImport方法注入WebMvcSecurityConfiguration配置类
      - WebMvcSecurityConfiguration ： 往容器中注入SpringMVC的方法参数解析器
        - AuthenticationPrincipalArgumentResolver ： 解析被@AuthenticationPrincipal标记的方法参数；
        - CsrfTokenArgumentResolver ：解析方法参数类型为CsrfToken的参数；
        - CurrentSecurityContextArgumentResolver：解析被@CurrentSecurityContext标记的方法参数；
      
    - @EnableGlobalAuthentication：通过@Import注解(AuthenticationConfiguration.class)类；
      
      - AuthenticationConfiguration ： 扫描SpringSecurity注解标记的方法或类、创建动态代理对对象
        - AuthenticationManagerBuilder ： 创建密码编码器、并设置给认证管理器；
        - GlobalAuthenticationConfigurerAdapter ： 用来获取类上存在EnableGlobalAuthentication注解的Bean;
        - InitializeUserDetailsBeanManagerConfigurer : 用来创建用户密码认证的基础组件DaoAuthenticationProvider



WebSecurity继承关系

![image-20240102082310427](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20240102082310427.png)

类型为Filter， Bean名称为springSecurityFilterChain的创建过程

- WebSecurityConfiguration#springSecurityFilterChain()

  - webSecurity.build() => this.object = doBuild();

    - AbstractConfiguredSecurityBuilder#init() ：WebSecurity父类AbstractConfiguredSecurityBuilder实现init方法

      - Collection<SecurityConfigurer<O, B>> configurers = getConfigurers() : 拿到配置类WebSecurityAnnotationConfig

      - configurer.init((B) this) ：init方法默认由父类WebSecurityConfigurerAdapter实现；

        -  HttpSecurity http = getHttp() ： 获取HttpSecurity, 若不存在，则创建

          - AuthenticationManager authenticationManager = authenticationManager() ：利用AuthenticationConfiguration中的认证构造器创建认证管理器；

          - http = new HttpSecurity(objectPostProcessor, authenticationBuilder,
                  sharedObjects) ： 创建HttpSecurity对象；

          - ```java
            http
               .csrf().and()
               .addFilter(new WebAsyncManagerIntegrationFilter())
               .exceptionHandling().and()
               .headers().and()
               .sessionManagement().and()
               .securityContext().and()
               .requestCache().and()
               .anonymous().and()
               .servletApi().and()
               .apply(new DefaultLoginPageConfigurer<>()).and()
               .logout();
            ```

            ```java
            private final LinkedHashMap<Class<? extends SecurityConfigurer<O, B>>, List<SecurityConfigurer<O, B>>> configurers = new LinkedHashMap<>();
            ```

            调用csrf(), exceptionHandling()方法都是往HttpSecurity属性configurers集合中放入配置类；

          - configure(http) :  由于自定义类WebSecurityAnnotationConfig重写了configure方法， 即调用formLogin(), http.authorizeRequests() 是往HttpSecurity属性configurers集合中放入配置类， http.csrf().disable() 把csrf的配置类从集合中删除；

        - ```java
          web.addSecurityFilterChainBuilder(http).postBuildAction(() -> {
             FilterSecurityInterceptor securityInterceptor = http
                   .getSharedObject(FilterSecurityInterceptor.class);
             web.securityInterceptor(securityInterceptor);
          });
          ```

          给WebSecurity设置securityFilterChainBuilders属性为http；

    - AbstractConfiguredSecurityBuilder#configure();

    - AbstractConfiguredSecurityBuilder#performBuild();WebSecurity实现了该方法

      - WebSecurity#performBuild()
        - securityFilterChainBuilder.build() : 调用HttpSecurity#builder(), 在AbstractSecurityBuilder实现该方法；
          - HttpSecurity#dobuild() : 在父类AbstractConfiguredSecurityBuilder实现该方法
            - init() ： 部分配置类会改init阶段创建过滤器；
            - configure();
              - Collection<SecurityConfigurer<O, B>> configurers = getConfigurers(); 获取HttpSecurity$configurer配置类集合；
              - configurer.configure((B) this) ： 调用每个配置类的configure方法；

      

    - **ExceptionHandlingConfigurer**#configure

      - ExceptionTranslationFilter exceptionTranslationFilter = new ExceptionTranslationFilter(
              entryPoint, getRequestCache(http)) ： 创建异常处理器过滤器；
      - AccessDeniedHandler deniedHandler = getAccessDeniedHandler(http) ： 获取403访问拒绝处理器，若没有自定义、则创建默认处理器；
      - exceptionTranslationFilter.setAccessDeniedHandler(deniedHandler);
      - http.addFilter(exceptionTranslationFilter) ： 给HttpSecurity$filters属性添加过滤器；

    - HeadersConfigurer#configure

      - HeaderWriterFilter headersFilter = createHeaderWriterFilter() ： 创建更新请求头过滤器；
      - http.addFilter(headersFilter); 给HttpSecurity$filters属性添加过滤器；

    - **SessionManagementConfigurer**#configure

      - SecurityContextRepository securityContextRepository = http
              .getSharedObject(SecurityContextRepository.class) ： 创建存储会话的库；
      - SessionManagementFilter sessionManagementFilter = new SessionManagementFilter(
              securityContextRepository, getSessionAuthenticationStrategy(http)) ： 创建会话管理过滤器；
      - InvalidSessionStrategy strategy = getInvalidSessionStrategy() ： 创建处理会话非法的策略
      - sessionManagementFilter.setInvalidSessionStrategy(strategy);
      - AuthenticationFailureHandler failureHandler = getSessionAuthenticationFailureHandler() ：获取会话认证失败处理器；
      - sessionManagementFilter.setAuthenticationFailureHandler(failureHandler);
      - http.addFilter(sessionManagementFilter) ：给HttpSecurity$filters属性添加过滤器；

    - **AnonymousConfigurer**#configure

      - http.addFilter(authenticationFilter); 在init阶段创建AnonymousAuthenticationFilter过滤器，给HttpSecurity$filters属性添加过滤器；

    - **DefaultLoginPageConfigurer**#configure

      - loginPageGeneratingFilter属性饥饿加载，直接new创建；
      - http.addFilter(loginPageGeneratingFilter) : 给HttpSecurity$filters属性添加过滤器；

    - **LogoutConfigurer**#configure

      - LogoutFilter logoutFilter = createLogoutFilter(http) ： 创建注销过滤器；
      - http.addFilter(logoutFilter) ：  给HttpSecurity$filters属性添加过滤器；

    - **FormLoginConfigurer**#configure：由父类AbstractAuthenticationFilterConfigurer实现方法

      - authFilter.setAuthenticationManager(http
              .getSharedObject(AuthenticationManager.class)); 设置认证管理器；
      - authFilter.setAuthenticationSuccessHandler(successHandler);
        authFilter.setAuthenticationFailureHandler(failureHandler) ：设置认证成功、失败处理器
      - http.addFilter(filter); 给HttpSecurity$filters属性添加过滤器；

    - **ExpressionUrlAuthorizationConfigurer**#configure

      - **FilterSecurityInterceptor** securityInterceptor = createFilterSecurityInterceptor(
              http, metadataSource, http.getSharedObject(AuthenticationManager.class)); 创建拦截器、用于权限判断；
      - http.addFilter(securityInterceptor); 给HttpSecurity$filters属性添加过滤器；

      

    - 最终调用每个配置类的configure方法、创建过滤器并将过滤器放入HttpSecurity$filters属性中；

    - 

      

      最终configurers集合中的配置如下：

      

      ![image-20240102085003841](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20240102085003841.png)



核心流程总结：

1. 解析WebSecurityConfiguration的@Bean方法后、往BeanFactory中注册了名为springSecurityFilterChain的核心Bean定义；
2. 实例化WebSecurityConfiguration后，在依赖注入阶段、会执行@Autowired方法setFilterChainProxySecurityConfigurer， 方法需要SecurityConfigurer参数，先去实例化自定义实现了WebSecurityConfigAdapter的类对象；
3. SecurityConfigurer集合参数设置给WebSecurity的configurers，在父类AbstractConfiguredSecurityBuilder定义了该属性；
4. 在IOC容器启动过程中、会实例化springSecurityFilterChain的Bean;
   1. 先执行WebSecurity#init初始化方法、会创建HttpSecurity对象、并调用属性configurers的init方法、先执行设置默认的http配置链、往HttpSecurity$configurers属性放入SecurityConfigurer配置类；再调用 我们自定义实现WebSecurityConfigAdapter的类的configure方法、 执行自定义的http配置连、也是往HttpSecurity$configurers属性新增或删除SecurityConfigurer配置类；
   2. 再执行WebSecurity#performBuild执行构建方法、会调用HttpSecurity#build方法过程、先调用HttpSecurity#init方法、获取HttpSecurity$configurers配置类属性并调用init方法、部分配置类会在该方法中创建过滤器； 再调用HttpSecurity#configure方法、获取HttpSecurity$configurers配置类属性并调用configure方法、大部分配置类会在该方法中创建过滤器、并放入HttpSecurity$filters集合中；



### 注解配置解密

使用SpringSecurity权限注解时、需要通过 

```java
@EnableGlobalMethodSecurity(jsr250Enabled = true, prePostEnabled = true, securedEnabled = true)
```

该注解往容器中注入GlobalMethodSecuritySelector；

```java
@Import({ GlobalMethodSecuritySelector.class })
@EnableGlobalAuthentication
@Configuration
public @interface EnableGlobalMethodSecurity {
```

在Spring容器启动过程中，ConfigurationClassPostProcessor会解析@Import注解，拿到GlobalMethodSecuritySelector类的Bean定义、实例化后、调用Bean的selectImport方法；

- GlobalMethodSecuritySelector#selectImport

  - Class<?> importingClass = ClassUtils
          .resolveClassName(importingClassMetadata.getClassName(),
                ClassUtils.getDefaultClassLoader()) ： 拿到被@EnableGlobalMethodSecurity标记的类信息；

  - boolean **skipMethodSecurityConfiguration** = GlobalMethodSecurityConfiguration.class
          .isAssignableFrom(importingClass) ： 判断是否为GlobalMethodSecurityConfiguration， 为false;

  - if (!**skipMethodSecurityConfiguration**) {classNames.add(GlobalMethodSecurityConfiguration.class.getName());}

    往容器中注入GlobalMethodSecurityConfiguration配置、负责处理@PreAuthorize、@PostAuthorize等注解；

  - if (jsr250Enabled) {
       classNames.add(Jsr250MetadataSourceConfiguration.class.getName());
    } ： 往容器中注入Jsr250MetadataSourceConfiguration， 负责处理@HasRole, @DenyAll, @permitAll注解；

  - classNames.add(MethodSecurityMetadataSourceAdvisorRegistrar.class.getName()) ： 往容器中注入MethodSecurityMetadataSourceAdvisorRegistrar， 负责创建AOP的Advisor;

#### Jsr250MetadataSourceConfiguration

Jsr250MethodSecurityMetadataSource ： 往容器注入工具类，用来扫描DenyAll， PermitAll、RolesAllowed



#### GlobalMethodSecurityConfiguration

解析该配置类时、会往容器中注入SpringSecurity AOP 的基础组件；

##### MethodSecurityMetadataSource

该类的作用主要用于扫描方法注解、是一个工具类；

- GlobalMethodSecurityConfiguration#methodSecurityMetadataSource() 

  - List<MethodSecurityMetadataSource> sources = new ArrayList<>();

  - if (isPrePostEnabled)  ：查看EnableGlobalMethodSecurity注解的prePostEnabled属性为true

    - sources.add(new PrePostAnnotationSecurityMetadataSource(attributeFactory)) ： 负责扫描PreFilter，PreAuthorize，PostFilter，PostAuthorize注解标记的类或方法

  - if (isSecuredEnabled)  ： 查看EnableGlobalMethodSecurity的securedEnabled属性为true
       sources.add(new SecuredAnnotationSecurityMetadataSource()); 负责扫描@Secured注解

  - if (isJsr250Enabled) ： 查看EnableGlobalMethodSecurity的isJsr250Enabled属性为true
       sources.add(jsr250MethodSecurityMetadataSource) ： 负责扫描@DenyAll， @PermitAll， @RolesAllowed注解；

  - ```java
    return new DelegatingMethodSecurityMetadataSource(sources);
    ```

##### AccessDecisionManager

bean名称为accessDecisionManager， 仲裁权限判断；

- GlobalMethodSecurityConfiguration#accessDecisionManager()

  - List<AccessDecisionVoter<?>> decisionVoters = new ArrayList<>();

  - decisionVoters .add(new PreInvocationAuthorizationAdviceVoter(expressionAdvice)); 

    负责仲裁判断@PreFilter，PreAuthorize，PostFilter，PostAuthorize；

  - decisionVoters.add(new Jsr250Voter());

    负责扫描@DenyAll， @PermitAll， @RolesAllowed注解；

  - RoleVoter roleVoter = new RoleVoter();

  - decisionVoters.add(roleVoter);

  - decisionVoters.add(new AuthenticatedVoter());

  - return new AffirmativeBased(decisionVoters);

    

##### MethodInterceptor 

bean名称为methodSecurityInterceptor、 对应AOP中的Advise通知对象，执行环绕增强处理；

- GlobalMethodSecurityConfiguration#methodSecurityInterceptor
  - this.methodSecurityInterceptor = new MethodSecurityInterceptor();
  - methodSecurityInterceptor.setAccessDecisionManager(accessDecisionManager());
  - methodSecurityInterceptor.setAfterInvocationManager(afterInvocationManager());
  - methodSecurityInterceptor.setSecurityMetadataSource(methodSecurityMetadataSource);

增强功能依赖accessDecisionManager， methodSecurityMetadataSource实现；



#### MethodSecurityMetadataSourceAdvisorRegistrar

实现了ImportBeanDefinitionRegistrar接口、spring容器启动过程， ConfigurationClassPostProcessor扫描到后、会调用ImportBeanDefinitionRegistrar#registerBeanDefinitions

- ImportBeanDefinitionRegistrar#registerBeanDefinitions
  - BeanDefinitionBuilder advisor = BeanDefinitionBuilder
          .rootBeanDefinition(MethodSecurityMetadataSourceAdvisor.class);
  - advisor.addConstructorArgValue("methodSecurityInterceptor") ： 构造第一个参数值为methodSecurityInterceptor
  - advisor.addConstructorArgReference("methodSecurityMetadataSource") ：构造第二参数依赖methodSecurityMetadataSource
  - advisor.addConstructorArgValue("methodSecurityMetadataSource") ： 构造器第三个参数值为methodSecurityMetadataSource
  - registry.registerBeanDefinition("metaDataSourceAdvisor", advisor.getBeanDefinition()); 往容器放入MethodSecurityMetadataSourceAdvisor的Bean定义、负责处理权限注解的增强逻辑；

#### 相关接口

##### MethodSecurityMetadataSourceAdvisor

- 该Bean在创建时、属性饥饿加载属性pointcut = new MethodSecurityMetadataSourcePointcut()， 

- 该类为AOP的Advisor角色，在Spring的后置处理阶段、会调用该类的pointcut的匹配方法、pointcut 利用MetadataSource工具类判断是否存在权限注解、若是存在、则创建AOP动态代理对象；
- 当调用接口时、先进入动态代理对象的增强业务逻辑、最终Advisor的Advise、即MethodInterceptor处理；

##### MethodSecurityInterceptor

SpringSecurity的Advise、实现了权限判断的增强逻辑；

- MethodSecurityInterceptor#invoke, 通常会在super.beforeInvocation(mi)进行权限判断

  - ```
    public Object invoke(MethodInvocation mi) throws Throwable {
       InterceptorStatusToken token = super.beforeInvocation(mi);
    
       Object result;
       try {
          result = mi.proceed();
       }
       finally {
          super.finallyInvocation(token);
       }
       return super.afterInvocation(token, result);
    }
    ```

##### PrePostAnnotationSecurityMetadataSource

负责扫描PreFilter，PreAuthorize，PostFilter，PostAuthorize注解标记的类或方法

##### SecuredAnnotationSecurityMetadataSource

负责扫描@Secure注解

##### jsr250MethodSecurityMetadataSource

负责扫描@DenyAll， @PermitAll， @RolesAllowed注解；

## 过滤器链执行

未登录情况下、访问需要登录权限的资源、会自动跳转到登录页面、流程如下：

1. WebAsyncManagerIntegrationFilter#doFilter 

   - 会给异步管理器注册拦截器SecurityContextCallableProcessingInterceptor,  暂时不知道作用

2. SecurityContextPersistenceFilter#doFilter

   - 若未登录、则会创建匿名会话，将会话相关的信息放入SecurityContextHolder中

3. HeaderWriterFilter#doFilter

   - 给请求头中设置org.springframework.security.web.header.HeaderWriterFilter@1e1237ab.FILTERED为true

4. LogoutFilter#doFilter

   - 若请求为注销，默认请求url为/logout
     -  遍历调用注销处理器LogoutHandler#logout处理=>扩展点， 
     - 调用LogoutSuccessHandler#onLogoutSuccess处理注销成功后需要执行的业务操作=>扩展点

5. **UsernamePasswordAuthenticationFilter**#doFilter， 由父类实现doFilter方法

   - 请求url 匹配指定的url("/user/login")，
     - **匹配则执行认证逻辑**
     - 不匹配则执行下一个Filter

6. DefaultLoginPageGeneratingFilter#doFilter 

   - String loginPageHtml = generateLoginPageHtml(request, loginError,logoutSuccess);
   - response.getWriter().write(loginPageHtml);
   - **拦截登录url请求、默认为/login,  且请求方式为GET方法、是则生成默认的HTML页面文本、然后写入response；**

7. RequestCacheAwareFilter#doFilter

   - 暂时不知作用

8. SecurityContextHolderAwareRequestFilter#doFilter

   - 暂时不知作用

9. AnonymousAuthenticationFilter#doFilter

   - 未登录情况下、用户信息为空、会往SecurityContextHolder的context中设置匿名用户另外AnonymousAuthenticationToken；

10. SessionManagementFilter#doFilter

    - 未登录情况下、 会判断是否设置会话非法策略InvalidSessionStrategy

      - 若有、则调用InvalidSessionStrategy#onInvalidSessionDetected处理， 直接return， 因此若设置了InvalidSessionStrategy，但不做任何处理、则页面无响应、所以、需要在InvalidSessionStrategy设置转发或者重定向；

        ```
        http.sessionManagement().invalidSessionStrategy(new EmptyInvalidSessionStrategy());
        ```

        ![image-20240102140241302](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20240102140241302.png)

      - 若没有、则调用下一个过滤处理

11. **ExceptionTranslationFilter**#doFilter

    - trycatch 捕捉chain.doFilter(request, response)抛出的异常
      - 捕捉到FilterSecurityInterceptor抛出的AccessDeniedException异常、则调用handleSpringSecurityException处理SpringSecurity的异常；
        - 若异常为AuthenticationException类型、则
        - 若异常为AccessDeniedException类型、
          - **若用户是匿名用户、则调用authenticationEntryPoint#commerce处理(扩展点自定义处理)**
            - 默认情况下、构造redirectUrl、 默认值为/login, 再发起重定向请求 => 重新走一遍过滤器链到DefaultLoginPageGeneratingFilter#doFilter中生成HTML返回；
              - 对应配置：http.formLogin().permitAll();
          - 若用户非匿名用户、已经登录、则获取AccessDeniedHandler处理器处理无权访问；

12. **FilterSecurityInterceptor**#doFilter

    - InterceptorStatusToken token = super.beforeInvocation(fi) :  权限仲裁判断
      - 若无权访问则抛出访问拒绝异常org.springframework.security.access.**AccessDeniedException**: Access is denied
      - 若有权限执行、则执行fi.getChain().doFilter(fi.getRequest(), fi.getResponse()) 、执行我们的业务逻辑；
    - super.afterInvocation(token, null) ： 后置处理增强；



Spring Security本质就是一条过滤器链；

![image-20240103021722222](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20240103021722222.png)

### SecurityContextPersistenceFilter

两个主要职责：请求来临时，创建 `SecurityContext`安全上下文信息，请求结束时清空SecurityContextHolder。过滤器负责核心的处理流程，存储安全上下文和读取安全上下文的工作完全委托给了HttpSessionSecurityContextRepository去处理

```java
//SecurityContextPersistenceFilter#doFilter

HttpRequestResponseHolder holder = new HttpRequestResponseHolder(request,
				response);
//从Session中获取安全上下文信息,不存在创建一个新的SecurityContext
SecurityContext contextBeforeChainExecution = repo.loadContext(holder);

try {
    //请求开始时，设置安全上下文信息
    SecurityContextHolder.setContext(contextBeforeChainExecution);

    chain.doFilter(holder.getRequest(), holder.getResponse());

}
finally {
    //请求结束后，清空安全上下文信息
    SecurityContext contextAfterChainExecution = SecurityContextHolder
        .getContext();
```

### UsernamePasswordAuthenticationFilter

表单提交了username和password，被封装成token进行一系列的认证，便是主要通过这个过滤器完成的，在表单认证的方法中，这是最最关键的过滤器。 

```java
// AbstractAuthenticationProcessingFilter#doFilter

Authentication authResult;

try {
    //  调用UsernamePasswordAuthenticationFilter的attemptAuthentication方法
    authResult = attemptAuthentication(request, response);
    if (authResult == null) {
        // return immediately as subclass has indicated that it hasn't completed
        // authentication
        //子类未完成认证，立刻返回
        return;
    }
    sessionStrategy.onAuthentication(authResult, request, response);
}
catch (InternalAuthenticationServiceException failed) {
    unsuccessfulAuthentication(request, response, failed);
    return;
}
catch (AuthenticationException failed) {
    unsuccessfulAuthentication(request, response, failed);
    return;
}

// Authentication success
if (continueChainBeforeSuccessfulAuthentication) {
    chain.doFilter(request, response);
}

successfulAuthentication(request, response, chain, authResult);


// UsernamePasswordAuthenticationFilter#attemptAuthentication
public Authentication attemptAuthentication(HttpServletRequest request,
			HttpServletResponse response) throws AuthenticationException {
		if (postOnly && !request.getMethod().equals("POST")) {
			throw new AuthenticationServiceException(
					"Authentication method not supported: " + request.getMethod());
		}

		String username = obtainUsername(request);
		String password = obtainPassword(request);

		username = username.trim();
		//将认证信息封装成token, 当前认证状态是false
		UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
				username, password);

		// Allow subclasses to set the "details" property
		setDetails(request, authRequest);
		// 通过AuthenticationManager去认证，并返回认证信息
		return this.getAuthenticationManager().authenticate(authRequest);
	}
```

### ExceptionTranslationFilter

ExceptionTranslationFilter异常转换过滤器位于整个springSecurityFilterChain的后方，用来转换整个链路中出现的异常。此过滤器本身不处理异常，而是将认证过程中出现的异常交给内部维护的一些类去处理，一般处理两大类异常：AccessDeniedException访问异常和AuthenticationException认证异常。

### FilterSecurityInterceptor

FilterSecurityInterceptor从SecurityContextHolder中获取Authentication对象，然后比对用户拥有的权限和资源所需的权限。这是一个方法级的权限过滤器, 基本位于过滤链的最底部 。这个过滤器决定了访问特定路径应该具备的权限，访问的用户的角色，权限是什么？访问的路径需要什么样的角色和权限？这些判断和处理都是由该类进行的。





## 登录认证



### 自定义登录页面

用户访问需要权限的资源时、检测到用户未登录、则跳转到我们自己写的页面

login.html、放在static目录下

```html
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<form action="/user/login" method="post">
    用户名:<input type="text" name="username"/><br/>
    密码： <input type="password" name="password"/><br/>
    <input type="checkbox" name="remember-me" value="true"/><br/>
    <input type="submit" value="提交"/>
</form>
</body>
</html>
```

修改config配置类

```java
@Configuration
public class WebSecurityLoginConfig extends WebSecurityConfigurerAdapter{
    @Autowired
    UserServiceInDB userServiceInDB;

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userServiceInDB);
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        //自定义登录页面、登录成功，登录失败； /user/index必须是POST请求
        http.formLogin().loginPage("/login.html").loginProcessingUrl("/user/login")
                .successForwardUrl("/user/index") //重定向到index.html, 因此登录成功，无需放行
                .failureForwardUrl("/user/toerror"); //重定向到error.html, 因此登录失败，需要放行/error.html，否则因为没有权限，再次调到登录页面

http.authorizeRequests().antMatchers("/user/login","/login.html","/error.html").permitAll();
        //非登录的URL需要认证
        http.authorizeRequests().anyRequest().authenticated();

        http.csrf().disable();
    }
}
```



为什么设置了login.html后、DefaultLoginPageGeneratingFilter就不生效了？



- FormLoginConfigurer#loginPage("/login.html")

  - AbstractAuthenticationFilterConfigurer#loginPage(loginPage)
    - this.loginPage = loginPage ;   设置登录页面路径属性；
    - this.authenticationEntryPoint = new LoginUrlAuthenticationEntryPoint(loginPage) ： 设置point的登录页面路径；
    - this.customLoginPage = true; 设置自定义登录页面标志为true;

  

- 在HttpSecurity的init阶段、会调用FormLoginConfigurer，DefaultLoginPageConfigurer#init方法

  - DefaultLoginPageConfigurer#init
    - http.setSharedObject(DefaultLoginPageGeneratingFilter.class,
            loginPageGeneratingFilter) ： 将生成登录页面的Filter放入共享对象集合中；
  - FormLoginConfigurer#init
    - FormLoginConfigurer#initDefaultLoginFilter(http);
      - DefaultLoginPageGeneratingFilter loginPageGeneratingFilter = http
              .getSharedObject(DefaultLoginPageGeneratingFilter.class) ： 从共享对象集合中获取Filter
        - **!isCustomLoginPage() : 若自定义登录页面、则formLoginEnable为false， 否则设置为true**
          -  loginPageGeneratingFilter.setFormLoginEnabled(true);
          -  loginPageGeneratingFilter.setLoginPageUrl(getLoginPage()) : 设置自定义的登录页面路径
          -  loginPageGeneratingFilter.setAuthenticationUrl(getLoginProcessingUrl()) ： 设置处理请求路径

- 在HttpSecurity的configure阶段、 会调用DefaultLoginPageConfigurer#configure方法

  - DefaultLoginPageConfigurer#configure
    - if (loginPageGeneratingFilter.isEnabled() && authenticationEntryPoint == null) ： 由于开启了自定义、因此loginPageGeneratingFilter.isEnabled() 返回为false , 因此不会讲Filter放入过滤器链；
      - http.addFilter(loginPageGeneratingFilter);

  **因此若是设置自定义登录页面、 DefaultLoginPageGeneratingFilter 是不生效的**

  

  

- 项目启动后、发送需要权限的请求、会跳转到登录页面；

  - FilterSecurityInterceptor#doFilter :  权限判断、 权限不足、抛出AccessDeniedException异常；

    - ExceptionTranslationFilter#doFilter : 捕获抛出的AccessDeniedException异常；

    - ExceptionTranslationFilter#handleSpringSecurityException(request, response, chain, ase) ： 处理异常

      - 异常类型为AccessDeniedException

      - Authentication authentication = SecurityContextHolder.getContext().getAuthentication(); 未登录，因此是匿名会话

      - ExceptionTranslationFilter#sendStartAuthentication 

        - authenticationEntryPoint.commence(request, response, reason) ： 实现类为**LoginUrlAuthenticationEntryPoint**

          - redirectUrl = buildRedirectUrlToLoginPage(request, response, authException)

            拿到登录页面路径 /login.html, 再拼接协议、ip， 端口号出重定向url;

          - redirectStrategy.sendRedirect(request, response, redirectUrl) : 发起重定向请求

- 最终SpringMVC请求到login.html资源返回；





### 登录请求处理流程

在表单登录HTML页面中、提交按钮请求的URL为 /user/login

```html
<form action="/user/login" method="post">
```

后台config配置：

```java
http.formLogin().loginPage("/login.html").loginProcessingUrl("/user/login")
        .successForwardUrl("/user/index") //重定向到index.html, 因此登录成功，无需放行
        .failureForwardUrl("/user/toerror");
```

在登陆页面输入用户密码，登录后、后台都没有登录接口、为什么就处理成功了？

- HttpSecurity#formLogin 
  - getOrApply(new FormLoginConfigurer<>());
    - FormLoginConfigurer#<init> 
      - super(new **UsernamePasswordAuthenticationFilter**(), null);
        - **this.authFilter = authenticationFilter;**

- FormLoginConfigurer#loginProcessingUrl、 由父类AbstractAuthenticationFilterConfigurer实现
  - this.loginProcessingUrl = loginProcessingUrl ： 设置登录请求处理路径
  - authFilter.setRequiresAuthenticationRequestMatcher(createLoginProcessingUrlMatcher(loginProcessingUrl))  ： 设置匹配器的匹配路径为loginProcessingUrl, authFilter类型为UsernamePasswordAuthenticationFilter;
    - new AntPathRequestMatcher(loginProcessingUrl, "POST") ： 请求方式为POST



- UsernamePasswordAuthenticationFilter#doFilter, 有父类AbstractAuthenticationProcessingFilter实现

  - !requiresAuthentication(request, response) ： request请求路径为/user/login， 匹配成功

    - authResult = attemptAuthentication(request, response) ：尝试认证，由子类实现

      - String username = obtainUsername(request);
        String password = obtainPassword(request) ： 从请求中获取账户密码；
      - **UsernamePasswordAuthenticationToken** authRequest = new UsernamePasswordAuthenticationToken(  username, password);  账户密码包装为认证token
        - *setAuthenticated(false)* : token的认证状态为false 、 未登录成功
      - this.getAuthenticationManager().authenticate(authRequest) ： 获取认证管理器发起认证， 类型为ProviderManager, 即调用ProviderManager#authenticate
        - for (AuthenticationProvider provider : getProviders()) ： 遍历providers, 当前provider类型为AnonymousAuthenticationProvider，不处理当前token;
        - parent.authenticate(authentication) ： parent类型还是ProviderManager, 调用ProviderManager#authenticate
          - for (AuthenticationProvider provider : getProviders()) ： 遍历providers, 当前provider类型为DaoAuthenticationProvider
            - **DaoAuthenticationProvider#authenticate(authentication) ： 用户登录认证**

    - sessionStrategy.onAuthentication(authResult, request, response) ： 正在认证

      - **调用SessionAuthenticationStrategy#onAuthentication => 业务可利用的扩展点增强功能**

    - unsuccessfulAuthentication(request, response, failed) ： 认证失败处理

      - **调用AuthenticationFailureHandler#onAuthenticationFailure => 认证失败的扩展点增强功能**

    - successfulAuthentication(request, response, chain, authResult) ： 认证成功处理

      - **SecurityContextHolder.getContext().setAuthentication(authResult) ： 会话信息放入Context、本质为ThreadLocal, 存储类型为SecurityContext, SecurityContext包含Authentication**

      - **调用AuthenticationSuccessHandler#onAuthenticationSuccess=> 认证成功的扩展点增强功能**

        - 默认实现为SimpleUrlAuthenticationSuccessHandler#onAuthenticationSuccess

        - redirectStrategy.sendRedirect(request, response, targetUrl) ：重定向策略发送重定向，默认实现类为DefaultDirectStrategy,

          

- **DaoAuthenticationProvider**##authenticate(authentication)

  - UserDetails user = this.userCache.getUserFromCache(username) ： 从缓存中获取用户信息
  - user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);
  - preAuthenticationChecks.check(user) ： 检查用户状态、是否被锁、是否被禁用、是否账号过期
  - additionalAuthenticationChecks(user,  (UsernamePasswordAuthenticationToken) authentication) ： 执行密码校验
    - String presentedPassword = authentication.getCredentials().toString() ： 拿出请求中的密码
    - !**passwordEncoder**.matches(presentedPassword, userDetails.getPassword()) ： 利用密码校验工具、判断请求密码和用户密码是否匹配；
      - 不匹配则抛出异常throw new BadCredentialsException
  - postAuthenticationChecks.check(user) ： 校验密码是否过期；
  - createSuccessAuthentication(principalToReturn, authentication, user) ： 登录成功、创建用户会话
    - UsernamePasswordAuthenticationToken result = new UsernamePasswordAuthenticationToken(principal, authentication.getCredentials()，      authoritiesMapper.mapAuthorities(user.getAuthorities())) ： 创建用户密码Token
      - super.setAuthenticated(true) : 设置状态为认证成功
    - result.setDetails(authentication.getDetails()) ： 设置用户信息

![image-20240102222009353](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20240102222009353.png)



##### 会话处理







#### 自定义登录成功/失败处理器

###### AuthenticationSuccessHandler

实现AuthenticationSuccessHandler接口：

```java
public class LoginAuthenticationSuccessHandler implements AuthenticationSuccessHandler {
    private String forwardUrl;

    public LoginAuthenticationSuccessHandler() {
    }

    public LoginAuthenticationSuccessHandler(String forwardUrl) {
        this.forwardUrl = forwardUrl;
    }

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
        request.getRequestDispatcher(forwardUrl).forward(request, response);
    }
}
```

###### AuthenticationFailureHandler

实现AuthenticationFailureHandler接口

```java
public class LoginAuthenticationFailHandler implements AuthenticationFailureHandler {
    private String forwardUrl;

    public LoginAuthenticationFailHandler() {
    }

    public LoginAuthenticationFailHandler(String forwardUrl) {
        this.forwardUrl = forwardUrl;
    }


    @Override
    public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response, AuthenticationException exception) throws IOException, ServletException {
        request.getRequestDispatcher(forwardUrl).forward(request, response);
    }
}
```

修改config文件

```java
http.formLogin().loginPage("/login.html")
        .loginProcessingUrl("/user/login")
        .successHandler(new LoginAuthenticationSuccessHandler("/user/index"))
        .failureHandler(new LoginAuthenticationFailHandler("/user/toerror"));
```

#### 认证接口

##### **AuthenticationManager**

```java
public interface AuthenticationManager {
	Authentication authenticate(Authentication authentication)
			throws AuthenticationException;
}
```

![image-20240102220205097](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20240102220205097.png)



###### ProviderManager

ProviderManager是 `AuthenticationManager` 的一个实现类，提供了基本的认证逻辑和方法；它包含了一个List<AuthenticationProvider>属性，通过 AuthenticationProvider 接口来扩展出多种认证方式，实际上这是委托者模式的应用（Delegate）。

```java
public interface AuthenticationProvider {
	
	Authentication authenticate(Authentication authentication)
			throws AuthenticationException;

	boolean supports(Class<?> authentication);
}
```

![image-20240102220301325](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20240102220301325.png)

###### DaoAuthenticationProvider



在Spring Security中，提交的用户名和密码，被封装成UsernamePasswordAuthenticationToken，而根据用户名加载用户的任务则是交给了UserDetailsService，在DaoAuthenticationProvider中，对应的方法便是retrieveUser，返回一个UserDetails。

![image-20240102220417622](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20240102220417622.png)

##### Authentication

Authentication在spring security中是最高级别的身份/认证的抽象，由这个顶级接口，我们可以得到用户拥有的权限信息列表，密码，用户细节信息，用户身份信息，认证信息。

###### UsernamePasswordAuthenticationToken

`UsernamePasswordAuthenticationToken`实现了 `Authentication`主要是将用户输入的用户名和密码进行封装，并供给 `AuthenticationManager` 进行验证；验证完成以后将返回一个认证成功的 `Authentication` 对象

![image-20240102220858066](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20240102220858066.png)



```java
public interface Authentication extends Principal, Serializable {
	//1.权限信息列表，可使用AuthorityUtils.commaSeparatedStringToAuthorityList("admin,ROLE_ADMIN")返回字符串权限集合
	Collection<? extends GrantedAuthority> getAuthorities();
	//2.密码信息，用户输入的密码字符串，在认证过后通常会被移除，用于保障安全。
	Object getCredentials();
	//3.认证时包含的一些信息，web应用中的实现接口通常为 WebAuthenticationDetails，它记录了访问者的ip地址和sessionId的值。
	Object getDetails();
	//4.身份信息，大部分情况下返回的是UserDetails接口的实现类
	Object getPrincipal();
	//5.是否被认证，认证为true	
	boolean isAuthenticated();
	//6.设置是否能被认证
	void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException;
}
```

###### SecurityContextHolder

用于存储安全上下文（security context）的信息，`SecurityContextHolder`默认使用`ThreadLocal` 策略来存储认证信息。

```java
// 获取当前用户名
Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();

if (principal instanceof UserDetails) {
    String username = ((UserDetails)principal).getUsername();
} else {
    String username = principal.toString();
}
```

##### UserDetailsService

```java
public interface UserDetailsService {
   // 根据用户名加载用户信息
   UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

Spring Security内置了两种 UserDetailsManager实现

![image-20240102221111535](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20240102221111535.png)

实际项目中，我们更多采用调用 `AuthenticationManagerBuilder#userDetailsService(userDetailsService)` 方法，使用自定义实现的 UserDetailsService实现类，更加灵活且自由的实现认证的用户信息的读取

```java
@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    auth.userDetailsService(userServiceInDB);
}
```

```java
@Service
public class UserServiceInDB implements UserDetailsService{
    @Autowired
    UserMapper userMapper;
    @Autowired
    PermissionMapper permissionMapper;

    @Override
    public SysUser getByUsername(String username) {
        return userMapper.getByUsername(username);
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        SysUser user = getByUsername(username);
        if (Objects.isNull(user)) {
            throw new RuntimeException("user not exist");
        }
        List<GrantedAuthority> authorities = CollectionUtil.newArrayList();
        List<SysPermission> permissions = permissionMapper.selectByUserId(user.getId());
        permissions.forEach(permission -> {
            SimpleGrantedAuthority simpleGrantedAuthority = new SimpleGrantedAuthority(permission.getEnname());
            authorities.add(simpleGrantedAuthority);
        });
        return new User(user.getUsername(), user.getPassword(), authorities);
    }
}
```

##### UserDetails

用户信息核心接口，默认实现类org.springframework.security.core.userdetails.User

![image-20240102221326335](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20240102221326335.png)

##### PasswordEncoder

```java
public interface PasswordEncoder {

	/**
	 * 表示把密码按照特定的解析规则进行解析
	 */
	String encode(CharSequence rawPassword);

	/**
	 * 表示验证从存储中获取的编码密码与编码后提交的原始密码是否匹配。如果密码匹配，则返回 true；如果不匹配，	 * 则返回 false。第一个参数表示需要被解析的密码。第二个参数表示存储的密码
	 */
	boolean matches(CharSequence rawPassword, String encodedPassword);
    
}
```

![image-20240102221407064](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20240102221407064.png)

BCryptPasswordEncoder 是 Spring Security 官方推荐的密码解析器 。BCryptPasswordEncoder 是对 bcrypt 强散列方法的具体实现，是基于Hash算法实现的单向加密，可以通过strength控制加密强度，默认 10。

```java
@Test
public void test(){
    String passwd = BCrypt.hashpw("123",BCrypt.gensalt());
    System.out.println(passwd);

    boolean checkpw = BCrypt.checkpw("123", passwd);
    System.out.println(checkpw);
}
```



### 注销请求处理流程

当我们输入/logout路径后、自动就退出了

- requiresLogout(request, response) ： 注销路径匹配、默认为/logout

  - this.handler.logout(request, response, auth) ： handler为CompositeLogoutHandler，包装了LogoutHandler集合

    - for (LogoutHandler handler : this.logoutHandlers) handler.logout(request, response, authentication)  ： 遍历LogoutHandler#logout方法、处理退出逻辑，默认情况下存在两个Handler=>SecurityContextLogoutHandler, LogoutSuccessEventPublishingLogoutHandler
      - SecurityContextLogoutHandler#logout
        - HttpSession session = request.getSession(false) ：Http会话设置为失效
        - SecurityContext context = SecurityContextHolder.getContext();
          context.setAuthentication(null) : 清理ThreadLocal的SecurityContext 的用户会话信息
        - SecurityContextHolder.clearContext() ： 删除ThreadLocal的SecurityContext信息

  - logoutSuccessHandler.onLogoutSuccess(request, response, auth) :  调用注销成功处理器、扩展点默认处理器为SimpleUrlLogoutSuccessHandler

    - SimpleUrlLogoutSuccessHandler#onLogoutSuccess

      - String targetUrl = determineTargetUrl(request, response, authentication) : 获取注销后跳转的目标路径、默认为登录页面；
      - redirectStrategy.sendRedirect(request, response, targetUrl) ： 重定向到登录页面；

      

#### 自定义注销处理器

##### LogoutHandler

业务中、注销时通常需要清理用户临时数据、可以实现LogoutHandler完成增强、如

```java
public class UserLogoutHandler implements LogoutHandler {
    @Override
    public void logout(HttpServletRequest request, HttpServletResponse response, Authentication authentication) {
        //删除用户相关的一些临时数据；
        System.out.println("delete user info ... [" + JSONUtil.toJsonStr(authentication) + "]");
    }
}
```

##### LogoutSuccessHandler

业务中、注销成功后、在跳转页面前需要做一些业务处理、例如请求头设置等、可实现LogoutSuccessHandler实现增强

```java
public class UserLogoutSuccessHandler implements LogoutSuccessHandler {
    private String forwardUrl;

    public UserLogoutSuccessHandler() {
    }

    public UserLogoutSuccessHandler(String forwardUrl) {
        this.forwardUrl = forwardUrl;
    }

    @Override
    public void onLogoutSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
        //清理用户数据；
        System.out.println("user [" + JSONUtil.toJsonStr(authentication)+ "] logout success" );
        request.getRequestDispatcher(forwardUrl).forward(request,response);
    }
}
```

修改config配置类

```java
//退出登录,推出成功跳转业面；
http.logout().logoutSuccessUrl("/login.html").addLogoutHandler(new UserLogoutHandler()).logoutSuccessHandler(new UserLogoutSuccessHandler("/login.html"));
```

### 会话处理

#### 设置HttpSession

调用AuthenticationSuccessHandler#onAuthenticationSuccess处理

- 默认实现为SimpleUrlAuthenticationSuccessHandler#onAuthenticationSuccess
- redirectStrategy.sendRedirect(request, response, targetUrl) ：重定向策略发送重定向，默认实现类为DefaultDirectStrategy,
  - response.sendRedirect(redirectUrl) ： response类型为HeaderWriterResponse, 由父类为OnCommittedResponseWrapper#sendRedirect方法
  - HttpSessionSecurityContextRepository#onResponseCommitted ：由父类实现该方法SaveContextOnUpdateOrErrorResponseWrapper
    - **HttpSessionSecurityContextRepository#saveContext(SecurityContextHolder.getContext()) **
      - final Authentication authentication = context.getAuthentication() ： 获取用户会话
      - HttpSession httpSession = request.getSession(false) : 为空
      - httpSession = createNewSessionIfAllowed(context) ： 创建HttpSession会话信息
      - httpSession.setAttribute(springSecurityContextKey, context) ： 往tomcat的HttpSession中放入包含用户会话信息的context， key为springSecurityContextKey



**总结： 登录成功后重定向过程中、利用response#sendRedirect方法、调用HttpSessionSecurityContextRepository#saveContext方法、往HttpSession的attribute中放入key为springSecurityContextKey，value为SecurityContext的键值对；**



#### 获取HttpSession

浏览器登录认证成功后， 发送其他的请求给服务器

- SecurityContextPersistenceFilter#doFilter
  - **SecurityContext contextBeforeChainExecution = repo.loadContext(holder); 从HttpSessionSecurityContextRepository获取用户会话信息**
    - HttpSession httpSession = request.getSession(false);
    - SecurityContext context = readSecurityContextFromSession(httpSession);
      - Object contextFromSession = httpSession.getAttribute(springSecurityContextKey) ： 从HttpSession中根据key为springSecurityContextKey从attributes中获取用户会话context;
      - return (SecurityContext) contextFromSession;
  - SecurityContextHolder.setContext(contextBeforeChainExecution);



**总结：在登录认证成功后、访问其他URL时、SecurityContextPersistenceFilter过滤器调用HttpSessionSecurityContextRepository从HttpSession根据key为springSecurityContextKey获取用户会话SecurityContext,  再设置给SecurityContextHolder上下文、即ThreadLocal对象；**



因此、同个浏览器登录认证成功后、后续访问其他的url都能拿到用户会话信息；本质依赖tomcat的HttpSession机制；



#### 会话并发控制

用户在这个手机登录后，他又在另一个手机登录相同账户，对于之前登录的账户`是否需要被挤兑，或者说在第二次登录时限制它登录`，更或者像腾讯视频VIP账号一样，最多只能五个人同时登录，第六个人限制登录。

- maximumSessions：最大会话数量，设置为1表示一个用户只能有一个会话
- expiredSessionStrategy：会话过期策略

```java
http.sessionManagement()
                .invalidSessionUrl("/session/invalid")
                .maximumSessions(1)
                .expiredSessionStrategy(new MyExpiredSessionStrategy());
```

```java
public class MyExpiredSessionStrategy implements SessionInformationExpiredStrategy {
    @Override
    public void onExpiredSessionDetected(SessionInformationExpiredEvent event) throws IOException, ServletException {
        HttpServletResponse response = event.getResponse();
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write("您已被挤兑下线！");
    }
}
```

测试

1. 使用chrome浏览器，先登录，再访问http://localhost:8080/admin/index
2. 使用ie浏览器，再登录，再访问http://localhost:8080/admin/index
3. 使用chrome浏览器，重新访问http://localhost:8080/admin/index，会执行expiredSessionStrategy，页面上显示”您已被挤兑下线！“



阻止用户第二次登录

sessionManagement也可以配置 maxSessionsPreventsLogin：boolean值，当达到maximumSessions设置的最大会话个数时阻止登录。

````java
http.sessionManagement()
                .invalidSessionUrl("/session/invalid")
                .maximumSessions(1)
                .expiredSessionStrategy(new MyExpiredSessionStrategy())
                .maxSessionsPreventsLogin(true);
````

##### 会话控制

我们可以通过以下选项准确控制会话何时创建以及Spring Security如何与之交互：  

| 机制       | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| always     | 如果session不存在总是需要创建                                |
| ifRequired | 如果需要就创建一个session（默认）登录时                      |
| never      | Spring Security 将不会创建session，但是如果应用中其他地方创建了session，那么Spring Security将会使用它 |
| stateless  | Spring Security将绝对不会创建session，也不使用session。并且它会暗示不使用cookie，所以每个请求都需要重新进行身份验证。这种无状态架构适用于REST API及其无状态认证机制。 |

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.sessionManagement()
    .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
}
```



#### 会话机制隐藏问题

缺点：多节点部署场景下、不适用，会出现会话不一致问题；

![image-20240103020403732](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20240103020403732.png)

利用redis解决共享会话问题；

```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>

<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>3.1.0</version>
</dependency>
```

修改application.yaml

```yml
spring:
  session:
    store-type: redis
  redis:
    host: localhost
    port: 6379

server:
  port: 8080
  servlet:
    session:
      timeout: 600
```

启动两个服务8080，8081 ，其中一个登录后访问http://localhost:8080/admin/index，另外一个不需要登录就可以访问

缺点：pring Session + Redis实现分布式Session共享 有个非常大的缺陷, 无法实现跨域名共享session , 只能在单台服务器上共享session , 因为是依赖cookie做的 , cookie 无法跨域。 Spring Session一般是用于多台服务器负载均衡时共享Session的，都是同一个域名，不会跨域。



## 用户授权（访问控制）

权的方式包括 web授权和方法授权，web授权是通过 url拦截进行授权，方法授权是通过 方法拦截进行授权。他
们都会调用accessDecisionManager进行授权决策，若为web授权则拦截器为FilterSecurityInterceptor；若为方
法授权则拦截器为MethodSecurityInterceptor。如果同时通过web授权和方法授权则先执行web授权，再执行方
法授权，最后决策通过，则允许访问资源，否则将禁止访问。    

![image-20240103023131286](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20240103023131286.png)

**Web授权本质是通过Filter判断的、而方法授权是通过AOP判断的， 在Servlet层面执行、因此若是Web层不通过， 就直接返回，不会执行方法授权的判断；**



### web授权

Spring Security可以通过 http.authorizeRequests() 对web请求进行授权保护 ，Spring Security使用标准Filter建立了对web请求的拦截，最终实现对资源的授权访问。

```javva
http.authorizeRequests()
            //设置哪些路径可以直接访问，不需要认证
            .antMatchers("/user/login","/login.html").permitAll()
            .anyRequest().authenticated();  //需要认证才能访问
```

#### 访问控制的url匹配

在配置类中http.authorizeRequests() 主要是对url进行控制。配置顺序会影响之后授权的效果，越是具体的应该放在前面，越是笼统的应该放到后面。  

#### anyRequest()

表示匹配所有的请求。一般情况下此方法都会使用，设置全部内容都需要进行认证，会放在最后。  

```java
.anyRequest().authenticated()
```

#### antMatchers

```java
public C antMatchers(String... antPatterns)
```

参数是不定向参数，每个参数是一个 ant 表达式，用于匹配 URL规则。

| 通配符 | 说明                    |
| ------ | ----------------------- |
| ?      | 匹配任何单字符          |
| *      | 匹配0或者任意数量的字符 |
| **     | 匹配0或者更多的目录     |

```java
// 放行 js和css 目录下所有的文件
.antMatchers("/js/**","/css/**").permitAll()
// 只要是.js 文件都放行
.antMatchers("/**/*.js").permitAll()  
```

#### regexMatchers()  

使用正则表达式进行匹配。

```java
//所有以.js 结尾的文件都被放行
.regexMatchers( ".+[.]js").permitAll()
```

无论是 antMatchers() 还是 regexMatchers() 都具有两个参数的方法，其中第一个参数都是HttpMethod ，表示请求方式，当设置了 HttpMethod 后表示只有设定的特定的请求方式才执行对应的权限设置。

```java
.antMatchers(HttpMethod.POST,"/admin/demo").permitAll()
.regexMatchers(HttpMethod.GET,".+[.]jpg").permitAll()
```

#### RequestMatcher接口

`RequestMatcher`是`Spring Security Web`的一个概念模型接口，用于抽象建模对`HttpServletRequest`请求的匹配器这一概念。`Spring Security`内置提供了一些`RequestMatcher`实现类

| 实现类                  | 介绍                                |
| ----------------------- | ----------------------------------- |
| `AnyRequestMatcher`     | 匹配任何请求                        |
| `AntPathRequestMatcher` | 使用`ant`风格的路径匹配模板匹配请求 |

#### 内置的访问控制

- 【常用】`#permitAll()` 方法，所有用户可访问。

- 【常用】`#denyAll()` 方法，所有用户不可访问。

- 【常用】`#authenticated()` 方法，登录用户可访问。

- `#anonymous()` 方法，无需登录，即匿名用户可访问。

- `#rememberMe()` 方法，通过 remember me登录的用户可访问。

- `#fullyAuthenticated()` 方法，非 remember me 登录的用户可访问。

- `#hasIpAddress(String ipaddressExpression)` 方法，来自指定 IP 表达式的用户可访问。

- 【常用】`#hasRole(String role)` 方法， 拥有指定角色的用户可访问，角色将被增加 “ROLE_”   前缀。

- 【常用】`#hasAnyRole(String... roles)` 方法，拥有指定任一角色的用户可访问。

- 【常用】`#hasAuthority(String authority)` 方法，拥有指定权限(`authority`)的用户可访问。

- 【常用】`#hasAuthority(String... authorities)` 方法，拥有指定任一权限(`authority`)的用户可访问。

- 【最牛】`#access(String attribute)` 方法，当 Spring EL 表达式的执行结果为 `true` 时，可以访问。

#### 基于权限的访问控制

除了之前讲解的内置权限控制。Spring Security 中还支持很多其他权限控制。这些方法一般都用于用户已经被认证后，判断用户是否具有特定的要求。  

###### hasAuthority(String)  

判断用户是否具有特定的权限，用户的权限是在自定义登录逻辑中创建 User 对象时指定的。权限名称大小写敏感

```java
 return new User("fox", pw, AuthorityUtils.commaSeparatedStringToAuthorityList("admin,user"));//admin,user就是用户的权限
```

 在配置类中通过 hasAuthority(“admin”)设置具有 admin 权限时才能访问。  

```java
.antMatchers("/admin/demo").hasAuthority("admin")
```

否则报403错误

![image-20240103024224352](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20240103024224352.png)

###### hasAnyAuthority(String ...)  

如果用户具备给定权限中某一个，就允许访问。  

```
.antMatchers("/admin/demo").hasAnyAuthority("admin","System")
```



#### 基于角色的访问控制

##### hasRole(String)  

如果用户具备给定角色就允许访问，否则出现 403。参数取值来源于自定义登录逻辑 UserDetailsService 实现类中创建 User 对象时给 User 赋予的授权。
在给用户赋予角色时角色需要以： `ROLE_`开头 ，后面添加角色名称。例如：ROLE_admin 其中 admin是角
色名，`ROLE_`是固定的字符开头。

```java
return new User("fox", pw, AuthorityUtils.commaSeparatedStringToAuthorityList("ROLE_admin,user"));//给用户赋予admin角色
```

使用 hasRole()时参数也只写admin 即可，否则启动报错。  

```java
.antMatchers("/admin/demo").hasRole("admin")
```

##### hasAnyRole(String ...)

如果用户具备给定角色的任意一个，就允许被访问 。



#### 基于表达式的访问控制

###### access(表达式)

之前学习的登录用户权限判断实际上底层实现都是调用access(表达式)  

https://docs.spring.io/spring-security/site/docs/5.2.7.RELEASE/reference/htmlsingle/#tech-intro-access-control

表达式根对象的基类是SecurityExpressionRoot，提供了一些在web和方法安全性中都可用的通用表达式。

![image-20240103025434964](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20240103025434964.png)

可以通过 access() 实现和之前学习的权限控制完成相同的功能。

```java
.antMatchers("/user/login","/login.html").access("permitAll")
.antMatchers("/admin/demo").access("hasAuthority('System')") 
```

##### 自定义方法

判断登录用户是否具有访问当前 URL 的权限。

```java
@Component
public class MySecurityExpression implements MySecurityExpressionOperations{
    @Override
    public boolean hasPermission(HttpServletRequest request, Authentication authentication) {
        // 获取主体
        Object obj = authentication.getPrincipal();
        if (obj instanceof UserDetails){
            UserDetails userDetails = (UserDetails) obj;
            //
            String name = request.getParameter("name");
            //获取权限
            Collection<? extends GrantedAuthority> authorities = userDetails.getAuthorities();
            //判断name值是否在权限中
            return authorities.contains(new SimpleGrantedAuthority(name));
        }
        return false;
    }
}
```

在 access 中通过bean的beanName.方法(参数)的形式进行调用：

```java
.anyRequest().access("@mySecurityExpression.hasPermission(request,authentication)")
```

### 自定义403处理方案

使用 Spring Security 时经常会看见 403（无权限）。Spring Security 支持自定义权限受限处理，需要实现 AccessDeniedHandler接口

```java
public class MyAccessDeniedHandler implements AccessDeniedHandler {
    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) throws IOException, ServletException {
        response.setStatus(HttpServletResponse.SC_FORBIDDEN);
        response.setHeader("Content-Type", "application/json;charset=utf-8");
        PrintWriter out = response.getWriter();
        out.write("{\"status\":\"error\",\"msg\":\"权限不足，请联系管理员！\"}");
        out.flush();
        out.close();
    }
}
```

```java
http.exceptionHandling()
     .accessDeniedHandler(new MyAccessDeniedHandler());
```



### 方法授权

#### 基于注解的访问控制  

Spring Security在方法的权限控制上支持三种类型的注解，JSR-250注解、@Secured注解和支持表达式的注解。这三种注解默认都是没有启用的，需要通过@EnableGlobalMethodSecurity来进行启用。

这些注解可以写到 Service 接口或方法上，也可以写到 Controller或 Controller 的方法上。通常情况下都是写在控制器方法上的，控制接口URL是否允许被访问。  

##### JSR-250注解

###### **@RolesAllowed**

表示访问对应方法时所应该具有的角色。其可以标注在类上，也可以标注在方法上，当标注在类上时表示其中所有方法的执行都需要对应的角色，当标注在方法上表示执行该方法时所需要的角色，当方法和类上都使用了@RolesAllowed进行标注，则方法上的@RolesAllowed将覆盖类上的@RolesAllowed，即方法上@RolesAllowed将对当前方法起作用。@RolesAllowed的值是由角色名称组成的数组。

###### **@PermitAll**

表示允许所有的角色进行访问，也就是说不进行权限控制。@PermitAll可以标注在方法上也可以标注在类上，当标注在方法上时则只对对应方法不进行权限控制，而标注在类上时表示对类里面所有的方法都不进行权限控制。（1）当@PermitAll标注在类上，而@RolesAllowed标注在方法上时则按照@RolesAllowed将覆盖@PermitAll，即需要@RolesAllowed对应的角色才能访问。

（2）当@RolesAllowed标注在类上，而@PermitAll标注在方法上时则对应的方法也是不进行权限控制的。

（3）当在类和方法上同时使用了@PermitAll和@RolesAllowed时先定义的将发生作用（这个没多大的实际意义，实际应用中不会有这样的定义）。

###### **@DenyAll**

是和PermitAll相反的，表示无论什么角色都不能访问。@DenyAll只能定义在方法上。你可能会有疑问使用@DenyAll标注的方法无论拥有什么权限都不能访问，那还定义它干啥呢？使用@DenyAll定义的方法只是在我们的权限控制中不能访问，脱离了权限控制还是可以访问的。

开启注解
在启动类或者在配置类上添加 @EnableGlobalMethodSecurity(jsr250Enabled = true)

```java
@EnableGlobalMethodSecurity(jsr250Enabled = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {
```

#### 支持表达式的注解

Spring Security中定义了四个支持使用表达式的注解，分别是@PreAuthorize、@PostAuthorize、@PreFilter和@PostFilter。其中前两者可以用来在方法调用前或者调用后进行权限检查，后两者可以用来对集合类型的参数或者返回值进行过滤。

```java
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {
```

```java
//@PreAuthorize("hasRole('ROLE_ADMIN')")
//@PreAuthorize("hasRole('ROLE_USER') or hasRole('ROLE_ADMIN')")
//限制只能查询Id小于10的用户
@PreAuthorize("#id<10")
@RequestMapping("/findById")
public User findById(long id) {
    User user = new User();
    user.setId(id);
    return user;
}


// 限制只能查询自己的信息
@PreAuthorize("principal.username.equals(#username)")
@RequestMapping("/findByName")
public User findByName(String username) {
    User user = new User();
    user.setUsername(username);
    return user;
}

//限制只能新增用户名称为abc的用户
@PreAuthorize("#user.username.equals('abc')")
@RequestMapping("/add")
public User add(User user) {
    return user;
}
```



#### Web授权原理

重写 `#configure(HttpSecurity http)` 方法，主要配置 URL 的权限控制

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http
            // 配置请求地址的权限
            .authorizeRequests()
                .antMatchers("/test/echo").permitAll() // 所有用户可访问
                .antMatchers("/test/admin").hasRole("ADMIN") // 需要 ADMIN 角色
                .antMatchers("/test/normal").access("hasRole('ROLE_NORMAL')") // 需要 NORMAL 角色。
                // 任何请求，访问的用户都需要经过认证
                .anyRequest().authenticated()
            .and()
            // 设置 Form 表单登录
        	//自定义登录页面，可以通过 #loginPage(String loginPage) 设置
            .formLogin()
//                    .loginPage("/login") // 登录 URL 地址
                .permitAll() // 所有用户可访问
            .and()
            // 配置退出相关
            .logout()
//                    .logoutUrl("/logout") // 退出 URL 地址
                .permitAll(); // 所有用户可访问
}
```

```java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {
```

```java
@RestController
@RequestMapping("/demo")
public class DemoController {

    @PermitAll
    @GetMapping("/echo")
    public String demo() {
        return "示例返回";
    }

    @GetMapping("/home")
    public String home() {
        return "我是首页";
    }

    @PreAuthorize("hasRole('ROLE_ADMIN')")
    @GetMapping("/admin")
    public String admin() {
        return "我是管理员";
    }

    @PreAuthorize("hasRole('ROLE_NORMAL')")
    @GetMapping("/normal")
    public String normal() {
        return "我是普通用户";
    }

}
```

##### Web授权流程图

![image-20240103033400041](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20240103033400041.png)

1. 拦截请求，已认证用户访问受保护的web资源将被SecurityFilterChain中的 FilterSecurityInterceptor 的子
   类拦截。

2. 获取资源访问策略，FilterSecurityInterceptor会从 SecurityMetadataSource 的子类
   DefaultFilterInvocationSecurityMetadataSource 获取要访问当前资源所需要的权限Collection<ConfigAttribute> 。SecurityMetadataSource其实就是读取访问策略的抽象，而读取的内容，其实就是我们配置的访问规则  

3. 最后，FilterSecurityInterceptor会调用 AccessDecisionManager 进行授权决策，若决策通过，则允许访问资
   源，否则将禁止访问。  

   

若是注解授权过程、则入口为MethodSecurityInterceptor、后续流程大体一致；



#### 注解授权原理

```java
//MethodSecurityInterceptor#invoke
public Object invoke(MethodInvocation mi) throws Throwable {
    InterceptorStatusToken token = super.beforeInvocation(mi);

    Object result;
    try {
        result = mi.proceed();
    }
    finally {
        super.finallyInvocation(token);
    }
    return super.afterInvocation(token, result);
}
```



#### 相关接口

##### AccessDecisionManager

ccessDecisionManager采用投票的方式来确定是否能够访问受保护资源。  AccessDecisionManager中包含的一系列AccessDecisionVoter将会被用来对Authentication是否有权访问受保护对象进行投票，AccessDecisionManager根据投票结果，做出最终决策 。

```java
public interface AccessDecisionManager {
	// ~ Methods
	// ========================================================================================================

	/**
	 * 用来鉴定当前用户是否有访问对应受保护资源的权限
	 * authentication：要访问资源的访问者的身份
	 * object：要访问的受保护资源，web请求对应FilterInvocation
	 * configAttributes：是受保护资源的访问策略，通过SecurityMetadataSource获取	 
	 */
	void decide(Authentication authentication, Object object,
			Collection<ConfigAttribute> configAttributes) throws AccessDeniedException,
			InsufficientAuthenticationException;

	boolean supports(ConfigAttribute attribute);

	boolean supports(Class<?> clazz);
}
```

![image-20240103033653212](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20240103033653212.png)

###### AffirmativeBased

AffirmativeBased的逻辑是：
（1）只要有AccessDecisionVoter的投票为ACCESS_GRANTED则同意用户进行访问；
（2）如果全部弃权也表示通过；
（3）如果没有一个人投赞成票，但是有人投反对票，则将抛出AccessDeniedException。
Spring security默认使用的是AffirmativeBased。

###### ConsensusBased

ConsensusBased的逻辑是：
（1）如果赞成票多于反对票则表示通过。
（2）反过来，如果反对票多于赞成票则将抛出AccessDeniedException。
（3）如果赞成票与反对票相同且不等于0，并且属性allowIfEqualGrantedDeniedDecisions的值为true，则表
示通过，否则将抛出异常AccessDeniedException。参数allowIfEqualGrantedDeniedDecisions的值默认为true。
（4）如果所有的AccessDecisionVoter都弃权了，则将视参数allowIfAllAbstainDecisions的值而定，如果该值
为true则表示通过，否则将抛出异常AccessDeniedException。参数allowIfAllAbstainDecisions的值默认为false。

###### UnanimousBased

UnanimousBased的逻辑与另外两种实现有点不一样，另外两种会一次性把受保护对象的配置属性全部传递
给AccessDecisionVoter进行投票，而UnanimousBased会一次只传递一个ConfigAttribute给
AccessDecisionVoter进行投票。这也就意味着如果我们的AccessDecisionVoter的逻辑是只要传递进来的
ConfigAttribute中有一个能够匹配则投赞成票，但是放到UnanimousBased中其投票结果就不一定是赞成了。
UnanimousBased的逻辑具体来说是这样的：
（1）如果受保护对象配置的某一个ConfigAttribute被任意的AccessDecisionVoter反对了，则将抛出
AccessDeniedException。
（2）如果没有反对票，但是有赞成票，则表示通过。
（3）如果全部弃权了，则将视参数allowIfAllAbstainDecisions的值而定，true则通过，false则抛出
AccessDeniedException。  

##### AccessDecisionVoter

```java
public interface AccessDecisionVoter<S> {

	int ACCESS_GRANTED = 1; //同意
	int ACCESS_ABSTAIN = 0; //弃权
	int ACCESS_DENIED = -1; //拒绝

	boolean supports(ConfigAttribute attribute);

	boolean supports(Class<?> clazz);
	// 返回结果是AccessDecisionVoter中定义的三个常量之一
	int vote(Authentication authentication, S object,
			Collection<ConfigAttribute> attributes);
}
```

![image-20240103033943772](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20240103033943772.png)

##### MethodSecurityInterceptor

Spring Security提供了两类AbstractSecurityInterceptor，基于AOP Alliance的MethodSecurityInterceptor，和基于Aspectj继承自MethodSecurityInterceptor的AspectJMethodSecurityInterceptor

```java
//MethodSecurityInterceptor#invoke
public Object invoke(MethodInvocation mi) throws Throwable {
    InterceptorStatusToken token = super.beforeInvocation(mi);

    Object result;
    try {
        result = mi.proceed();
    }
    finally {
        super.finallyInvocation(token);
    }
    return super.afterInvocation(token, result);
}
```

![image-20240103034033859](C:\Users\user\AppData\Roaming\Typora\typora-user-images\image-20240103034033859.png)