# SpringBoot应用(二)

## 	1、请求参数常用注解

1. `@PathVariable` : 路径变量
2. `@RequestHeader` 获取请求头
3. `@RequestParam` 获取请求参数（指问号后的参数，url?a=1&b=2）
4. `@CookieValue` 获取Cookie值
5. `@RequestAttribute` 获取request域属性
6. `@RequestBody` 获取请求体[POST]
7. `@MatrixVariable` 矩阵变量

使用：

```java
@RestController
public class ParameterTestController {


    //  car/2/owner/zhangsan
    @GetMapping("/car/{id}/owner/{username}")
    public Map<String,Object> getCar(@PathVariable("id") Integer id,
                                     @PathVariable("username") String name,
                                     @PathVariable Map<String,String> pv,
                                     @RequestHeader("User-Agent") String userAgent,
                                     @RequestHeader Map<String,String> header,
                                     @RequestParam("age") Integer age,
                                     @RequestParam("inters") List<String> inters,
                                     @RequestParam Map<String,String> params,
                                     @CookieValue("_ga") String _ga,
                                     @CookieValue("_ga") Cookie cookie){

        Map<String,Object> map = new HashMap<>();

//        map.put("id",id);
//        map.put("name",name);
//        map.put("pv",pv);
//        map.put("userAgent",userAgent);
//        map.put("headers",header);
        map.put("age",age);
        map.put("inters",inters);
        map.put("params",params);
        map.put("_ga",_ga);
        System.out.println(cookie.getName()+"===>"+cookie.getValue());
        return map;
    }


    @PostMapping("/save")
    public Map postMethod(@RequestBody String content){
        Map<String,Object> map = new HashMap<>();
        map.put("content",content);
        return map;
    }
}
```



### @RequestAttribute

```java
@Controller
public class AnnotateController {
    @GetMapping("/goto")
    public String gotoSuccess(HttpServletRequest request, HttpServletResponse response) {
        request.setAttribute("code", 1000);
        request.setAttribute("msg", "success");
        Cookie cookie = new Cookie("test", "wolaile");
        response.addCookie(cookie);
        return "forward:/success";
    }

    @ResponseBody
    @GetMapping("/success")
    public Object success(@RequestAttribute Integer code, @RequestAttribute String msg, HttpServletRequest request, @CookieValue("test") String cookieVal) {
        HashMap<Object, Object> hashMap = new HashMap<>();
        hashMap.put("code", code);
        hashMap.put("msg", msg);
        System.out.println(cookieVal);
        return hashMap;
    }

}
```

### Redirect

```java
@GetMapping("/redirect")
public String redirect(Map<String,Object> map,
                        Model model,
                        HttpServletRequest request,
                        HttpServletResponse response){
    //下面三位都是可以给request域中放数据
    map.put("hello","world666");
    model.addAttribute("world","hello666");
    request.setAttribute("message","HelloWorld");

    return "redirect:https://search.bilibili.com/";
}
```

redirect后可以跳转到任何页面，而不会限制服务器资源；

### HttpEntity

```java
@ResponseBody
@PostMapping("/entity")
public String entity(HttpEntity<Pet> pet){
    //下面三位都是可以给request域中放数据

    Pet body = pet.getBody();
    HttpHeaders headers = pet.getHeaders();
    System.out.println(body);
    System.out.println(headers);
    return "wolaile";
}
```

HttpEntity<T>: 泛型T 决定 我们目标参数类型，另外，会将Http头部信息封装进其属性headers中，类型为HttpHeaders; ===> 

相当于@RequestBody Pet pet + @RequestHeader HttpHeaders headers组合； 

```java
@ResponseBody
@PostMapping("/entity2")
public String entity2(@RequestBody Pet pet, @RequestHeader HttpHeaders headers){
    //下面三位都是可以给request域中放数据

    System.out.println(pet);
    System.out.println(headers);
    return "wolaile";
}
```

### @ResponseBody

```java
@ResponseBody
@PostMapping("/respBod")
public String entity2(){
    return "wolaile";
}
```

1. 如果是复杂类型(BeanUtils.isSimpleTpye())会将返回值转换为JSON字符串返回；
2. 如果是字符串和普通类型返回值，则直接返回；

### ResponseEntity

```java
@PostMapping("/respEntitiy")
public ResponseEntity<Pet> resEntity(){
    Pet pet = new Pet();
    pet.setAge("1");
    pet.setName("2");
    ResponseEntity<Pet> petResponseEntity = new ResponseEntity<>(pet, HttpStatus.OK);
    return petResponseEntity;
}

@GetMapping("/something")
public ResponseEntity<String> handle() {
    String body = ... ;
    String etag = ... ;
    return ResponseEntity.ok().eTag(etag).body(body);
}
```

类似@ResponseBody,   并可以设置Http响应头部；

### @JsonView

```java
@RestController
public class JsonViewController {

    @GetMapping("/user")
    @JsonView(User.WithoutPasswordView.class)
    public User getUser() {
        return new User("eric", "7!jd#h23");
    }
}

class User {

    public interface WithoutPasswordView {};
    public interface WithPasswordView extends WithoutPasswordView {};

    private String username;
    private String password;

    public User() {
    }

    public User(String username, String password) {
        this.username = username;
        this.password = password;
    }

    @JsonView(WithoutPasswordView.class)
    public String getUsername() {
        return this.username;
    }

    @JsonView(WithPasswordView.class)
    public String getPassword() {
        return this.password;
    }
}
```

定义了两种显示模式；

 不显示密码类型WithoutPasswordView接口  ；

WithPasswordView显示接口类型 继承了WithoutPasswordView： 

**如果方法被@JsonView(User.WithoutPasswordView.class)修饰， 则User类中，被@WithoutPasswordView修饰的注解才能显示；**

**如果方法被@JsonView(User.WithPasswordView.class)， 则类属性中，被@WithPasswordView注解， 或者属性的注解被@WithPasswordView继承了，则这些属性可以显示；**

====> 用处不大， 还麻烦， 不如定义多个DTO来区分显示；

### @ModelAttribute - 数据绑定器

```java
@PostMapping("modelAttribute")
public Object modelAttr(@ModelAttribute Pet pet) {
    System.out.println(pet);
    return pet;
}
```

**会从Request请求中将Get/Post方式提交的参数通过WebDataBinder数据绑定器转换为目标参数实例**

以上代码相当于：

```java
@GetMapping("modelAttribute2")
public Object modelAttr2(Pet pet) {
    System.out.println(pet);
    return pet;
}
```



### @InitBInder 数据绑定器 ===> 时间格式化

- @InitBinder注解可以作用在被@Controller注解的类的方法上，表示为当前控制器注册一个属性编辑器，用于对WebDataBinder进行初始化，且只对当前的Controller有效。@InitBinder标注的方法会被多次执行的，也就是说来一次请求就会执行一次@InitBinder注解方法的内容。

  

  A. @InitBinder注解是在其所标注的方法执行之前被解析和执行；
  B. @InitBinder的value属性，控制的是模型Model里的KEY，而不是方法名；
  C. @InitBinder标注的方法也不能有返回值；
  D. @InitBinder对@RequestBody这种基于消息转换器的请求参数是失效的。 使用的是HttpMessageConverter直接转换字节数据为目标对象， 与数据绑定无关


- 客户端提交特定格式的时间格式数据，对日期Date类型参数进行绑定，就会报错IllegalStateException错误。所以需要注册一些类型绑定器用于对参数进行绑定。

```java
@RequestMapping(value = "format2", method ={ RequestMethod.POST, RequestMethod.GET })
public String format(Date data) {
    System.out.println(data);
    return "seccess";
}
```

![image-20221030145530066](assets/\image-20221030145530066.png)

对于这种情况，可以利用@InitBinder生成时间格式化器：

```java
@InitBinder
public void initBinder1(WebDataBinder binder) {
    binder.setDisallowedFields("name");
    //增加自定义事件格式化器， 提交的时间格式必须为yyyy-mm-dd
    binder.addCustomFormatter(new DateFormatter("yyyy-mm-dd"));
}
```

这样，如果前端提交的时间字段格式为“yyyy-mm-dd"就可以被接收到， 如果是其他格式，就会抛出400异常；





## 2、请求参数处理源码

拿到处理请求的处理器后，会去获取处理器适配器；

```java
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

				// Determine handler adapter for the current request.
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

				// Process last-modified header, if supported by the handler.
```
### 2.1 获取处理器适配器

获取处理器适配器：在容器加载过程中，放初始化处理器适配器；

```java
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
    if (this.handlerAdapters != null) {
        Iterator var2 = this.handlerAdapters.iterator();

        while(var2.hasNext()) {
            HandlerAdapter adapter = (HandlerAdapter)var2.next();
            if (adapter.supports(handler)) {
                return adapter;
            }
        }
    }

    throw new ServletException("No adapter for handler [" + handler + "]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
}
```

![handlerAdapter](assets/\image-20221012222830971.png)

- RequestMappingHandlerAdapter ： 用来处理注解方法方式的处理器URL请求，如@RequestMapping,@PostMapping等；
- HandlerFunctionAdapter:用来处理函数式方式的请求；
- HttpRequestHandlerAdpater:用来处理实现了了HttpRequestHandler方式的处理器；
- SimpleControllerHandlerAdapter:用来处理实现了Controller接口的处理器；

此处只关注注解方法模式的URL请求；

![image-20221012223818825](assets/\image-20221012223818825.png)

而对应的handler类型**为HandlerMethod**

```java
public abstract class AbstractHandlerMethodAdapter extends WebContentGenerator implements HandlerAdapter, Ordered {
    private int order = 2147483647;

    public AbstractHandlerMethodAdapter() {
        super(false);
    }


    public final boolean supports(Object handler) {
        return handler instanceof HandlerMethod && this.supportsInternal((HandlerMethod)handler);
    }
    
}
```

调用RequestmappingHandlerAdapter#supports判断该适配器是否支持该处理器； 由于处理器类型为HandlerMethod, 因此第一个判断成立，返回true， 因此返回的最终处理器适配器类型为RequestMappingHandlerAdapter;

### 2.2 HandlerAdapter#handle

```java
mappedHandler = this.getHandler(processedRequest);
if (mappedHandler == null) {
    this.noHandlerFound(processedRequest, response);
    return;
}

HandlerAdapter ha = this.getHandlerAdapter(mappedHandler.getHandler());
String method = request.getMethod();
boolean isGet = "GET".equals(method);
if (isGet || "HEAD".equals(method)) {
    long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
    if ((new ServletWebRequest(request, response)).checkNotModified(lastModified) && isGet) {
        return;
    }
}

if (!mappedHandler.applyPreHandle(processedRequest, response)) {
    return;
}

mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
if (asyncManager.isConcurrentHandlingStarted()) {
    return;
}
```

通过getHandlerAdapter处理器适配器后，最终调用其handle方法，而该方法最终会通过反射方式执行我们的注解方法；

```java
//------------------------RequestMappingHandlerAdapter-----------------
@Nullable
public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
    return this.handleInternal(request, response, (HandlerMethod)handler);
}

```

```java
//---------------RequestMappingHandlerAdapter------------------------
protected ModelAndView handleInternal(HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
    this.checkRequest(request);
    ModelAndView mav;
    if (this.synchronizeOnSession) {
        HttpSession session = request.getSession(false);
        if (session != null) {
            Object mutex = WebUtils.getSessionMutex(session);
            synchronized(mutex) {
                mav = this.invokeHandlerMethod(request, response, handlerMethod);
            }
        } else {
            mav = this.invokeHandlerMethod(request, response, handlerMethod);
        }
    } else {
        mav = this.invokeHandlerMethod(request, response, handlerMethod);
    }

	//....
    return mav;
}
```



这里才开始真正的执行请求。

```java
@Nullable
protected ModelAndView invokeHandlerMethod(HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
    ServletWebRequest webRequest = new ServletWebRequest(request, response);

    ModelAndView var15;
    try {
        WebDataBinderFactory binderFactory = this.getDataBinderFactory(handlerMethod);
        ModelFactory modelFactory = this.getModelFactory(handlerMethod, binderFactory);
        ServletInvocableHandlerMethod invocableMethod = this.createInvocableHandlerMethod(handlerMethod);
        if (this.argumentResolvers != null) {
            invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
        }

        if (this.returnValueHandlers != null) {
            invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
        }

        invocableMethod.setDataBinderFactory(binderFactory);
        invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);
        ModelAndViewContainer mavContainer = new ModelAndViewContainer();
        mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
        modelFactory.initModel(webRequest, mavContainer, invocableMethod);
        mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);
		//.....

        invocableMethod.invokeAndHandle(webRequest, mavContainer, new Object[0]);
        if (asyncManager.isConcurrentHandlingStarted()) {
            result = null;
            return (ModelAndView)result;
        }

        var15 = this.getModelAndView(mavContainer, modelFactory, webRequest);
    } finally {
        webRequest.requestCompleted();
    }

    return var15;
}
```

1. request,response包装为ServletWebRequest;
2. 将HandlerMethod处理器包装为ServletInvocableHandlerMethod类型；
3. 设置参数转换器argumentResolvers

4. 设置返回值处理器returnValueHandlers;
5. 执行和处理请求ServletInvocableHandlerMethod#invokeAndHandle；

6. 返回ModelAndView;

```java
//--------------------------ServletInvocableHandlerMethod--------------------
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
    Object returnValue = this.invokeForRequest(webRequest, mavContainer, providedArgs);
    this.setResponseStatus(webRequest);
    if (returnValue == null) {
        if (this.isRequestNotModified(webRequest) || this.getResponseStatus() != null || mavContainer.isRequestHandled()) {
            this.disableContentCachingIfNecessary(webRequest);
            mavContainer.setRequestHandled(true);
            return;
        }
    } else if (StringUtils.hasText(this.getResponseStatusReason())) {
        mavContainer.setRequestHandled(true);
        return;
    }

    mavContainer.setRequestHandled(false);
    Assert.state(this.returnValueHandlers != null, "No return value handlers");

    try {
        this.returnValueHandlers.handleReturnValue(returnValue, this.getReturnValueType(returnValue), mavContainer, webRequest);
    } catch (Exception var6) {
        if (this.logger.isTraceEnabled()) {
            this.logger.trace(this.formatErrorForReturnValue(returnValue), var6);
        }

        throw var6;
    }
}
```

1. 调用invokeForRequest处理器请求；
2. 调用返回值处理器处理返回值；

```java
//-------ServletInvocableHandlerMethod extends InvocableHandlerMethod--------------------
@Nullable
public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
    Object[] args = this.getMethodArgumentValues(request, mavContainer, providedArgs);
    if (this.logger.isTraceEnabled()) {
        this.logger.trace("Arguments: " + Arrays.toString(args));
    }

    return this.doInvoke(args);
}
```

1. 获取请求参数，通过30种参数转换器，确定方法所需要的参数值
2. 执行方法doInvoke

#### 2.2.1 getMethodArgumentValues获取请求参数值

```java
//------------------invocationHandlerMethod---------------------------
protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
        MethodParameter[] parameters = this.getMethodParameters();
        if (ObjectUtils.isEmpty(parameters)) {
            return EMPTY_ARGS;
        } else {
            //返回值
            Object[] args = new Object[parameters.length];

            //遍历
            for(int i = 0; i < parameters.length; ++i) {
                MethodParameter parameter = parameters[i];
                parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
                args[i] = findProvidedArgument(parameter, providedArgs);
                if (args[i] == null) {
                    if (!this.resolvers.supportsParameter(parameter)) {
                        throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
                    }

                    try {
                        args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
                    } catch (Exception var10) {
                        if (this.logger.isDebugEnabled()) {
                            String exMsg = var10.getMessage();
                            if (exMsg != null && !exMsg.contains(parameter.getExecutable().toGenericString())) {
                                this.logger.debug(formatArgumentError(parameter, exMsg));
                            }
                        }

                        throw var10;
                    }
                }
            }

            return args;
        }
    }
```
1. 获取方法的参数信息parameters；
2. 创建参数值数组Object[] args；
3. 遍历parameters
4. 调用findProvidedArgument获取参数值，由于providedArgs默认为空，因此返回的结果为空；
5. 调用所有的参数转换器获取参数resolvers#resolveArgument；

5. 返回；

##### 

```java
// HanderMethodArgumentResolverComposite#resoloveArgument
private final List<HandlerMethodArgumentResolver> argumentResolvers = new ArrayList();
private final Map<MethodParameter, HandlerMethodArgumentResolver> argumentResolverCache = new ConcurrentHashMap(256);

@Nullable
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
    HandlerMethodArgumentResolver resolver = this.getArgumentResolver(parameter);
    if (resolver == null) {
        throw new IllegalArgumentException("Unsupported parameter type [" + parameter.getParameterType().getName() + "]. supportsParameter should be called first.");
    } else {
        return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
    }
}
```

- 类属性包装了所有的参数转换器argumentResolvers
- **参数转换器缓存argumentResolverCache， key是方法参数MethodParameter, value为参数解析器**

1. 调用getArgumentResolver根据MethodParameter方法参数信息拿到适配的参数转换器
2. 调用参数转换器处理参数，拿到参数值；

```java
//-------------------HandlerMethodArgumentResolverComposite-------------------------
private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
    HandlerMethodArgumentResolver result = (HandlerMethodArgumentResolver)this.argumentResolverCache.get(parameter);
    if (result == null) {
        Iterator var3 = this.argumentResolvers.iterator();

        while(var3.hasNext()) {
            HandlerMethodArgumentResolver resolver = (HandlerMethodArgumentResolver)var3.next();
            if (resolver.supportsParameter(parameter)) {
                result = resolver;
                this.argumentResolverCache.put(parameter, resolver);
                break;
            }
        }
    }

    return result;
}
```

- 从参数处理器器缓存argumentResolverCache根据Parameter参数获取参数处理器；
- 缓存中存在，则返回；
- 不存在，则遍历所有的参数处理器
- 判断是否参数处理器是否可以处理该参数，如果可以，则以parameter为key, 参数解析器resolver为value,放入参数处理器缓存argumentResolverCache; 因此下次就不需要遍历，直接从cache中获取返回即可；

因此，如果参数越多的处理器，首次请求处理时间就越长，后续就会变快

#### 2.2.2 doInvoke执行处理器方法

```java
//-------ServletInvocableHandlerMethod extends InvocableHandlerMethod--------------------
@Nullable
protected Object doInvoke(Object... args) throws Exception {
    ReflectionUtils.makeAccessible(this.getBridgedMethod());

    try {
        return this.getBridgedMethod().invoke(this.getBean(), args);
    } catch (IllegalArgumentException var4) {
    //...
    } 
    }
}
```

**经典反射执行目标注解方法**；



## 3、参数转换器ArgumentResolver

参数转换器：注解方法中，会见到@RequestBody,@RequestParam, @PathVarible等注解参数， 发送的请求参数要么是URL参数，要么是Body参数， 如何将传过来的参数转换为方法所需要的目标参数，依赖的就是参数转换器ArgumentResolver;

```java
public interface HandlerMethodArgumentResolver {
    boolean supportsParameter(MethodParameter var1);

    @Nullable
    Object resolveArgument(MethodParameter var1, @Nullable ModelAndViewContainer var2, NativeWebRequest var3, @Nullable WebDataBinderFactory var4) throws Exception;
}
```

- supportsParameter: 判断参数处理器是否支持处理某个参数；
- resolveArgument：处理请求参数值；

常见的参数转换器如下：

![ArgumentResolver](assets/\image-20221012233708244.png)

### 1、 @RequestParam、 @RequestPart、@RequestHeader、 @RequestBody的处理

1. RequestParamMEthodArgumentResolver：处理@RequestParam注解参数， 参数类型非Map;

```java
// -------------- RequestParamMEthodArgumentResolver ---------------------------
public boolean supportsParameter(MethodParameter parameter) {
    if (parameter.hasParameterAnnotation(RequestParam.class)) {
        if (!Map.class.isAssignableFrom(parameter.nestedIfOptional().getNestedParameterType())) {
            return true;
        } else {
            RequestParam requestParam = (RequestParam)parameter.getParameterAnnotation(RequestParam.class);
            return requestParam != null && StringUtils.hasText(requestParam.name());
        }
    } else if (parameter.hasParameterAnnotation(RequestPart.class)) {
        return false;
    } else {
        parameter = parameter.nestedIfOptional();
        if (MultipartResolutionDelegate.isMultipartArgument(parameter)) {
            return true;
        } else {
            return this.useDefaultResolution ? BeanUtils.isSimpleProperty(parameter.getNestedParameterType()) : false;
        }
    }
}
```

- supportParameter:判断参数是否被@RequestParam注解修饰， 有则判断参数类型是否为Map, 如果不是，则返回true, 代表该参数可以被RequestParamMEthodArgumentResolver处理；

如果是，则判断注解属性name字段是否为空，为空，返回false,表示不支持此参数;不为空，返回true, 支持此参数(但是获取值会抛出异常MissingServletRequestParameterException) 



```java
//-------------------AbstractNamedValueMethodArgumentResolver--------------------------  
@Nullable
    public final Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
        AbstractNamedValueMethodArgumentResolver.NamedValueInfo namedValueInfo = this.getNamedValueInfo(parameter);
        MethodParameter nestedParameter = parameter.nestedIfOptional();
        Object resolvedName = this.resolveEmbeddedValuesAndExpressions(namedValueInfo.name);
        if (resolvedName == null) {
            throw new IllegalArgumentException("Specified name must not resolve to null: [" + namedValueInfo.name + "]");
        } else {
            Object arg = this.resolveName(resolvedName.toString(), nestedParameter, webRequest);
            if (arg == null) {
                if (namedValueInfo.defaultValue != null) {
                    arg = this.resolveEmbeddedValuesAndExpressions(namedValueInfo.defaultValue);
                } else if (namedValueInfo.required && !nestedParameter.isOptional()) {
                    this.handleMissingValue(namedValueInfo.name, nestedParameter, webRequest);
                }

                arg = this.handleNullValue(namedValueInfo.name, arg, nestedParameter.getNestedParameterType());
            } else if ("".equals(arg) && namedValueInfo.defaultValue != null) {
                arg = this.resolveEmbeddedValuesAndExpressions(namedValueInfo.defaultValue);
            }

           //....

            this.handleResolvedValue(arg, namedValueInfo.name, parameter, mavContainer, webRequest);
            return arg;
        }
    }
protected abstract Object resolveName(String var1, MethodParameter var2, NativeWebRequest var3) throws Exception;
```
- resolveArgument : RequestParamMEthodArgumentResolver没有实现该方法，继承了父类AbstractNamedValueMethodArgumentResolver的resolveArgument方法； 调用resolveName获取参数值， 该方法为抽象方法， 实现由子类实现；因此参数的值由RequestParamMEthodArgumentResolver#resolveName中获取；

```java
// --------------RequestParamMEthodArgumentResolver---------------------------
protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception {
    HttpServletRequest servletRequest = (HttpServletRequest)request.getNativeRequest(HttpServletRequest.class);
    Object arg;
  //...
    arg = null;
    MultipartRequest multipartRequest = (MultipartRequest)request.getNativeRequest(MultipartRequest.class);
    if (multipartRequest != null) {
        List<MultipartFile> files = multipartRequest.getFiles(name);
        if (!files.isEmpty()) {
            arg = files.size() == 1 ? files.get(0) : files;
        }
    }

    if (arg == null) {
        String[] paramValues = request.getParameterValues(name);
        if (paramValues != null) {
            arg = paramValues.length == 1 ? paramValues[0] : paramValues;
        }
    }

    return arg;
}
```

- MultipartRequest为上传文件时，会将HttpServletRequest包装为MultipartRequest,  通过MultipartRequest获取参数值；

- 如果非文件上传请求， 则调用request#getParameterValues获取请求参数值；



因此， RequestParamMethodArgumentResolver的参数值最终通过Request#getParameterValues获取到的。





2. RequestParamMethodMapArgumentResolver:处理@RequestParam注解参数， 参数类型为Map;

```java
public class RequestParamMapMethodArgumentResolver implements HandlerMethodArgumentResolver {
    public RequestParamMapMethodArgumentResolver() {
    }

    public boolean supportsParameter(MethodParameter parameter) {
        RequestParam requestParam = (RequestParam)parameter.getParameterAnnotation(RequestParam.class);
        return requestParam != null && Map.class.isAssignableFrom(parameter.getParameterType()) && !StringUtils.hasText(requestParam.name());
    }

    public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
        ResolvableType resolvableType = ResolvableType.forMethodParameter(parameter);
//....
        MultipartRequest multipartRequest;
        if (!MultiValueMap.class.isAssignableFrom(parameter.getParameterType())) {
            valueType = resolvableType.asMap().getGeneric(new int[]{1}).resolve();
            if (valueType == MultipartFile.class) {
                //处理文件请求
                multipartRequest = MultipartResolutionDelegate.resolveMultipartRequest(webRequest);
                return multipartRequest != null ? multipartRequest.getFileMap() : new LinkedHashMap(0);
            } else if (valueType == Part.class) {
 		//....
            } else {
                parameterMap = webRequest.getParameterMap();
                Map<String, String> result = new LinkedHashMap(parameterMap.size());
                parameterMap.forEach((key, values) -> {
                    if (values.length > 0) {
                        result.put(key, values[0]);
                    }

                });
                return result;
            }
        } else {
       //...
        }
    }
}
```

- supportParameter:参数被@RequestParam修饰，且参数类型为Map, 且@RuquestParam的name属性为空，则支持处理此参数；

- resolveArgument： 调用Request#getParameterMap方法获取到所有参数， 创建一个LinkHashMap实例result， 将所有参数遍历放进去；返回result值；



3. RequestHeaderMethodArgumentResolver:处理@RequestHander注解的路径参数；

```java
public class RequestHeaderMethodArgumentResolver extends AbstractNamedValueMethodArgumentResolver {
    public RequestHeaderMethodArgumentResolver(@Nullable ConfigurableBeanFactory beanFactory) {
        super(beanFactory);
    }

    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(RequestHeader.class) && !Map.class.isAssignableFrom(parameter.nestedIfOptional().getNestedParameterType());
    }

    protected NamedValueInfo createNamedValueInfo(MethodParameter parameter) {
        RequestHeader ann = (RequestHeader)parameter.getParameterAnnotation(RequestHeader.class);
        Assert.state(ann != null, "No RequestHeader annotation");
        return new RequestHeaderMethodArgumentResolver.RequestHeaderNamedValueInfo(ann);
    }

    @Nullable
    protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception {
        String[] headerValues = request.getHeaderValues(name);
        if (headerValues != null) {
            return headerValues.length == 1 ? headerValues[0] : headerValues;
        } else {
            return null;
        }
    }
 //....   
}
```

- supportsParameter： 方法参数被@RequestHeader注解修饰，且参数类型不是Map， 则RequestHeaderMethodArgumentResolver可以处理此参数；

- resolveArgument: 同RequestParamMethodArgumentResolver一样，真正获取参数值逻辑在resolveName中；
- resolveName: 获取Request#getHeaderValues(name)， 根据请求头参数名获取参数值；



4. RequestHeaderMethodMapArgumentResolver:处理@RequestHeader注解参数， 且参数类型为Map;

```java
public class RequestHeaderMapMethodArgumentResolver implements HandlerMethodArgumentResolver {
    public RequestHeaderMapMethodArgumentResolver() {
    }

    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(RequestHeader.class) && Map.class.isAssignableFrom(parameter.getParameterType());
    }

    public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
        Class<?> paramType = parameter.getParameterType();
        Iterator iterator;
        String headerName;
        if (!MultiValueMap.class.isAssignableFrom(paramType)) {
            Map<String, String> result = new LinkedHashMap();
            iterator = webRequest.getHeaderNames();

            while(iterator.hasNext()) {
                headerName = (String)iterator.next();
                String headerValue = webRequest.getHeader(headerName);
                if (headerValue != null) {
                    result.put(headerName, headerValue);
                }
            }

            return result;
        } else {
            Object result;
            if (HttpHeaders.class.isAssignableFrom(paramType)) {
                result = new HttpHeaders();
            } else {
                result = new LinkedMultiValueMap();
            }

            iterator = webRequest.getHeaderNames();

            while(true) {
                String[] headerValues;
                do {
                    if (!iterator.hasNext()) {
                        return result;
                    }

                    headerName = (String)iterator.next();
                    headerValues = webRequest.getHeaderValues(headerName);
                } while(headerValues == null);

                String[] var10 = headerValues;
                int var11 = headerValues.length;

                for(int var12 = 0; var12 < var11; ++var12) {
                    String headerValue = var10[var12];
                    ((MultiValueMap)result).add(headerName, headerValue);
                }
            }
        }
    }
}
```

- supportParameter: 参数被@RequestHeader修饰， 且参数类型为Map， 或者继承了Map，则RequestHeaderMethodMapArgumentResolver可处理此参数；

- resolveArgument: 判断参数是否为MultiValueMap ,或者继承了MultiValueMap；

  如果不是， 则创建LinkHashMap实例LinkedHashMap， 通过webRequest.getHeaderNames()获取请求头参数名，通过webRequest.getHeader(headerName)获取请求头参数值， 放入result中；

   如果是，则判断类型是否为HttpHeaders， 或者继承了HttpHeaders, 则创建HttpHeaders实例， 否则创建LinkMutliValueMap实例，调用webRequest.getHeaderNames()获取请求头参数名，webRequest.getHeaderValues(headerName)获取请求参数值放入headerValues数组， 再遍历headerValue数组， 将值放入result中；



5. RequestResponseBodyMethodProcessor: 处理@ResponseBody, @RequestBody的参数

```java
public class RequestResponseBodyMethodProcessor extends AbstractMessageConverterMethodProcessor {
	//....

    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(RequestBody.class);
    }

    public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
        parameter = parameter.nestedIfOptional();
        Object arg = this.readWithMessageConverters(webRequest, parameter, parameter.getNestedGenericParameterType());
        String name = Conventions.getVariableNameForParameter(parameter);
		//...

        return this.adaptArgumentIfNecessary(arg, parameter);
    }

    protected <T> Object readWithMessageConverters(NativeWebRequest webRequest, MethodParameter parameter, Type paramType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {
        HttpServletRequest servletRequest = (HttpServletRequest)webRequest.getNativeRequest(HttpServletRequest.class);
        Assert.state(servletRequest != null, "No HttpServletRequest");
        ServletServerHttpRequest inputMessage = new ServletServerHttpRequest(servletRequest);
        Object arg = this.readWithMessageConverters(inputMessage, parameter, paramType);
        if (arg == null && this.checkRequired(parameter)) {
            throw new HttpMessageNotReadableException("Required request body is missing: " + parameter.getExecutable().toGenericString(), inputMessage);
        } else {
            return arg;
        }
    }
    //....
}    
    
```

- supportParameter:判断参数是否被@RequestBody注解修饰， 是则可以处理此参数；

- resolveArgument: 调用readWithMessageConverters方法，获取参数值；

- readWithMessageConverters：

  - 获取HttpServletRequest实例servletRequest;

  - 将servletRequest实例包装为ServletServerHttpRequest实例inputMessage；

  - 调用readWithMessageConverters从请求体获取数据，并转换为参数类型的实例；

    

```java
//-----------------AbstractMessageConverterMethodArgumentResolver------------------
@Nullable
protected <T> Object readWithMessageConverters(HttpInputMessage inputMessage, MethodParameter parameter, Type targetType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {
    boolean noContentType = false;
	//请求内容类型
    MediaType contentType;
    try {
        contentType = inputMessage.getHeaders().getContentType();
    } catch (InvalidMediaTypeException var16) {
        throw new HttpMediaTypeNotSupportedException(var16.getMessage());
    }
	//如果没有获取到，则默认设置为application/octet-stream
    if (contentType == null) {
        noContentType = true;
        contentType = MediaType.APPLICATION_OCTET_STREAM;
    }

    Class<?> contextClass = parameter.getContainingClass();
    //拿到参数类型Class；
    Class<T> targetClass = targetType instanceof Class ? (Class)targetType : null;
    if (targetClass == null) {
        ResolvableType resolvableType = ResolvableType.forMethodParameter(parameter);
        targetClass = resolvableType.resolve();
    }
	
    //获取HttpMethod
    HttpMethod httpMethod = inputMessage instanceof HttpRequest ? ((HttpRequest)inputMessage).getMethod() : null;
    Object body = NO_VALUE;

    AbstractMessageConverterMethodArgumentResolver.EmptyBodyCheckingHttpInputMessage message;
    try {
        label94: {
            //将参数内容转换为EmptyBodyCheckingHttpInputMessage， 
            //该类包装了请求体数据，以及所有的请求头
            message = new AbstractMessageConverterMethodArgumentResolver.EmptyBodyCheckingHttpInputMessage(inputMessage);
            
           //拿到所有消息转换器HttpMessageConvertor
            Iterator var11 = this.messageConverters.iterator();

            HttpMessageConverter converter;
            Class converterType;
            GenericHttpMessageConverter genericConverter;
            
            //循环拿到可以合适的消息转换器
            while(true) {
                if (!var11.hasNext()) {
                    break label94;
                }

                converter = (HttpMessageConverter)var11.next();
                //消息转换器的类型
                converterType = converter.getClass();
                genericConverter = converter instanceof GenericHttpMessageConverter ? (GenericHttpMessageConverter)converter : null;
                //调用canRead方法判断是否可以支持请求的内容格式；
                if (genericConverter != null) {
                    if (genericConverter.canRead(targetType, contextClass, contentType)) {
                        break;
                    }
                } else if (targetClass != null && converter.canRead(targetClass, contentType)) {
                    break;
                }
            }
			
            //调用HttpMessageConvertor的read方法，读取数据，最终通过JSON反射生成实例参数值；返回；
            if (message.hasBody()) {
                HttpInputMessage msgToUse = this.getAdvice().beforeBodyRead(message, parameter, targetType, converterType);
                body = genericConverter != null ? genericConverter.read(targetType, contextClass, msgToUse) : converter.read(targetClass, msgToUse);
                body = this.getAdvice().afterBodyRead(body, msgToUse, parameter, targetType, converterType);
            } else {
                body = this.getAdvice().handleEmptyBody((Object)null, message, parameter, targetType, converterType);
            }
        }
    } catch (IOException var17) {
        throw new HttpMessageNotReadableException("I/O error while reading input message", var17, inputMessage);
    }

    if (body != NO_VALUE) {
        LogFormatUtils.traceDebug(this.logger, (traceOn) -> {
            String formatted = LogFormatUtils.formatValue(body, !traceOn);
            return "Read \"" + contentType + "\" to [" + formatted + "]";
        });
        return body;
    } else if (httpMethod != null && SUPPORTED_METHODS.contains(httpMethod) && (!noContentType || message.hasBody())) {
        throw new HttpMediaTypeNotSupportedException(contentType, this.allSupportedMediaTypes);
    } else {
        return null;
    }
}
```





6. RequestPartMethodArgumentResolver:处理@RequestPart注解的路径参数；

```java
public class RequestPartMethodArgumentResolver extends AbstractMessageConverterMethodArgumentResolver {
    public RequestPartMethodArgumentResolver(List<HttpMessageConverter<?>> messageConverters) {
        super(messageConverters);
    }

    public RequestPartMethodArgumentResolver(List<HttpMessageConverter<?>> messageConverters, List<Object> requestResponseBodyAdvice) {
        super(messageConverters, requestResponseBodyAdvice);
    }

    public boolean supportsParameter(MethodParameter parameter) {
        if (parameter.hasParameterAnnotation(RequestPart.class)) {
            return true;
        } else {
            return parameter.hasParameterAnnotation(RequestParam.class) ? false : MultipartResolutionDelegate.isMultipartArgument(parameter.nestedIfOptional());
        }
    }

    @Nullable
    public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest request, @Nullable WebDataBinderFactory binderFactory) throws Exception {
        HttpServletRequest servletRequest = (HttpServletRequest)request.getNativeRequest(HttpServletRequest.class);
        Assert.state(servletRequest != null, "No HttpServletRequest");
        RequestPart requestPart = (RequestPart)parameter.getParameterAnnotation(RequestPart.class);
        boolean isRequired = (requestPart == null || requestPart.required()) && !parameter.isOptional();
        String name = this.getPartName(parameter, requestPart);
        parameter = parameter.nestedIfOptional();
        Object arg = null;
        Object mpArg = MultipartResolutionDelegate.resolveMultipartArgument(name, parameter, servletRequest);
        if (mpArg != MultipartResolutionDelegate.UNRESOLVABLE) {
            arg = mpArg;
        } else {
            try {
                HttpInputMessage inputMessage = new RequestPartServletServerHttpRequest(servletRequest, name);
                arg = this.readWithMessageConverters(inputMessage, parameter, parameter.getNestedGenericParameterType());
                if (binderFactory != null) {
                    WebDataBinder binder = binderFactory.createBinder(request, arg, name);
                    if (arg != null) {
                        this.validateIfApplicable(binder, parameter);
                        if (binder.getBindingResult().hasErrors() && this.isBindExceptionRequired(binder, parameter)) {
                            throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
                        }
                    }

                    if (mavContainer != null) {
                        mavContainer.addAttribute(BindingResult.MODEL_KEY_PREFIX + name, binder.getBindingResult());
                    }
                }
            } catch (MultipartException | MissingServletRequestPartException var13) {
                if (isRequired) {
                    throw var13;
                }
            }
        }

        if (arg == null && isRequired) {
            if (!MultipartResolutionDelegate.isMultipartRequest(servletRequest)) {
                throw new MultipartException("Current request is not a multipart request");
            } else {
                throw new MissingServletRequestPartException(name);
            }
        } else {
            return this.adaptArgumentIfNecessary(arg, parameter);
        }
    }

    private String getPartName(MethodParameter methodParam, @Nullable RequestPart requestPart) {
        String partName = requestPart != null ? requestPart.name() : "";
        if (partName.isEmpty()) {
            partName = methodParam.getParameterName();
            if (partName == null) {
                throw new IllegalArgumentException("Request part name for argument type [" + methodParam.getNestedParameterType().getName() + "] not specified, and parameter name information not found in class file either.");
            }
        }

        return partName;
    }
}

    @Nullable
    public static Object resolveMultipartArgument(String name, MethodParameter parameter, HttpServletRequest request) throws Exception {
        MultipartHttpServletRequest multipartRequest = (MultipartHttpServletRequest)WebUtils.getNativeRequest(request, MultipartHttpServletRequest.class);
        boolean isMultipart = multipartRequest != null || isMultipartContent(request);
        if (MultipartFile.class == parameter.getNestedParameterType()) {
            if (multipartRequest == null && isMultipart) {
                multipartRequest = new StandardMultipartHttpServletRequest(request);
            }

            return multipartRequest != null ? ((MultipartHttpServletRequest)multipartRequest).getFile(name) : null;
        } else if (isMultipartFileCollection(parameter)) {
            if (multipartRequest == null && isMultipart) {
                multipartRequest = new StandardMultipartHttpServletRequest(request);
            }

            return multipartRequest != null ? ((MultipartHttpServletRequest)multipartRequest).getFiles(name) : null;
        } else if (isMultipartFileArray(parameter)) {
            if (multipartRequest == null && isMultipart) {
                multipartRequest = new StandardMultipartHttpServletRequest(request);
            }

            if (multipartRequest != null) {
                List<MultipartFile> multipartFiles = ((MultipartHttpServletRequest)multipartRequest).getFiles(name);
                return multipartFiles.toArray(new MultipartFile[0]);
            } else {
                return null;
            }
        } else if (Part.class == parameter.getNestedParameterType()) {
            return isMultipart ? request.getPart(name) : null;
        } else if (isPartCollection(parameter)) {
            return isMultipart ? resolvePartList(request, name) : null;
        } else if (isPartArray(parameter)) {
            return isMultipart ? resolvePartList(request, name).toArray(new Part[0]) : null;
        } else {
            return UNRESOLVABLE;
        }
    }

```

- supportParameter:判断方法参数是否被@RequestPart注解修饰，是则可以处理此参数；

- resolveArgument: 
  - 获取RequestPart的参数名称name
  - 调用resolveMultipartArgument， 如果参数类型为MultipartFile， MultipartFile集合，MultipartFile数组，则获取调用getFiles获取文件； 如果为Part,或者Part集合，Part数组， 则调用getPart获取对应的参数返回；如果都不是， 则返回UNRESOLVABLE ,表示未找到；
  - 判断是否找到参数值， 如果不为UNRESOLVABLE ， 表示找到，返回；
  - 如果为UNRESOLVABLE ， 则未找到参数值，调用readWithMessageConverters方法，从请求体获取参数值；

### 2、 Servlet API参数处理

7. ServletRequestMethodArgumentResolver : 处理HttpServletReqeust参数

```java
public class ServletRequestMethodArgumentResolver implements HandlerMethodArgumentResolver {

    public boolean supportsParameter(MethodParameter parameter) {
        Class<?> paramType = parameter.getParameterType();
        return WebRequest.class.isAssignableFrom(paramType) || ServletRequest.class.isAssignableFrom(paramType) || MultipartRequest.class.isAssignableFrom(paramType) || HttpSession.class.isAssignableFrom(paramType) || pushBuilder != null && pushBuilder.isAssignableFrom(paramType) || Principal.class.isAssignableFrom(paramType) || InputStream.class.isAssignableFrom(paramType) || Reader.class.isAssignableFrom(paramType) || HttpMethod.class == paramType || Locale.class == paramType || TimeZone.class == paramType || ZoneId.class == paramType;
    }

    public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
        Class<?> paramType = parameter.getParameterType();
        if (WebRequest.class.isAssignableFrom(paramType)) {
            if (!paramType.isInstance(webRequest)) {
                throw new IllegalStateException("Current request is not of type [" + paramType.getName() + "]: " + webRequest);
            } else {
                return webRequest;
            }
        } else {
            return !ServletRequest.class.isAssignableFrom(paramType) && !MultipartRequest.class.isAssignableFrom(paramType) ? this.resolveArgument(paramType, (HttpServletRequest)this.resolveNativeRequest(webRequest, HttpServletRequest.class)) : this.resolveNativeRequest(webRequest, paramType);
        }
    }

    private <T> T resolveNativeRequest(NativeWebRequest webRequest, Class<T> requiredType) {
        T nativeRequest = webRequest.getNativeRequest(requiredType);
        if (nativeRequest == null) {
            throw new IllegalStateException("Current request is not of type [" + requiredType.getName() + "]: " + webRequest);
        } else {
            return nativeRequest;
        }
    }

    @Nullable
    private Object resolveArgument(Class<?> paramType, HttpServletRequest request) throws IOException {
        if (HttpSession.class.isAssignableFrom(paramType)) {
            HttpSession session = request.getSession();
            if (session != null && !paramType.isInstance(session)) {
                throw new IllegalStateException("Current session is not of type [" + paramType.getName() + "]: " + session);
            } else {
                return session;
            }
        } else if (pushBuilder != null && pushBuilder.isAssignableFrom(paramType)) {
            return ServletRequestMethodArgumentResolver.PushBuilderDelegate.resolvePushBuilder(request, paramType);
        } else if (InputStream.class.isAssignableFrom(paramType)) {
            InputStream inputStream = request.getInputStream();
            if (inputStream != null && !paramType.isInstance(inputStream)) {
                throw new IllegalStateException("Request input stream is not of type [" + paramType.getName() + "]: " + inputStream);
            } else {
                return inputStream;
            }
        } else if (Reader.class.isAssignableFrom(paramType)) {
            Reader reader = request.getReader();
            if (reader != null && !paramType.isInstance(reader)) {
                throw new IllegalStateException("Request body reader is not of type [" + paramType.getName() + "]: " + reader);
            } else {
                return reader;
            }
        } else if (Principal.class.isAssignableFrom(paramType)) {
            Principal userPrincipal = request.getUserPrincipal();
            if (userPrincipal != null && !paramType.isInstance(userPrincipal)) {
                throw new IllegalStateException("Current user principal is not of type [" + paramType.getName() + "]: " + userPrincipal);
            } else {
                return userPrincipal;
            }
        } else if (HttpMethod.class == paramType) {
            return HttpMethod.resolve(request.getMethod());
        } else if (Locale.class == paramType) {
            return RequestContextUtils.getLocale(request);
        } else {
            TimeZone timeZone;
            if (TimeZone.class == paramType) {
                timeZone = RequestContextUtils.getTimeZone(request);
                return timeZone != null ? timeZone : TimeZone.getDefault();
            } else if (ZoneId.class == paramType) {
                timeZone = RequestContextUtils.getTimeZone(request);
                return timeZone != null ? timeZone.toZoneId() : ZoneId.systemDefault();
            } else {
                throw new UnsupportedOperationException("Unknown parameter type: " + paramType.getName());
            }
        }
    }
}
```

- supportParameter: 判断参数类型是否为WebRequest, ServletRequest, MultipartRequest, HttpSession, InputStream, Reader, HttpMethod, Locale以及其他几种类型或者继承这些类， 如果是，则可以处理此参数；

- resolveArgument: 

  - 参数为WebRequest或者继承了WebRequest, 则将传进来的参数webRequest就是参数值，返回；
  - 参数不是ServletRequest或者MultipartRequest或者继承了这两个类型，则调用resolveArgument方法处理其他参数；
  - 参数类型为HttpSession， 则调用request.getSession获取参数值
  - 参数类型为InputStream， 则调用request.getInputStream获取参数值；
  - 参数类型为Reader， 则调用request.getReader获取参数值；
  - 参数类型为HttpMethod， 则调用request.getMethod获取参数值；
  - 参数类型为Locale， 则调用RequestContextUtils.getLocale(request)获取国际化语言的值；

  - 处理一些不常见的参数，如ZoneId，TimeZone，pushBuilder

  - 获取到参数值后，返回；





8. ServletResponseMethodArgumentResolver: 获取HttpsServletResponse响应参数；

```java
public class ServletResponseMethodArgumentResolver implements HandlerMethodArgumentResolver {

    public boolean supportsParameter(MethodParameter parameter) {
        Class<?> paramType = parameter.getParameterType();
        return ServletResponse.class.isAssignableFrom(paramType) || OutputStream.class.isAssignableFrom(paramType) || Writer.class.isAssignableFrom(paramType);
    }

    public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
        if (mavContainer != null) {
            mavContainer.setRequestHandled(true);
        }

        Class<?> paramType = parameter.getParameterType();
        return ServletResponse.class.isAssignableFrom(paramType) ? this.resolveNativeResponse(webRequest, paramType) : this.resolveArgument(paramType, (ServletResponse)this.resolveNativeResponse(webRequest, ServletResponse.class));
    }

    private <T> T resolveNativeResponse(NativeWebRequest webRequest, Class<T> requiredType) {
        T nativeResponse = webRequest.getNativeResponse(requiredType);
        if (nativeResponse == null) {
            throw new IllegalStateException("Current response is not of type [" + requiredType.getName() + "]: " + webRequest);
        } else {
            return nativeResponse;
        }
    }

    private Object resolveArgument(Class<?> paramType, ServletResponse response) throws IOException {
        if (OutputStream.class.isAssignableFrom(paramType)) {
            return response.getOutputStream();
        } else if (Writer.class.isAssignableFrom(paramType)) {
            return response.getWriter();
        } else {
            throw new UnsupportedOperationException("Unknown parameter type: " + paramType);
        }
    }
}
```

- supportParameter:判断参数类型是否ServletResponse,OutputStream, Writer或者继承了这些类，则可以处理这些参数；
- resolveArgument: 获取参数类型paramType，判断参数类型是否为ServletResponse或者继承它， 则获取paramType类型的参数值返回（如果参数类型为HttpServletResponse），否则调用 resolveArgument，判断参数类型是否为OutputStream, 则调用response.getOutputStream, 如果为Writer, 则调用response.getWriter的参数值返回；

### 3、@CookieValue、 @SessionAttribute参数处理

9. ServletCookieValueMethodArgumentResolver ： 处理@CookieValue标记的参数；

   ```java
   public class ServletCookieValueMethodArgumentResolver extends AbstractCookieValueMethodArgumentResolver {
       private UrlPathHelper urlPathHelper;
   	//....
       @Nullable
       protected Object resolveName(String cookieName, MethodParameter parameter, NativeWebRequest webRequest) throws Exception {
           HttpServletRequest servletRequest = (HttpServletRequest)webRequest.getNativeRequest(HttpServletRequest.class);
           Assert.state(servletRequest != null, "No HttpServletRequest");
           Cookie cookieValue = WebUtils.getCookie(servletRequest, cookieName);
           if (Cookie.class.isAssignableFrom(parameter.getNestedParameterType())) {
               return cookieValue;
           } else {
               return cookieValue != null ? this.urlPathHelper.decodeRequestString(servletRequest, cookieValue.getValue()) : null;
           }
       }
   }
   ```

   ```java
   public abstract class AbstractCookieValueMethodArgumentResolver extends AbstractNamedValueMethodArgumentResolver {
       public boolean supportsParameter(MethodParameter parameter) {
           return parameter.hasParameterAnnotation(CookieValue.class);
       }
   }
   ```

```java
//-------------------WebUtils---------------------------
@Nullable
public static Cookie getCookie(HttpServletRequest request, String name) {
    Assert.notNull(request, "Request must not be null");
    Cookie[] cookies = request.getCookies();
    if (cookies != null) {
        Cookie[] var3 = cookies;
        int var4 = cookies.length;

        for(int var5 = 0; var5 < var4; ++var5) {
            Cookie cookie = var3[var5];
            if (name.equals(cookie.getName())) {
                return cookie;
            }
        }
    }

    return null;
}
```

- supportParameter: 判断参数是否被@CookieValue注解修饰，是则表示可以处理此参数；

- resolveArgument: 获取HttpServletRequest， 调用WebUtils.getCookie(request, cookieName)从Cookie中获取@CookieValue参数名对应的值；



10. SessionAttributeMethodArgumentResolver: 处理@SessionAttribute注解参数；

```java
public class SessionAttributeMethodArgumentResolver extends AbstractNamedValueMethodArgumentResolver {
    public SessionAttributeMethodArgumentResolver() {
    }

    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(SessionAttribute.class);
    }
//....

    @Nullable
    protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) {
        return request.getAttribute(name, 1);
    }
//...
}
```

```java
public Object getAttribute(String name, int scope) {
    if (scope == 0) {
        if (!this.isRequestActive()) {
            throw new IllegalStateException("Cannot ask for request attribute - request is not active anymore!");
        } else {
            return this.request.getAttribute(name);
        }
    } else {
        HttpSession session = this.getSession(false);
        if (session != null) {
            try {
                Object value = session.getAttribute(name);
                if (value != null) {
                    this.sessionAttributesToUpdate.put(name, value);
                }

                return value;
            } catch (IllegalStateException var5) {
                ;
            }
        }

        return null;
    }
}
```

- supportParameter:判断方法参数是否被@SessionAttribute注解修饰，是则可以处理该参数；

- resolveName：调用NativeWebRequest（继承了RequestAttribute）#getAttribute从会话中获取参数值返回；

## 4、请求处理-【源码分析】-Model、Map原理

```    java
@GetMapping("/params")
public String testParam(Map<String,Object> map,
                        Model model,
                        HttpServletRequest request,
                        HttpServletResponse response){
    //下面三位都是可以给request域中放数据
    map.put("hello","world666");
    model.addAttribute("world","hello666");
    request.setAttribute("message","HelloWorld");

    Cookie cookie = new Cookie("c1","v1");
    response.addCookie(cookie);
    return "forward:/success";
}

@ResponseBody
@GetMapping("/success")
public Map success(@RequestAttribute(value = "msg",required = false) String msg,
                   @RequestAttribute(value = "code",required = false)Integer code,
                   HttpServletRequest request){
    Object msg1 = request.getAttribute("msg");

    Map<String,Object> map = new HashMap<>();
    Object hello = request.getAttribute("hello");//得出testParam方法赋予的值 world666
    Object world = request.getAttribute("world");//得出testParam方法赋予的值 hello666
    Object message = request.getAttribute("message");//得出testParam方法赋予的值 HelloWorld

    map.put("reqMethod_msg",msg1);
    map.put("annotation_msg",msg);
    map.put("hello",hello);
    map.put("world",world);
    map.put("message",message);

    return map;
}

```

- 如果参数为Model，对应的参数处理器为ModelMethodProcessor;

- 如果参数为Map, 对应的参数处理器为MapMethodProcessor;

- 且重定向后，Model和Map的数据可以从请求request.getAttribute()中获取到；

11.  **ModelMethodProcessor**

```java
public class ModelMethodProcessor implements HandlerMethodArgumentResolver, HandlerMethodReturnValueHandler {
    public ModelMethodProcessor() {
    }

    public boolean supportsParameter(MethodParameter parameter) {
        return Model.class.isAssignableFrom(parameter.getParameterType());
    }

    @Nullable
    public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
        Assert.state(mavContainer != null, "ModelAndViewContainer is required for model exposure");
        return mavContainer.getModel();
    }

//....
}
```

- supportParameter ： 参数如果为Model, 或者继承了Model类型， 则可以处理此参数；
- resolveArgument:  从mavContaioner中获取model返回；

```java
public class ModelAndViewContainer {
    private boolean ignoreDefaultModelOnRedirect = false;
    @Nullable
    private Object view;
    private final ModelMap defaultModel = new BindingAwareModelMap();
    private boolean redirectModelScenario = false;

    public ModelMap getModel() {
        if (this.useDefaultModel()) {
            return this.defaultModel;
        } else {
            if (this.redirectModel == null) {
                this.redirectModel = new ModelMap();
            }

            return this.redirectModel;
        }
    }
    private boolean useDefaultModel() {
        return !this.redirectModelScenario || this.redirectModel == null && !this.ignoreDefaultModelOnRedirect;
    }
    //...
}    
```

useDefaultModel方法中，判断是否使用默认的model, redirectModelScenario默认为false, 因此返回true, 因此返回的defaultModel,  该model类型为BindingAwareModelMap

![BindingAwareModelMap](assets/\image-20221023135709679.png)

**BindingAwareModelMap与ModelMap都继承了HashMap，因此本质和HashMap一样；**

ModelMethodProcessor该处理器处理Model参数，最终就是返回Map实例；

12. **MapMethodProcessor** : 处理Map参数

```java
public class MapMethodProcessor implements HandlerMethodArgumentResolver, HandlerMethodReturnValueHandler {
    public MapMethodProcessor() {
    }

    public boolean supportsParameter(MethodParameter parameter) {
        return Map.class.isAssignableFrom(parameter.getParameterType()) && parameter.getParameterAnnotations().length == 0;
    }

    @Nullable
    public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
        Assert.state(mavContainer != null, "ModelAndViewContainer is required for model exposure");
        return mavContainer.getModel();
    }
}
```

- supportParameter  : 参数如果为Map或者继承了Map,  且该参数没有被其他参数注解修饰，则可以处理此参数；
- resolveArgument:  从mavContainer中获取model返回； 与ModelMethodProcessor同理，返回的是defaultModel实例；

MapMethodProcessor处理器处理Map参数， 最终返回的是mavContainer中的defaultModel实例；

#### ModelAndView

SpringMVC执行完处理器后，会返回ModelAndView实例， 将返回的数据Model 和视图View包装起来；

model数据放在了mavContainer中，View视图名也存放在了mavContainer中；

```java
//----------------------RequestMappingHandlerAdapter#invokeHandlerMethod
@Nullable
protected ModelAndView invokeHandlerMethod(HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
    ServletWebRequest webRequest = new ServletWebRequest(request, response);
ModelAndView var15;
try {
    ServletInvocableHandlerMethod invocableMethod = this.createInvocableHandlerMethod(handlerMethod);
    if (this.argumentResolvers != null) {
        invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
    }

    if (this.returnValueHandlers != null) {
        invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
    }

    ModelAndViewContainer mavContainer = new ModelAndViewContainer();
    mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
    modelFactory.initModel(webRequest, mavContainer, invocableMethod);
    mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

    //...
    Object result;

    invocableMethod.invokeAndHandle(webRequest, mavContainer, new Object[0]);
	
    var15 = this.getModelAndView(mavContainer, modelFactory, webRequest);
} finally {
    webRequest.requestCompleted();
}

return var15;}
```
```java
@Nullable
private ModelAndView getModelAndView(ModelAndViewContainer mavContainer, ModelFactory modelFactory, NativeWebRequest webRequest) throws Exception {
    modelFactory.updateModel(webRequest, mavContainer);
    if (mavContainer.isRequestHandled()) {
        return null;
    } else {
        ModelMap model = mavContainer.getModel();
        ModelAndView mav = new ModelAndView(mavContainer.getViewName(), model, mavContainer.getStatus());
        if (!mavContainer.isViewReference()) {
            mav.setView((View)mavContainer.getView());
        }
		//....
        return mav;
    }
}
```

该方法创建了ModelAndView实例，从mavContainer中获取数据Model 和视图名， 返回；



#### 重定向后，Model和Map的数据可以从请求request.getAttribute()中获取到的原因

ModelMethodProcessor和MapMethodProcessor处理器，处理完后，都会将数据放入defaultModel实例中；

**而HanderAdapter#handle处理结束后，返回ModelAndView实例；**

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    try {
        try {
            ModelAndView mv = null;

            try {
                mappedHandler = this.getHandler(processedRequest);

                HandlerAdapter ha = this.getHandlerAdapter(mappedHandler.getHandler());


                mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

                this.applyDefaultViewName(processedRequest, mv);
                mappedHandler.applyPostHandle(processedRequest, response, mv);
            } catch (Exception var20) {
                //...
            }

            this.processDispatchResult(processedRequest, response, mappedHandler, mv, (Exception)dispatchException);
        } catch (Exception var22) {
            //....
        }
    } finally {
        //....
    }
}

	private void processDispatchResult(HttpServletRequest request, HttpServletResponse response, @Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv, @Nullable Exception exception) throws Exception {
        boolean errorView = false;
		//...

        if (mv != null && !mv.wasCleared()) {
            this.render(mv, request, response);
        } 
        //...
    }

	//View的类型为InternalResourceView
    protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
        //...
        String viewName = mv.getViewName();
        View view;
        if (viewName != null) {
            view = this.resolveViewName(viewName, mv.getModelInternal(), locale, request);
		//...
        } 
        //...
        }
		//...
        try {
		//...
            view.render(mv.getModelInternal(), request, response);
        } catch (Exception var8) {
            //...
        }
    }

```

![InternalResouceView继承关系](assets/\image-20221023144133280.png)

```java
//--------------------InternalResouceView#render-------------------------------
public void render(@Nullable Map<String, ?> model, HttpServletRequest request, HttpServletResponse response) throws Exception {
	//model的数据合并起来
    Map<String, Object> mergedModel = this.createMergedOutputModel(model, request, response);
    this.prepareResponse(request, response);
    
    //渲染合并后的数据
    this.renderMergedOutputModel(mergedModel, this.getRequestToExpose(request), response);
}
//由InternalResouceView实现
protected abstract void renderMergedOutputModel(Map<String, Object> var1, HttpServletRequest var2, HttpServletResponse var3) throws Exception;

protected void renderMergedOutputModel(Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception {
        //暴露Model数据作为Request的属性；
        this.exposeModelAsRequestAttributes(model, request);
        if (rd == null) {
        //...
        } else {
            //转发或者重定向
            if (this.useInclude(request, response)) {
                response.setContentType(this.getContentType());

                rd.include(request, response);
            } else {

                rd.forward(request, response);
            }

        }
}
//遍历Map, 将所有数据放入request的attributes中；
protected void exposeModelAsRequestAttributes(Map<String, Object> model, HttpServletRequest request) throws Exception {
        model.forEach((name, value) -> {
            if (value != null) {
                request.setAttribute(name, value);
            } else {
                request.removeAttribute(name);
            }

        });
}
```

最终会将Model， Map参数中的数据作为请求Request的属性， 重定向或者转发到下一个请求中；



### 5、请求处理-【源码分析】-自定义参数绑定原理

```java
@RestController
public class BindTest {
    /**
     * 数据绑定：页面提交的请求数据（GET、POST）都可以和对象属性进行绑定
     * @param person
     * @return
     */
    @PostMapping("/saveuser")
    public Person saveuser(Person person){
        return person;
    }
}
```

- 注意：必须使用@RestController, 而不是@Controller；

- 此类参数对应的参数处理器为**ServletModelAttributeMethodProcessor；**

```java
public class ModelAttributeMethodProcessor implements HandlerMethodArgumentResolver, HandlerMethodReturnValueHandler {
    private static final ParameterNameDiscoverer parameterNameDiscoverer = new DefaultParameterNameDiscoverer();
    protected final Log logger = LogFactory.getLog(this.getClass());
    private final boolean annotationNotRequired;

    public ModelAttributeMethodProcessor(boolean annotationNotRequired) {
        this.annotationNotRequired = annotationNotRequired;
    }
	//isSimpleProperty : 由于Person不是普通的参数类型，因此返回false; 
    //annotationNotRequired: 属性为true;
    //因此返回了true;
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(ModelAttribute.class) || this.annotationNotRequired && !BeanUtils.isSimpleProperty(parameter.getParameterType());
    }

    @Nullable
    public final Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
		//...
        String name = ModelFactory.getNameForParameter(parameter);
        ModelAttribute ann = (ModelAttribute)parameter.getParameterAnnotation(ModelAttribute.class);
        
        //不存在注解，为null
        if (ann != null) {
            mavContainer.setBinding(name, ann.binding());
        }

        Object attribute = null;
        BindingResult bindingResult = null;
        if (mavContainer.containsAttribute(name)) {
            attribute = mavContainer.getModel().get(name);
        } else {
            try {
                attribute = this.createAttribute(name, parameter, binderFactory, webRequest);
            } catch (BindException var10) {
				//....
            }
        }

        if (bindingResult == null) {
            WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name);
            if (binder.getTarget() != null) {
                if (!mavContainer.isBindingDisabled(name)) {
                    this.bindRequestParameters(binder, webRequest);
                }

                this.validateIfApplicable(binder, parameter);
                //...
            }

            if (!parameter.getParameterType().isInstance(attribute)) {
                attribute = binder.convertIfNecessary(binder.getTarget(), parameter.getParameterType(), parameter);
            }

            bindingResult = binder.getBindingResult();
        }

        Map<String, Object> bindingResultModel = bindingResult.getModel();
        mavContainer.removeAttributes(bindingResultModel);
        mavContainer.addAllAttributes(bindingResultModel);
        return attribute;
    }   
}
```

- createAttribute  :  调用父类createAttribute, 获取默认的构造器， 通过利用构造器通过反射创建实例；
- createBinder : 创建数据绑定器工厂创建web数据绑定器，类型为ExtendedServletRequestDataBinder
- bindRequestParameters: 

```java
//
protected final Object createAttribute(String attributeName, MethodParameter parameter, WebDataBinderFactory binderFactory, NativeWebRequest request) throws Exception {
    String value = this.getRequestValueForAttribute(attributeName, request);
	//...

    return super.createAttribute(attributeName, parameter, binderFactory, request);
}
```

```java
protected Object createAttribute(String attributeName, MethodParameter parameter, WebDataBinderFactory binderFactory, NativeWebRequest webRequest) throws Exception {
    MethodParameter nestedParameter = parameter.nestedIfOptional();
    Class<?> clazz = nestedParameter.getNestedParameterType();
    Constructor<?> ctor = BeanUtils.findPrimaryConstructor(clazz);
    if (ctor == null) {
        Constructor<?>[] ctors = clazz.getConstructors();
        if (ctors.length == 1) {
            ctor = ctors[0];
        }

    Object attribute = this.constructAttribute(ctor, attributeName, parameter, binderFactory, webRequest);
    return attribute;
}
```



```java
public class DefaultDataBinderFactory implements WebDataBinderFactory {
    @Nullable
    private final WebBindingInitializer initializer;

    public final WebDataBinder createBinder(NativeWebRequest webRequest, @Nullable Object target, String objectName) throws Exception {
        WebDataBinder dataBinder = this.createBinderInstance(target, objectName, webRequest);
		//....
        this.initBinder(dataBinder, webRequest);
        return dataBinder;
    }

    protected ServletRequestDataBinder createBinderInstance(@Nullable Object target, String objectName, NativeWebRequest request) throws Exception {
        return new ExtendedServletRequestDataBinder(target, objectName);
    }
	//...
}
```

![ExtendedServletRequestDataBinder](assets/\image-20221023211919606.png)

```java
//------------------
protected void bindRequestParameters(WebDataBinder binder, NativeWebRequest request) {
    ServletRequest servletRequest = (ServletRequest)request.getNativeRequest(ServletRequest.class);
    Assert.state(servletRequest != null, "No ServletRequest");
    ServletRequestDataBinder servletBinder = (ServletRequestDataBinder)binder;
    servletBinder.bind(servletRequest);
}
```

```java
//----------------------ServletRequestDataBinder#bind---------------------------------
public void bind(ServletRequest request) {
    MutablePropertyValues mpvs = new ServletRequestParameterPropertyValues(request);
    MultipartRequest multipartRequest = (MultipartRequest)WebUtils.getNativeRequest(request, MultipartRequest.class);
    if (multipartRequest != null) {
        this.bindMultipart(multipartRequest.getMultiFileMap(), mpvs);
    }

    this.addBindValues(mpvs, request);
    this.doBind(mpvs);
}
```

封装了MutablePropertyValues， 里面包装了一个PropertyValue类型的集合，每个PropertyValue的key为参数名称，value为参数值；因此MutablePropertyValues包含了所有的参数key-value；

```java
//------------------------------WebBinder#doBind-----------------------------------
protected void doBind(MutablePropertyValues mpvs) {
    this.checkFieldDefaults(mpvs);
    this.checkFieldMarkers(mpvs);
    super.doBind(mpvs);
}
```

```java
//-----------------------------DataBinder#doBind--------------------------------
protected void doBind(MutablePropertyValues mpvs) {
    this.checkAllowedFields(mpvs);
    this.checkRequiredFields(mpvs);
    this.applyPropertyValues(mpvs);
}
```

```java
//-------------------------------DataBinder#applyPropertyValues---------------------
protected void applyPropertyValues(MutablePropertyValues mpvs) {
    try {
        this.getPropertyAccessor().setPropertyValues(mpvs, this.isIgnoreUnknownFields(), this.isIgnoreInvalidFields());
    } catch (PropertyBatchUpdateException var7) {
        //。...
    }
}
```



```java
//
public void setPropertyValues(PropertyValues pvs, boolean ignoreUnknown, boolean ignoreInvalid) throws BeansException {
    List<PropertyAccessException> propertyAccessExceptions = null;
    List<PropertyValue> propertyValues = pvs instanceof MutablePropertyValues ? ((MutablePropertyValues)pvs).getPropertyValueList() : Arrays.asList(pvs.getPropertyValues());
    try {
        Iterator var6 = propertyValues.iterator();
        while(var6.hasNext()) {
            PropertyValue pv = (PropertyValue)var6.next();

            try {
                this.setPropertyValue(pv);
            } 
        }
    } 
}
```

```java
public void setPropertyValue(PropertyValue pv) throws BeansException {
    AbstractNestablePropertyAccessor.PropertyTokenHolder tokens = (AbstractNestablePropertyAccessor.PropertyTokenHolder)pv.resolvedTokens;
    if (tokens == null) {
        String propertyName = pv.getName();

        AbstractNestablePropertyAccessor nestedPa;
        try {
            nestedPa = this.getPropertyAccessorForPropertyPath(propertyName);
        } 
        //...
        tokens = this.getPropertyNameTokens(this.getFinalPath(nestedPa, propertyName));
        nestedPa.setPropertyValue(tokens, pv);
    } 
}
```

```java

protected void setPropertyValue(AbstractNestablePropertyAccessor.PropertyTokenHolder tokens, PropertyValue pv) throws BeansException {
    if (tokens.keys != null) {
        //....
    } else {
        this.processLocalProperty(tokens, pv);
    }

}
```

```java
private void processLocalProperty(AbstractNestablePropertyAccessor.PropertyTokenHolder tokens, PropertyValue pv) {
    AbstractNestablePropertyAccessor.PropertyHandler ph = this.getLocalPropertyHandler(tokens.actualName);
    if (ph != null && ph.isWritable()) {
        Object oldValue = null;

        PropertyChangeEvent propertyChangeEvent;
        try {
            //拿出参数原始的值；即String类型的值；
            Object originalValue = pv.getValue();
            Object valueToApply = originalValue;
            if (!Boolean.FALSE.equals(pv.conversionNecessary)) {
                if (pv.isConverted()) {
                } else {
					//通过参数类型转换器，转换为目标参数的类型；
                    valueToApply = this.convertForProperty(tokens.canonicalName, oldValue, originalValue, ph.toTypeDescriptor());
                }
            }
            //设置属性值；
            ph.setValue(valueToApply);
        } catch (Exception var11) {
            //...
        }
    }
    //...
}

@Nullable
protected Object convertForProperty(String propertyName, @Nullable Object oldValue, @Nullable Object newValue, TypeDescriptor td) throws TypeMismatchException {
	return this.convertIfNecessary(propertyName, oldValue, newValue, td.getType(), td);
}

@Nullable
private Object convertIfNecessary(@Nullable String propertyName, @Nullable Object oldValue, @Nullable Object newValue, @Nullable Class<?> requiredType, @Nullable TypeDescriptor td) throws TypeMismatchException {
        PropertyChangeEvent pce;
        try {
            return this.typeConverterDelegate.convertIfNecessary(propertyName, oldValue, newValue, requiredType, td);
        } catch (IllegalStateException | ConverterNotFoundException var8) {
		//....
        }
}
//调用类型转换器代理器去进行参数格式转换；
//typeConverterDelegate的属性WebConverterService中，包装了128个参数格式转换器；
@Nullable
public <T> T convertIfNecessary(@Nullable String propertyName, @Nullable Object oldValue, @Nullable Object newValue, @Nullable Class<T> requiredType, @Nullable TypeDescriptor typeDescriptor) throws IllegalArgumentException {
        PropertyEditor editor = this.propertyEditorRegistry.findCustomEditor(requiredType, propertyName);
        ConversionFailedException conversionAttemptEx = null;
        ConversionService conversionService = this.propertyEditorRegistry.getConversionService();
        if (editor == null && conversionService != null && newValue != null && typeDescriptor != null) {
            TypeDescriptor sourceTypeDesc = TypeDescriptor.forObject(newValue);
            if (conversionService.canConvert(sourceTypeDesc, typeDescriptor)) {
                try {
                    return conversionService.convert(newValue, sourceTypeDesc, typeDescriptor);
                } catch (ConversionFailedException var14) {
                    conversionAttemptEx = var14;
                }
            }
        }
 //...
 }
```

![WebConversionService](assets/\image-20221023221357433.png)

![TypeConverterDalegate#webConversionService#converters](assets/\image-20221023221444065.png)

![部分参数格式转换器](assets/\image-20221023221659741.png)



```java
//BeamWrapperImpl$BeanPropertyHandler#setValue
public void setValue(@Nullable Object value) throws Exception {
        Method writeMethod = this.pd instanceof GenericTypeAwarePropertyDescriptor ? ((GenericTypeAwarePropertyDescriptor)this.pd).getWriteMethodForActualAccess() : this.pd.getWriteMethod();
        if (System.getSecurityManager() != null) {
            	//....
        } else {
            ReflectionUtils.makeAccessible(writeMethod);
            writeMethod.invoke(BeanWrapperImpl.this.getWrappedInstance(), value);
        }

    }
}
```

**获取属性的setter方法实例writeMethod, 然后通过反射调用setter方法，给之前用构造器创建实例进行设值；**

总结：

1、 web数据绑定机制 先将请求参数全部转换为 MutablePropertyValues实例，该实例内部包装了所有的参数键值对集合，类型为PropertyValue

2、 遍历键值对集合

3、 获取键值对的key,value,  http中，数据都为文本格式，即value的类型都为String;

4、 根据键值对的key获取属性描述器PropertyDescriptor， 该描述中包含了getter，setter方法；

5、 调用convertForProperty参数类型转换器将value转换为目标类型的参数值、SpringBoot中提供124中参数类型转换器，包含了String->Integer类型的转换器

6、 通过PropertyDescriptor的setter方法，通过反射调用setter方法，给之前用构造器创建实例进行设值

## 6. 请求处理-【源码分析】-自定义Converter原理

将字符串`“啊猫,3”`转换成`Pet`对象。

```java
@ResponseBody
@PostMapping("modelAttribute2")
public Object modelAttr2(Pet pet) {
    System.out.println(pet);
    return pet;
}
```

```java
@Bean
public WebMvcConfigurer webMvcConfigurer(){
    return new WebMvcConfigurer() {

        @Override
        public void addFormatters(FormatterRegistry registry) {
            registry.addConverter(new Converter<String, Pet>() {

                @Override
                public Pet convert(String source) {
                    // 啊猫,3
                    if(!StringUtils.isEmpty(source)){
                        Pet pet = new Pet();
                        String[] split = source.split(",");
                        pet.setName(split[0]);
                        pet.setAge(split[1]);
                        return pet;
                    }
                    return null;
                }
            });
        }
    };
}
```

创建一个匿名的参数转换器， 获取参数为pet的字符串，然后根据","分割， 设置Pet的属性；

