# SpringBoot(四)

## 视图解析-视图解析器与视图

1. 调用完HandlerAdapter#Handle方法过程中，通过反射执行完目标方法；
2. 拿到返回值、经过ReturnValueHandler处理后，视图view与模型数据都会放在ModelAndViewContainer，最终返回ModelAndView实例
3. DispatcherServlet#processDispatchResult处理分发结果；
4. 调用DispatcherServlet#render方法
   1. 获取返回的ViewName值，再根据viewName拿到View视图实例；
      - 遍历视图解析器ViewResolver#resolveViewName、根据当前的viewName拿到View实例返回；
      - 由ContentNegotiationViewResolver负责处理，其内部包装了所有的视图解析器， 包含了ThymeleafViewResolver、InternalResouceViewResolver视图解析器；
        -  redirect:/index.html ==> ThymeleafViewResolver创建RedirectView重定向视图实例， 放在结果集第一个；
        -  redirect:/index.html ==> InternalResouceViewResolver创建RedirectView实例，放在结果集的第二个
        -  调用getBestView方法后，拿到最合适的RedirectView视图实例返回；
   2. view.render(mv.getModelInternal(), request, response) 视图调用自定义render方法处理
      - 获取目标url地址targetUrl
      - 调用HttpServletResponse#redirect(targetUrl)进行重定向；

---

- 转发： 返回值为**“forward: xxx”**的转发结果值， ThymeleafViewResolver==>InternalResourceView视图，底层会调用request#getRequestDipatcher(path).forward(request, response)进行转发处理
- 普通string： 如果是普通的“xxxx”的返回值、 ThymeleafViewResolver ==> ThymeleafView视图进行处理；

### 源码分析

```java
//DispatcherServlet#render
/*
*1. 拿到视图名viewName
*2. 根据视图名拿到视图view
*3. 调用视图view的渲染render
*/
protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {

   View view;
   String viewName = mv.getViewName();
   if (viewName != null) {
      // We need to resolve the view name.
      view = resolveViewName(viewName, mv.getModelInternal(), locale, request);
 
   }
	try {
		view.render(mv.getModelInternal(), request, response);
	}catch (Exception ex) {}
	} 
   		
}
// 2. 根据视图名viewName拿到视图View;
// 遍历视图解析器ViewResovler，解析获取视图实例；
// 注： 通常情况下，一个ContentNegotiationViewResolver#revolveViewName就可以获得视图结果
@Nullable
protected View resolveViewName(String viewName, @Nullable Map<String, Object> model,
		Locale locale, HttpServletRequest request) throws Exception {
	for (ViewResolver viewResolver : this.viewResolvers) {
		View view = viewResolver.resolveViewName(viewName, locale);
		if (view != null) {
			return view;
		}
	}
	return null;
}
```

#### 获取最合适的视图View

```java
//调用ContentNegotiationViewResolver内容视图解析器获取视图实例；
//1. 获取到候选的视图结果集合；
//2. 获得最合适的视图；
//3. 返回；
@Override
@Nullable
public View resolveViewName(String viewName, Locale locale) throws Exception {
   RequestAttributes attrs = RequestContextHolder.getRequestAttributes();
   List<MediaType> requestedMediaTypes = getMediaTypes(((ServletRequestAttributes) attrs).getRequest());
   if (requestedMediaTypes != null) {
      List<View> candidateViews = getCandidateViews(viewName, locale, requestedMediaTypes);
      View bestView = getBestView(candidateViews, requestedMediaTypes, attrs);
      if (bestView != null) {
         return bestView;
      }
   }
//....
    
    
}
//ContentNegotiationViewResolver内部包含所有的视图解析器；
//调用Resolver#resolverName获取视图后，加入candidateViews，遍历完，返回；
private List<View> getCandidateViews(String viewName, Locale locale, List<MediaType> requestedMediaTypes) throws Exception {
		List<View> candidateViews = new ArrayList<>();
		if (this.viewResolvers != null) {
			for (ViewResolver viewResolver : this.viewResolvers) {
				View view = viewResolver.resolveViewName(viewName, locale);
				if (view != null) {
					candidateViews.add(view);
				}
			}
		}
		return candidateViews;
}
//1. 如果View结果集中包含了RedirectView， 则返回该view;
//2. 如果没有RedirectView， 遍历媒体类型集合、媒体类型若与视图的内容类型匹配，则返回该View实例；
private View getBestView(List<View> candidateViews, List<MediaType> requestedMediaTypes, RequestAttributes attrs) {
		for (View candidateView : candidateViews) {
			if (candidateView instanceof SmartView) {
				SmartView smartView = (SmartView) candidateView;
				if (smartView.isRedirectView()) {
					return candidateView;
				}
			}
		}
		for (MediaType mediaType : requestedMediaTypes) {
			for (View candidateView : candidateViews) {
				if (StringUtils.hasText(candidateView.getContentType())) {
					MediaType candidateContentType = MediaType.parseMediaType(candidateView.getContentType());
					if (mediaType.isCompatibleWith(candidateContentType)) {
						attrs.setAttribute(View.SELECTED_CONTENT_TYPE, mediaType, RequestAttributes.SCOPE_REQUEST);
						return candidateView;
					}
				}
			}
		}
		return null;
	}

```

##### 视图创建

```java
//ThymeleafViewResolver继承了AbstractViewResolver，其实现了resolveViewName；
//1. isCache默认为true;
//2. 会从viewAccessCache缓存中获取视图、
//3. 不存在、则获取viewCraeteCache视图创建缓存实例的锁；
//4. 判断viewCreateCache缓存中是否存在视图、不存在则创建视图View实例；
//5. 创建完放入viewAccessCache视图访问缓存、viewCreationCache视图创建缓存中；

//类似双端检索机制、synchronized保证串行执行、而viewCreationCache作为锁、则是保证每个视图只创建一次；
// 多线程下、同时访问viewAccessCache相同的视图缓存，若不存在、只能有一个线程去参加、创建完放入viewCreationCache缓存中，下一个线程获得viewCreationCache时、就可以获取视图实例返回；
@Override
@Nullable
public View resolveViewName(String viewName, Locale locale) throws Exception {
   if (!isCache()) {
      return createView(viewName, locale);
   }
   else {
      Object cacheKey = getCacheKey(viewName, locale);
      View view = this.viewAccessCache.get(cacheKey);
      if (view == null) {
         synchronized (this.viewCreationCache) {
            view = this.viewCreationCache.get(cacheKey);
            if (view == null) {
               // Ask the subclass to create the View object.
               view = createView(viewName, locale);
               if (view == null && this.cacheUnresolved) {
                  view = UNRESOLVED_VIEW;
               }
               if (view != null && this.cacheFilter.filter(view, viewName, locale)) {
                  this.viewAccessCache.put(cacheKey, view);
                  this.viewCreationCache.put(cacheKey, view);
               }
            }
         }
      }
      return (view != UNRESOLVED_VIEW ? view : null);
   }
}
```

```java
//、AbstractViewResolver默认实现但ThymeleafViewResolver重写了该方法；
//1. 判断是否为redirect:开头、是则创建RedirectView实例；
//2. 判断是否为forward:开头、是则创建InterResourceView实例；
//3. 都不是、则loadView创建ThymeleafView实例；
@Override
protected View createView(final String viewName, final Locale locale) throws Exception {
    if (!this.alwaysProcessRedirectAndForward && !canHandle(viewName, locale)) {
        return null;
    }
  
    if (viewName.startsWith(REDIRECT_URL_PREFIX)) {
        final String redirectUrl = viewName.substring(REDIRECT_URL_PREFIX.length(), viewName.length());
        final RedirectView view = new RedirectView(redirectUrl, isRedirectContextRelative(), isRedirectHttp10Compatible());
        return (View) getApplicationContext().getAutowireCapableBeanFactory().initializeBean(view, viewName);
    }

    if (viewName.startsWith(FORWARD_URL_PREFIX)) {
        return new InternalResourceView(forwardUrl);
    }
    return loadView(viewName, locale);
}
```

#### 视图渲染

```java
//RedirectView继承了AbstractView、AbstractView实现了该方法；
@Override
public void render(@Nullable Map<String, ?> model, HttpServletRequest request,
      HttpServletResponse response) throws Exception {

   if (logger.isDebugEnabled()) {
      logger.debug("View " + formatViewName() +
            ", model " + (model != null ? model : Collections.emptyMap()) +
            (this.staticAttributes.isEmpty() ? "" : ", static attributes " + this.staticAttributes));
   }

   Map<String, Object> mergedModel = createMergedOutputModel(model, request, response);
   prepareResponse(request, response);
   renderMergedOutputModel(mergedModel, getRequestToExpose(request), response);
}

```



```java
//RdirectView实现了该方法、先获取目标路径targetUrl、再redirect重定向；
@Override
protected void renderMergedOutputModel(Map<String, Object> model, HttpServletRequest request,
      HttpServletResponse response) throws IOException {

   String targetUrl = createTargetUrl(model, request);
   targetUrl = updateTargetUrl(targetUrl, model, request, response);

   // Save flash attributes
   RequestContextUtils.saveOutputFlashMap(targetUrl, request, response);

   // Redirect
   sendRedirect(request, response, targetUrl, this.http10Compatible);
}

```

```java
protected void sendRedirect(HttpServletRequest request, HttpServletResponse response,
      String targetUrl, boolean http10Compatible) throws IOException {

   String encodedURL = (isRemoteHost(targetUrl) ? targetUrl : response.encodeRedirectURL(targetUrl));
   if (http10Compatible) {
      HttpStatus attributeStatusCode = (HttpStatus) request.getAttribute(View.RESPONSE_STATUS_ATTRIBUTE);
      if (this.statusCode != null) {
      }
      else {
         // 重定向
         response.sendRedirect(encodedURL);
      }
   }
    //...
}
```



## 拦截器 -【原理分析】

### 流程分析 

1. getHandler方法返回的对象为HandlerExecutionChain、包含处理器Handler、还包含拦截器Interceptors
2. 调用处理器执行链applyPreHandle方法
   - 正向遍历执行拦截器的preHandle方法，如果拦截器返回为true、则执行下一个拦截器的preHandle方法；当所有的拦截器都返回true、才会执行目标方法；
   - 其中第index个拦截器若返回false、则从第index个拦截器开始，从后往前、倒序执行拦截器的triggerAfterCompletion方法、只要有一个拦截器返回false、就不会执行目标方法；
3. 执行目标方法Handler;
4. 调用处理器执行链的applyPostHandle方法
   - 倒序、从后往前调用拦截器的postHandle方法
5. 调用DispatcherServlet#render渲染试图；
6. 调用处理器执行链的triggerAfterCompletion方法
   - 从后往前、依次执行拦截器的triggerAfterCompletion方法；

7. 1~6步骤被trycatchfinally包住、若发生异常、则调用处理器的执行链的triggerAfterCompletion方法；

![拦截器](E:\javamd文档\拦截器.png)

### 源码分析 

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	//声明处理器执行链
   HandlerExecutionChain mappedHandler = null;

   WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

   try {
      ModelAndView mv = null;
      Exception dispatchException = null;

      try {

         // 1. 拿到处理器执行链；
         mappedHandler = getHandler(processedRequest);


         // 拿到处理器适配器
         HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

		 //2. 调用拦截器的preHandle方法
         if (!mappedHandler.applyPreHandle(processedRequest, response)) {
            return;
         }

         // 3. 执行目标方法
         mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
		 
         // 4. 执行拦截器的postHandle
         mappedHandler.applyPostHandle(processedRequest, response, mv);
      }
      catch (Exception ex) {
         dispatchException = ex;
      }
      catch (Throwable err) {
         dispatchException = new NestedServletException("Handler dispatch failed", err);
      }
      processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
   }
   catch (Exception ex) {
       //7. 服务发生异常、从后往前调用拦截器的triggerAfterCompletion;
      triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
   }
   catch (Throwable err) {
      triggerAfterCompletion(processedRequest, response, mappedHandler,
            new NestedServletException("Handler processing failed", err));
   }
   finally {

   }
}


```

```java
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
      @Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
      @Nullable Exception exception) throws Exception {


   //  5. 视图渲染
   if (mv != null && !mv.wasCleared()) {
      render(mv, request, response);
      if (errorView) {
         WebUtils.clearErrorRequestAttributes(request);
      }
   }

	//6. 正常执行完成、调用拦截器的triggerAfterCompletion方法；
   if (mappedHandler != null) {
      // Exception (if any) is already handled..
      mappedHandler.triggerAfterCompletion(request, response, null);
   }
}
```



#### getHandler获取处理器拦截器

```java
@Override
@Nullable
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    //1. 拿到处理器；
   Object handler = getHandlerInternal(request);
   if (handler == null) {
      handler = getDefaultHandler();
   }
   if (handler == null) {
      return null;
   }
    //2. 根据请求路径、判断遍历所有拦截器是否匹配；
   HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
   return executionChain;
}
```

```java
protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
   HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?
         (HandlerExecutionChain) handler : new HandlerExecutionChain(handler));

   String lookupPath = this.urlPathHelper.getLookupPathForRequest(request, LOOKUP_PATH);
  //根据请求路径、判断拦截器是否匹配、匹配则加入；
    for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
      if (interceptor instanceof MappedInterceptor) {
         MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
         if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {
            chain.addInterceptor(mappedInterceptor.getInterceptor());
         }
      }
      else {
         chain.addInterceptor(interceptor);
      }
   }
   return chain;
}
```

#### 执行拦截器preHandle方法

```java
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
   HandlerInterceptor[] interceptors = getInterceptors();
   if (!ObjectUtils.isEmpty(interceptors)) {
      for (int i = 0; i < interceptors.length; i++) {
         HandlerInterceptor interceptor = interceptors[i];
          //执行preHandle方法；
         if (!interceptor.preHandle(request, response, this.handler)) {
            
             triggerAfterCompletion(request, response, null);
            return false;
         }
         //标记当前执行到哪个拦截器；
         this.interceptorIndex = i;
      }
   }
   return true;
}
```

interceptorIndex : 标记当前执行正在拦截器；

#### 执行拦截器的triggerAfterCompletion

```java
void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, @Nullable Exception ex)
      throws Exception {

   HandlerInterceptor[] interceptors = getInterceptors();
   if (!ObjectUtils.isEmpty(interceptors)) {
       //从正在执行的拦截器开始、倒序执行afterCompletion方法；
      for (int i = this.interceptorIndex; i >= 0; i--) {
         HandlerInterceptor interceptor = interceptors[i];
         try {
            interceptor.afterCompletion(request, response, this.handler, ex);
         }
         catch (Throwable ex2) {
            logger.error("HandlerInterceptor.afterCompletion threw exception", ex2);
         }
      }
   }
}
```

#### 执行拦截器的postHandle

```java
//HandlerExecutionChain#applyPostHandle
void applyPostHandle(HttpServletRequest request, HttpServletResponse response, @Nullable ModelAndView mv)  throws Exception {
   HandlerInterceptor[] interceptors = getInterceptors();
   if (!ObjectUtils.isEmpty(interceptors)) {
      for (int i = interceptors.length - 1; i >= 0; i--) {
         HandlerInterceptor interceptor = interceptors[i];
         interceptor.postHandle(request, response, this.handler, mv);
      }
   }
}
```



## 文件上传 - 【源码分析】

#### MultipartFile参数

```java
@Controller
//@ResponseBody 需要加该注解、 或者@Controller换@RestController
public class ThymeleafController {
    @PostMapping("/upload")
   // @ResponseBody
    public Boolean upload(@RequestPart(value = "image") MultipartFile image, MultipartFile phone) throws IOException {
        //图片操作 本地路径
        File file = new File("xxxx");

        //将流数据传输到本地文件
        image.transferTo(file);
        return Boolean.valueOf(true);
    }
}
```

返回基础数据类型时、会出现异常、**因为没有一个ReturnValueHandler可以处理该基础类型数据**、包括Integer、Boolean、Long；

解决方案：通常为第一种；

1. 加上@ResponseBody、让RequestResponseBodyMethodProcessor来处理；
2. 自定义一个ReturnValueHandler加入容器、使该返回值处理器可以支持基础数据类型；

#### MultipartFile参数 + 对象实体类参数

```java
@PostMapping(value = "/upload")
@ResponseBody
public Boolean upload(@RequestPart(value = "image") MultipartFile image,
                      MultipartFile phone,
                      @RequestPart(value ="pet") Pet name) throws IOException {
    //图片操作 本地路径
    File file = new File("xxxx");
    //将流数据传输到本地文件
    System.out.println("参数"+name);
    image.transferTo(file);
    return Boolean.valueOf(true);
}
```

通过表单传递该参数时、pet的类型应该为JSON格式的字符串、RequestPartMethodArgumentResolver会自动处理、将JSON数据转换为实体类；

### 参数MultipartFile处理流程

- 在反射执行该方法前、会确定参数、使用参数处理器处理MultipartFile参数、从请求体中获取流数据封装为MultipartFile实例；

- 如果是MultipartFile参数，不管加@RequestPart注解、会交由RequestPartMethodArgumentResolver参数处理器处理；如果不加@RequestParamMethodArugmentResolver参数处理器处理；

- supportParameters : 参数如果被@RequestPart注解修饰、 或者参数类型为MultipartFile则可处理该参数
- resolveArgument ： 解析参数
  - MultipartResolutionDelegate.resolveMultipartArgument ： 如果参数类型为MultipartFile、MultipartFile集合、MultipartFile数组、 Part、Part集合、Part数组、则可获得该参数的值；
  - this.readWithMessageConverters(inputMessage, parameter, parameter.getNestedGenericParameterType()) ： 如果非MultipartFile、Part类型、**则会通过消息转换器HttpMessageConverter获取；**

```java
public class RequestPartMethodArgumentResolver extends AbstractMessageConverterMethodArgumentResolver {
	//...
    public boolean supportsParameter(MethodParameter parameter) {
        //判断是被@RequestPart注解修饰、是则处理该参数
        if (parameter.hasParameterAnnotation(RequestPart.class)) {
            return true;
        } else {
            //如果没有@RequestPart注解、但是参数类型为MultipartFile、则可处理此参数
            return parameter.hasParameterAnnotation(RequestParam.class) ? false : MultipartResolutionDelegate.isMultipartArgument(parameter.nestedIfOptional());
        }
    }

    @Nullable
    public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest request, @Nullable WebDataBinderFactory binderFactory) throws Exception {
        HttpServletRequest servletRequest = (HttpServletRequest)request.getNativeRequest(HttpServletRequest.class);
        RequestPart requestPart = (RequestPart)parameter.getParameterAnnotation(RequestPart.class);
        boolean isRequired = (requestPart == null || requestPart.required()) && !parameter.isOptional();
        String name = this.getPartName(parameter, requestPart);
        parameter = parameter.nestedIfOptional();
        Object arg = null;
        //处理参数、如果为MultipartFile、 MultipartFile集合、MultipartFile数组，Part则通过此方法直接		 //可以获取到结果、 如果@RequestPart修饰的参数类型为String、则返回为UNRESOLVABLE
        Object mpArg = MultipartResolutionDelegate.resolveMultipartArgument(name, parameter, servletRequest);
        if (mpArg != MultipartResolutionDelegate.UNRESOLVABLE) {
            arg = mpArg;
        } else {
            try {
                HttpInputMessage inputMessage = new RequestPartServletServerHttpRequest(servletRequest, name);
                arg = this.readWithMessageConverters(inputMessage, parameter, parameter.getNestedGenericParameterType());
                if (binderFactory != null) {
					//...
                    }
                }
            } catch (MultipartException | MissingServletRequestPartException var13) {}
        }
		//...
 		return this.adaptArgumentIfNecessary(arg, parameter);
    }
    
	//....
}
```

## 异常处理 - 【原理分析】

### 异常处理机制组件

- ErrorMvcAutoConfiguration ： 异常自动配置
  - **容器中的组件**：类型：`DefaultErrorAttributes` -> id：`errorAttributes`
  - public class DefaultErrorAttributes implements ErrorAttributes, HandlerExceptionResolver
    **DefaultErrorAttributes**：定义错误页面中可以包含数据（异常明细，堆栈信息等）。
  - 容器中的组件：类型：**BasicErrorController** => id: basicErrorController;
    - 包含多种处理方式、其中两种、一种为返回JSON数据，另一种返回HTML页面；
    - 处理/error 的请求、如果响应页面、则返回new ModelAndView("error", model)
      - 容器中、存在View视图Bean、id为error;
      - 利用BeanNameViewResolver视图解析器进行处理该modelAndView;
    - 处理/error的请求、如果响应JSON、则返回ResponseEntity实例、返回json数据；
- ErrorMvcAutoConfituration$DefaultErrorViewResolverConfiguration:默认异常视图解析器配置
  - 容器中的组件：类型：**DefaultErrorViewResolver** => id:defaultErrorViewResovler；
    - 如果发生异常、会根据异常状态码作为视图地址， 找到error/4xx | 5xx的页面；

---

### 异常源码流程分析

异常代码：

```java
@Slf4j
@RestController
public class HelloController {
    @RequestMapping("/hello")
    public String handle01(){
        int i = 1 / 0;//将会抛出ArithmeticException
        log.info("Hello, Spring Boot 2!");
        return "Hello, Spring Boot 2!";
    }
}
```

DispatcherServlet#doDispatch()中、当HandlerAdapter#handle时、会反射执行业务方法、会捕捉到异常、并进行DispatcherServlet#processDispatchResult执行视图渲染；

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
   try {
       Exception dispatchException = null;
      try {
          //1. 执行目标方法
         mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
      }
      catch (Exception ex) {
         //2. 如果执行出现异常、则异常会被捕捉
         dispatchException = ex;
      }catch (Throwable err) {
          //3. 如果执行出现异常、则异常会被捕捉
         dispatchException = new NestedServletException("Handler dispatch failed", err);
      }
       //4. 捕捉到异常之后、进行异常视图渲染
      processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
   }
   catch (Exception ex) {
       //5. 如果视图渲染没能处理掉业务异常、triggerAfterCompletion内部会再抛出异常；
      triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
   }
   catch (Throwable err) {
      triggerAfterCompletion(processedRequest, response, mappedHandler,
            new NestedServletException("Handler processing failed", err));
   }
   finally {
	//....
   }
}
```

业务发生异常、还是会进入processDispatchResult方法处理、

```java
//异常不为空、则会调用processHandleException处理异常、返回ModelAndView实例
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response, @Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
      @Nullable Exception exception) throws Exception {
   boolean errorView = false;
   if (exception != null) {
      if (exception instanceof ModelAndViewDefiningException) {}
      else {
         Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
         mv = processHandlerException(request, response, handler, exception);
         errorView = (mv != null);
      }
   }

   //....省略
}
```





```java
//1. 这里会遍历异常解析器HandlerExceptionResolver， 调用其resolveException处理、返回ModelAndView，如果异常解析器无法处理该异常、返回的ModelAndView实例为空；

//2. 我们业务中没有自定义异常处理、如果出现异常、而Spring的异常处理机制中、只会处理一些框架出现的异常；
//像参数不匹配、参数不合法等异常、而我们业务抛出的异常、Spring的异常处理不了;

//3。 导致throw ex再次抛出异常、回到上一层的doDispatch方法、
@Nullable
protected ModelAndView processHandlerException(HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) throws Exception {
   request.removeAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);

   // Check registered HandlerExceptionResolvers...
   ModelAndView exMv = null;
   if (this.handlerExceptionResolvers != null) {
      for (HandlerExceptionResolver resolver : this.handlerExceptionResolvers) {
         exMv = resolver.resolveException(request, response, handler, ex);
         if (exMv != null) {
            break;
         }
      }
   }
	//....

   throw ex;
}
```

![HandlerExceptionResolver异常解析器](assets/\image-20221127150834920.png)

- DefaultErrorAttributes: 往请求中加入异常信息, 返回为null；

```java
//DefaultErrorAttributes#resolveException
@Override
public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler,
      Exception ex) {
   storeErrorAttributes(request, ex);
   return null;
}

private void storeErrorAttributes(HttpServletRequest request, Exception ex) {
   request.setAttribute(ERROR_ATTRIBUTE, ex);
}
```

- HandlerExceptionResolverComposite: 异常解析器包装类、内部包装了三个异常解析器、其resolveException方法再次遍历异常解析器,由于是业务异常、返回的modelAndView实例为null

```java
//HandlerExceptionResolverComposite#resolveException
@Override
@Nullable
public ModelAndView resolveException(
      HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {

   if (this.resolvers != null) {
      for (HandlerExceptionResolver handlerExceptionResolver : this.resolvers) {
         ModelAndView mav = handlerExceptionResolver.resolveException(request, response, handler, ex);
         if (mav != null) {
            return mav;
         }
      }
   }
   return null;
}
```

由于除零异常，Spring的异常解析器没法处理该异常、processHandlerException因此最终返回会throw ex抛出异常；

----

processDispatchResult抛出异常、其被try-catch-finally包装、会异常被catch代码块处理、

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
   try{ 
       try{
		//1. 业务处理发生异常
         mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

      }
      catch (Exception ex) {
         //2. 第一层catch捕捉了异常、
         dispatchException = ex;
      }
      catch (Throwable err) {
      }
       //3. 第一次处理异常、由于业务异常无法处理、再次抛出异常 throw ex
      processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
   }
   catch (Exception ex) {
       //4. 捕捉了ex异常后、 遍历拦截器方法、最终再次抛出异常、没人可以处理此异常；
      triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
   }
}
```

```java
private void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response,@Nullable HandlerExecutionChain mappedHandler, Exception ex) throws Exception {
   if (mappedHandler != null) {
      mappedHandler.triggerAfterCompletion(request, response, ex);
   }
    //再次抛出异常
   throw ex;
}
```

5. 由于无法处理、最终tomcat底层就会转发`/error` 请求、而BasicErrorController可处理/error请求；

```java
@Controller
@RequestMapping("${server.error.path:${error.path:/error}}")
public class BasicErrorController extends AbstractErrorController {
	//....
   @RequestMapping(produces = MediaType.TEXT_HTML_VALUE)
   public ModelAndView errorHtml(HttpServletRequest request, HttpServletResponse response) {
       //获取异常状态码
      HttpStatus status = getStatus(request);
       //异常数据
      Map<String, Object> model = Collections
            .unmodifiableMap(getErrorAttributes(request, getErrorAttributeOptions(request, MediaType.TEXT_HTML)));
      response.setStatus(status.value());
       //解析异常视图、返回ModelAndView实例；调用DefaultErrorViewResolver解析异常、返回ModelAndView
      ModelAndView modelAndView = resolveErrorView(request, response, status, model);
       
       //如果resolveErrorView返回的modelAndView实例为null、则会以error为视图名、异常信息model创建ModelAndView实例；
      return (modelAndView != null) ? modelAndView : new ModelAndView("error", model);
   }
    //调用DefaultErrorViewResolver解析异常、返回ModelAndView实例、不为空；
	protected ModelAndView resolveErrorView(HttpServletRequest request, HttpServletResponse response, HttpStatus status,Map<String, Object> model) {
		for (ErrorViewResolver resolver : this.errorViewResolvers) {
			ModelAndView modelAndView = 
                resolver.resolveErrorView(request, status, model);
			}
		}
		return null;
	}    
}

```

```java
//DefaultErrorViewResolver:默认异常视图解析器、最终调用resolve方法；
//1. 状态码为errorView、以异常信息为model、如果资源路径下存在404、500.html,则会创建ModelAndView实例、以404.html | 500.html为视图名、创建ModelAndView实例返回；

//2. 如果没有404.html | 500.html 具体的状态码页面、ModelAndView实例为空、调用resolve(SERIES_VIEWS.get(status.series()), model)、其视图名为4xx | 5xx， 会查找4xx.html、 5xx.html资源文件；存在、则会以4xx.html | 5xx.html作为视图名、创建ModelAndView实例返回； 不存在则返回Null;
@Override
public ModelAndView resolveErrorView(HttpServletRequest request, HttpStatus status, Map<String, Object> model) {
   ModelAndView modelAndView = resolve(String.valueOf(status.value()), model);
   if (modelAndView == null && SERIES_VIEWS.containsKey(status.series())) {
      modelAndView = resolve(SERIES_VIEWS.get(status.series()), model);
   }
   return modelAndView;
}

private ModelAndView resolve(String viewName, Map<String, Object> model) {
   String errorViewName = "error/" + viewName;
   return resolveResource(errorViewName, model);
}

//从资源路径下获取 viewName.html 找到则返回ModelAndView实例、没有返回null：
private ModelAndView resolveResource(String viewName, Map<String, Object> model) {
	for (String location : this.resourceProperties.getStaticLocations()) {
		try {
			Resource resource = this.applicationContext.getResource(location);
			resource = resource.createRelative(viewName + ".html");
			if (resource.exists()) {
				return new ModelAndView(new HtmlResourceView(resource), model);
			}
		}
		catch (Exception ex) {}
		}
	return null;
}
```

HanderAdapter#handle执行结束、执行processDispatchResult视图渲染，容器存在名为error,类型为StaticView的Bean实例、因此被会BeanNameViewResolver处理；

-----------------------------------------------------------------------------------

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
   try {
      Exception dispatchException = null;
      try {
         //error请求执行完、返回ModelAndView实例
         mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
      }
      catch (Exception ex) {
         dispatchException = ex;
      }
      catch (Throwable err) {}
      //执行异常视图渲染
      processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
   }
   catch (Exception ex) {
      triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
   }
}
//执行异常视图渲染
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response, @Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
      @Nullable Exception exception) throws Exception {
   //执行异常视图渲染
   if (mv != null && !mv.wasCleared()) {
      render(mv, request, response);
      if (errorView) {
         WebUtils.clearErrorRequestAttributes(request);
      }
   }
	//...
}
```

```java
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response, @Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
      @Nullable Exception exception) throws Exception {
   //执行异常视图渲染
   if (mv != null && !mv.wasCleared()) {
      render(mv, request, response);
      if (errorView) {
         WebUtils.clearErrorRequestAttributes(request);
      }
   }
    

}
//1. 通过视图解析器解析最终获取的视图为StaticView的实例、
//2. 再调用StaticView#render渲染数据
protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {

   View view;
   String viewName = mv.getViewName();
   if (viewName != null) {
      view = resolveViewName(viewName, mv.getModelInternal(), locale, request);
   }

   try {
       //
      view.render(mv.getModelInternal(), request, response);
   }
   catch (Exception ex) {}
}

//遍历视图解析器、但是ContentNegotiationViewResolver获取到视图StaticView;
protected View resolveViewName(String viewName, @Nullable Map<String, Object> model,
      Locale locale, HttpServletRequest request) throws Exception {

   if (this.viewResolvers != null) {
      for (ViewResolver viewResolver : this.viewResolvers) {
         View view = viewResolver.resolveViewName(viewName, locale);
         if (view != null) {
            return view;
         }
      }
   }
   return null;
}
```

异常处理中、由于容器中存在id为error

```java
//ContentNegotiationViewResolver: 会遍历内部的视图解析器、 getCandidateViews获取候选的View、 
//getBestView方法根据内容协商拿到最合适的View实例返回；
//遍历中、类型为BeanNameViewResolver的视图解析器会获取到BeanId为error，类型为StaticView的视图实例；
@Override
@Nullable
public View resolveViewName(String viewName, Locale locale) throws Exception {
   RequestAttributes attrs = RequestContextHolder.getRequestAttributes();
   List<MediaType> requestedMediaTypes = getMediaTypes(((ServletRequestAttributes) attrs).getRequest());
   if (requestedMediaTypes != null) {
      List<View> candidateViews = getCandidateViews(viewName, locale, requestedMediaTypes);
      View bestView = getBestView(candidateViews, requestedMediaTypes, attrs);
      if (bestView != null) {
         return bestView;
      }
   }
}
```

```java
//BeanNameViewResolver#resolveViewName：以视图名为error作为BeanId、获取类型为View的Bean实例返回；
//容器存在名为error类型为StaticView的Bean实例
@Override
@Nullable
public View resolveViewName(String viewName, Locale locale) throws BeansException {
   ApplicationContext context = obtainApplicationContext();
   if (!context.containsBean(viewName)) {
      return null;
   }
   return context.getBean(viewName, View.class);
}
```

最终视图解析器返回StaticView实例；

```java
//1. 通过视图解析器解析最终获取的视图为StaticView的实例、
//2. 再调用StaticView#render渲染数据
protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {

   View view;
   String viewName = mv.getViewName();
   if (viewName != null) {
      view = resolveViewName(viewName, mv.getModelInternal(), locale, request);
   }

   try {
       //StaticView的渲染数据
      view.render(mv.getModelInternal(), request, response);
   }
   catch (Exception ex) {}
}
```



```java
//ErrorMvcAutoConfiguration$StaticView : 该类为ErrorMvcAutoConfiguration的内部类；
//拼接error的HTML页面的数据、在写入response响应内容中；
private static class StaticView implements View {

   private static final MediaType TEXT_HTML_UTF8 = new MediaType("text", "html", StandardCharsets.UTF_8);
   @Override
   public void render(Map<String, ?> model, HttpServletRequest request, HttpServletResponse response) throws Exception {
      response.setContentType(TEXT_HTML_UTF8.toString());
      StringBuilder builder = new StringBuilder();
      Object timestamp = model.get("timestamp");
      Object message = model.get("message");
      Object trace = model.get("trace");
      if (response.getContentType() == null) {
         response.setContentType(getContentType());
      }
      builder.append("<html><body><h1>Whitelabel Error Page</h1>").append(
            "<p>This application has no explicit mapping for /error, so you are seeing this as a fallback.</p>")
            .append("<div id='created'>").append(timestamp).append("</div>")
            .append("<div>There was an unexpected error (type=").append(htmlEscape(model.get("error")))
            .append(", status=").append(htmlEscape(model.get("status"))).append(").</div>");
      if (message != null) {
         builder.append("<div>").append(htmlEscape(message)).append("</div>");
      }
      if (trace != null) {
         builder.append("<div style='white-space:pre-wrap;'>").append(htmlEscape(trace)).append("</div>");
      }
      builder.append("</body></html>");
      response.getWriter().append(builder.toString());
   }

   private String getMessage(Map<String, ?> model) {
      Object path = model.get("path");
      String message = "Cannot render error page for request [" + path + "]";
      if (model.get("message") != null) {
         message += " and exception [" + model.get("message") + "]";
      }
      message += " as the response has already been committed.";
      message += " As a result, the response may have the wrong status code.";
      return message;
   }

}
```

最终浏览器显示的页面就是这样的：

![error.html](assets/\image-20221127163603133.png)

### 异常处理 - 几种异常处理应用与原理分析

#### 1、自定义错误页

- 在template/error目录下、新建404.html 、500.html的精确异常状态码页面

- 在template/error目录下、新建4xx.html、5xx.html的统一异常页面
  - 依赖于BasicErrorController、StaticView、DefaultErrorViewResolver组件；



#### 2、@ControllerAdvice + @ExceptionHandler

```java
@ControllerAdvice
public class AnnotationExceptionHandler {

    @ExceptionHandler(value = {ArithmeticException.class, NullPointerException.class})
    public ModelAndView resolveMath(Exception ex) {
        System.out.println("发生异常" + ex);
        HashMap<String, Object> model = new HashMap<>();
        model.put("message", ex.getMessage());
        return new ModelAndView("4xx", model);
    }
}
```

- 底层利用的是 ExceptionHandlerExceptionResolver组件

  - 继承关系

  ![ExceptionHandlerExceptionResolver](assets/\image-20221127213156775.png)

  - 源码流程

  resolveException => doResolveException =>do

  ```java
  //ExceptionHandlerExceptionResolver#resolveException、该方法在其父类AbstractHandlerExceptionResolver中实现
  @Override
  @Nullable
  public ModelAndView resolveException(
        HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {
     if (shouldApplyTo(request, handler)) {
        prepareResponse(ex, response);
        ModelAndView result = doResolveException(request, response, handler, ex);
   		
        return result;
     }
     else {
        return null;
     }
  }
  //父类定义的抽象方法doResolveException， 子类实现
  @Nullable
  protected abstract ModelAndView doResolveException(
        HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex);
  
  //子类AbstractHandlerMethodExceptionResolver实现了该方法
  @Override
  @Nullable
  protected final ModelAndView doResolveException(
        HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {
  
     return doResolveHandlerMethodException(request, response, (HandlerMethod) handler, ex);
  }
  //子类AbstractHandlerMethodExceptionResolver定义的抽象方法、由子类ExceptionHandlerExceptionResolver实现；
  @Nullable
  protected abstract ModelAndView doResolveHandlerMethodException(
        HttpServletRequest request, HttpServletResponse response, @Nullable HandlerMethod handlerMethod, Exception ex);
  
  ```

  ```java
  //ExceptionHandlerExceptionResolver#doResolveHandlerMethodException
  //1. 首先会获取目标的ExceptionHandler注解修饰的方法Method; 这个步骤有点意思；
  //2. 设置方法参数解析器；
  //3. 设置方法返回值处理器；
  //4. invokeAndHandle 反射执行目标方法 
  //	4.1 利用方法参数处理器解析出参数args、
  //	4.2 反射调用Method：
  //  4.3 处理返回值
  //5. 创建ModelAndView实例、设置viewName和数据返回；
  //6. 返回正常流程、进行异常视图渲染
  @Override
  @Nullable
  protected ModelAndView doResolveHandlerMethodException(HttpServletRequest request,
        HttpServletResponse response, @Nullable HandlerMethod handlerMethod, Exception exception) {
  
     ServletInvocableHandlerMethod exceptionHandlerMethod = getExceptionHandlerMethod(handlerMethod, exception);
     if (exceptionHandlerMethod == null) {
        return null;
     }
  
     if (this.argumentResolvers != null) {
        exceptionHandlerMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
     }
     if (this.returnValueHandlers != null) {
        exceptionHandlerMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
     }
  
     ServletWebRequest webRequest = new ServletWebRequest(request, response);
     ModelAndViewContainer mavContainer = new ModelAndViewContainer();
  
     try {
        Throwable cause = exception.getCause();
        if (cause != null) {
           exceptionHandlerMethod.invokeAndHandle(webRequest, mavContainer, exception, cause, handlerMethod);
        } else {
           exceptionHandlerMethod.invokeAndHandle(webRequest, mavContainer, exception, handlerMethod);
        }
     }
     catch (Throwable invocationEx) {
     }
  
     if (mavContainer.isRequestHandled()) {
        return new ModelAndView();
     }
     else {
        ModelMap model = mavContainer.getModel();
        HttpStatus status = mavContainer.getStatus();
        ModelAndView mav = new ModelAndView(mavContainer.getViewName(), model, status);
        mav.setViewName(mavContainer.getViewName());
        return mav;
     }
  }
  ```

  第一个步骤获取ServletInvocationHandlerMethod方法中、

  ```java
  //ExceptionHandlerExceptionResolver#ServletInvocationHandlerMethod
  protected ServletInvocableHandlerMethod getExceptionHandlerMethod(
        @Nullable HandlerMethod handlerMethod, Exception exception) {
  
     Class<?> handlerType = null;
  
     if (handlerMethod != null) {
        // Local exception handler methods on the controller class itself.
        // To be invoked through the proxy, even in case of an interface-based proxy.
        handlerType = handlerMethod.getBeanType();
        ExceptionHandlerMethodResolver resolver = this.exceptionHandlerCache.get(handlerType);
        if (resolver == null) {
           resolver = new ExceptionHandlerMethodResolver(handlerType);
           this.exceptionHandlerCache.put(handlerType, resolver);
        }
        Method method = resolver.resolveMethod(exception);
        if (method != null) {
           return new ServletInvocableHandlerMethod(handlerMethod.getBean(), method);
        }
        // For advice applicability check below (involving base packages, assignable types
        // and annotation presence), use target class instead of interface-based proxy.
        if (Proxy.isProxyClass(handlerType)) {
           handlerType = AopUtils.getTargetClass(handlerMethod.getBean());
        }
     }
      
  	//每个@ControllerAdvice 都会转换为一个Bean、 然后类中的异常处理器方法最终被包装为ExceptionHandlerMethodResolver； 
      //@ExceptionHandler(value = {xxx, xxx}), 会被解析为两个异常处理器、一个异常类型对应一个异常方法处理器；
      //会根据异常类型为key， 拿到异常处理器；最后包装为ServletInvocationHandlerMethod方法；
     for (Map.Entry<ControllerAdviceBean, ExceptionHandlerMethodResolver> entry : this.exceptionHandlerAdviceCache.entrySet()) {
        ControllerAdviceBean advice = entry.getKey();
        if (advice.isApplicableToBeanType(handlerType)) {
           ExceptionHandlerMethodResolver resolver = entry.getValue();
           Method method = resolver.resolveMethod(exception);
           if (method != null) {
              return new ServletInvocableHandlerMethod(advice.resolveBean(), method);
           }
        }
     }
  
     return null;
  }
  ```

  

![ControllerAdviceBean](assets/\image-20221127222101835.png)



#### 3、 `@ResponseStatus`+自定义异常

底层是 `ResponseStatusExceptionResolver` ，把responseStatus注解的信息底层调用 `response.sendError(statusCode, resolvedReason)`，tomcat发送的`/error`请求、由BasicErrorController进行处理

```java
@ResponseStatus(value= HttpStatus.FORBIDDEN,reason = "用户数量太多")
public class UserTooManyException extends RuntimeException {

    public  UserTooManyException(){

    }
    public  UserTooManyException(String message){
        super(message);
    }
}
```

#### 4、 框架异常

例如方法参数不匹配、类型不匹配、缺失参数等框架异常、一般都有框架自己定义的异常处理器处理；

底层为 DefaultHandlerExceptionResolver 异常解析器、最终response.sendError方式交由tomcat处理、tomcat发送/error请求、再由BasicErrorController处理；

```java
//DefaultHandlerExceptionResolver#doResolveException;
@Nullable
protected ModelAndView doResolveException(
      HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {

   try {
      if (ex instanceof HttpRequestMethodNotSupportedException) {
         return handleHttpRequestMethodNotSupported(
               (HttpRequestMethodNotSupportedException) ex, request, response, handler);
      }
      else if (ex instanceof HttpMediaTypeNotSupportedException) {
         return handleHttpMediaTypeNotSupported(
               (HttpMediaTypeNotSupportedException) ex, request, response, handler);
      }
      else if (ex instanceof HttpMediaTypeNotAcceptableException) {
         return handleHttpMediaTypeNotAcceptable(
               (HttpMediaTypeNotAcceptableException) ex, request, response, handler);
      }
      else if (ex instanceof MissingPathVariableException) {
         return handleMissingPathVariable(
               (MissingPathVariableException) ex, request, response, handler);
      }
      else if (ex instanceof MissingServletRequestParameterException) {
         return handleMissingServletRequestParameter(
               (MissingServletRequestParameterException) ex, request, response, handler);
      }
      else if (ex instanceof ServletRequestBindingException) {
         return handleServletRequestBindingException(
               (ServletRequestBindingException) ex, request, response, handler);
      }
      else if (ex instanceof ConversionNotSupportedException) {
         return handleConversionNotSupported(
               (ConversionNotSupportedException) ex, request, response, handler);
      }
      else if (ex instanceof TypeMismatchException) {
         return handleTypeMismatch(
               (TypeMismatchException) ex, request, response, handler);
      }
      else if (ex instanceof HttpMessageNotReadableException) {
         return handleHttpMessageNotReadable(
               (HttpMessageNotReadableException) ex, request, response, handler);
      }
      else if (ex instanceof HttpMessageNotWritableException) {
         return handleHttpMessageNotWritable(
               (HttpMessageNotWritableException) ex, request, response, handler);
      }
      else if (ex instanceof MethodArgumentNotValidException) {
         return handleMethodArgumentNotValidException(
               (MethodArgumentNotValidException) ex, request, response, handler);
      }
      else if (ex instanceof MissingServletRequestPartException) {
         return handleMissingServletRequestPartException(
               (MissingServletRequestPartException) ex, request, response, handler);
      }
      else if (ex instanceof BindException) {
         return handleBindException((BindException) ex, request, response, handler);
      }
      else if (ex instanceof NoHandlerFoundException) {
         return handleNoHandlerFoundException(
               (NoHandlerFoundException) ex, request, response, handler);
      }
      else if (ex instanceof AsyncRequestTimeoutException) {
         return handleAsyncRequestTimeoutException(
               (AsyncRequestTimeoutException) ex, request, response, handler);
      }
   }
   catch (Exception handlerEx) {
      if (logger.isWarnEnabled()) {
         logger.warn("Failure while trying to resolve exception [" + ex.getClass().getName() + "]", handlerEx);
      }
   }
   return null;
}
```

#### 5、自定义异常处理器

定义一个类、实现HandlerExceptionResolver接口、实现接口方法；

```java
@Order(value= Ordered.HIGHEST_PRECEDENCE)  //优先级，数字越小优先级越高
@Component
public class CustomerHandlerExceptionResolver implements HandlerExceptionResolver {
    @Override
    public ModelAndView resolveException(HttpServletRequest request,
                                         HttpServletResponse response,
                                         Object handler, Exception ex) {
        try {
            response.sendError(511,"我喜欢的错误");
        } catch (IOException e) {
            e.printStackTrace();
        }
        return new ModelAndView();
    }
}
```

同样、最终还是通过tomcat发送/error请求、由BasicErrorController进行处理；

需要通过@Order定义优先级、因为其他处理器先处理异常、则不符合我们的预期、因此可以调整Bean的优先级；



## JavaWeb原生组件使用与注入

### Servlet

```java
@WebServlet(name = "/MyServlet", urlPatterns = "/my")
public class MyServlet extends HttpServlet {
    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        System.out.println("业务处理");
    }

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        doPost(request, response);
    }
}
```

该Servlet用于接收/my请求；

### Filter

过滤器、用于过滤所有请求；

```java
@WebFilter(filterName = "MyFilter", urlPatterns = {"/*"})
public class MyFilter implements Filter {
    @Override
    public void destroy() {
        System.out.println("容器销毁");
    }

    @Override
    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain) throws ServletException, IOException {
        HttpServletRequest request = (HttpServletRequest) req;
        System.out.println("过滤请求、" + ((HttpServletRequest) req).getRequestURI());
        chain.doFilter(req, resp);
    }

    @Override
    public void init(FilterConfig config) throws ServletException {
        System.out.println("初始化过滤器");
    }

}
```

### Listener

用来监听容器初始化、销毁、以及监听会话过期、销毁等事件。

```java
@WebListener()
public class MyListener implements ServletContextListener,
        HttpSessionListener, HttpSessionAttributeListener {
    public MyListener() {
    }

    public void contextInitialized(ServletContextEvent sce) {
        System.out.println("容器初始化完成");
    }

    public void contextDestroyed(ServletContextEvent sce) {
        System.out.println("容器销毁");
    }

    public void sessionCreated(HttpSessionEvent se) {
        System.out.println("创建了Session会话");
    }

    public void sessionDestroyed(HttpSessionEvent se) {
    }
}
```

### 原生组件的注入

使用@ServletComponentScan("组件所在路径")、Spring启动会扫描这些组件、加载放入容器中；

```java
@ServletComponentScan("com.redis.zset")
public class WeiboApplication {

    public static void main(String[] args) {
        SpringApplication.run(WeiboApplication.class, args);
    }

}
```

#### 注意

常见的Servlet和DispatcherServlet是同级的、因为普通情况下、拦截器对于原生Servlet是无效的。

----



## 原生组件注入-【源码分析】DispatcherServlet注入原理

DispatcherServletAutoConfiguration配置类

```java
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass(DispatcherServlet.class)
@AutoConfigureAfter(ServletWebServerFactoryAutoConfiguration.class)
public class DispatcherServletAutoConfiguration {

   public static final String DEFAULT_DISPATCHER_SERVLET_BEAN_NAME = "dispatcherServlet";

   public static final String DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME = "dispatcherServletRegistration";

   @Configuration(proxyBeanMethods = false)
   @Conditional(DefaultDispatcherServletCondition.class)
   @ConditionalOnClass(ServletRegistration.class)
   @EnableConfigurationProperties(WebMvcProperties.class)
   protected static class DispatcherServletConfiguration {

      //注入DispatcherServlet实例到IOC容器
      @Bean(name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
      public DispatcherServlet dispatcherServlet(WebMvcProperties webMvcProperties) {
  		  //创建实例       
          DispatcherServlet dispatcherServlet = new DispatcherServlet();
	     return dispatcherServlet;
      }

      @Bean
      @ConditionalOnBean(MultipartResolver.class)
      @ConditionalOnMissingBean(name = DispatcherServlet.MULTIPART_RESOLVER_BEAN_NAME)
      public MultipartResolver multipartResolver(MultipartResolver resolver) {
         // Detect if the user has created a MultipartResolver but named it incorrectly
         return resolver;
      }

   }

   @Configuration(proxyBeanMethods = false)
   @Conditional(DispatcherServletRegistrationCondition.class)
   @ConditionalOnClass(ServletRegistration.class)
   @EnableConfigurationProperties(WebMvcProperties.class)
   @Import(DispatcherServletConfiguration.class)
   protected static class DispatcherServletRegistrationConfiguration {

      //创建DispatcherServlet注册Bean;
      @Bean(name = DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME)
      //存在DispatcherServlet实例、
       @ConditionalOnBean(value = DispatcherServlet.class, name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
      public DispatcherServletRegistrationBean dispatcherServletRegistration(DispatcherServlet dispatcherServlet,
            WebMvcProperties webMvcProperties, ObjectProvider<MultipartConfigElement> multipartConfig) {
          //DispatcherServletRegistrationBean包装Servlet、该Bean在Tomcat
          //过程中被调用，注入Tomcat的Servlet列表中；
          
          //DispatcherServletRegistrationBean ： 设置Servlet的处理路径path、Servlet实例；
         DispatcherServletRegistrationBean registration = new DispatcherServletRegistrationBean(dispatcherServlet,
               webMvcProperties.getServlet().getPath());
         
         //Servlet的名称
         registration.setName(DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
         
         //设置Servlet的加载顺序
         registration.setLoadOnStartup(webMvcProperties.getServlet().getLoadOnStartup());
 
          multipartConfig.ifAvailable(registration::setMultipartConfig);
         return registration;
      }}
}
```

DispatcherServletRegistrationBean :  什么时候调用、涉及到嵌入式tomcat的启动过程；



## 嵌入式Servlet容器-【源码分析】切换web服务器与定制化

### Tomcat启动

在SringBoot启动过程中，会创建并启动Tomcat;

```java
SpringApplication.run(WeiboApplication.class, args);

public ConfigurableApplicationContext run(String... args) {
   ConfigurableApplicationContext context = null;
   try {
  	  //创建IOC容器
      context = createApplicationContext();

      //刷新容器
      refreshContext(context);
   }
   catch (Throwable ex) {
   }
   return context;
}

//容器类型
public static final String DEFAULT_SERVLET_WEB_CONTEXT_CLASS = "org.springframework.boot."
      + "web.servlet.context.AnnotationConfigServletWebServerApplicationContext";

//反射创建IOC容器，容器类型为AnnotationConfigServletWebServerApplicationContext;
protected ConfigurableApplicationContext createApplicationContext() {
   Class<?> contextClass = this.applicationContextClass;
   switch (this.webApplicationType) {
      case SERVLET:
      contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
      break;
   }
   return (ConfigurableApplicationContext) BeanUtils.insantiateClass(contextClass);
}
```

![AnnotationConfigServletWebServerApplicationContext继承关系图](assets/\image-20221203142345459.png)

```java
//刷新容器
private void refreshContext(ConfigurableApplicationContext context) {
   refresh((ApplicationContext) context);
}

protected void refresh(ApplicationContext applicationContext) {
   refresh((ConfigurableApplicationContext) applicationContext);
}

protected void refresh(ConfigurableApplicationContext applicationContext) {
   applicationContext.refresh();
}
```

```java
//AnnotationConfigServletWebApplicationContext#refresh
@Override
public final void refresh() throws BeansException, IllegalStateException {
   try {
       //调用父类的refresh方法、即传统Spring IOC 的加载流程
      super.refresh();
   }
   catch (RuntimeException ex) {
       //启动失败则停止Tomcat
      WebServer webServer = this.webServer;
      if (webServer != null) {
         webServer.stop();
      }
      throw ex;
   }
}

@Override
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
      try {
          //...省略IOC加载流程
          
        // 扩展方法、由子类重写进行扩展； AnnotationConfigServletWebApplicationContext的父类
        //ServletWebApplicationContext中重写了该方法；
         onRefresh();
      } catch (BeansException ex) {
      }
   }
}
```

```java
//ServletWebApplicationContext#onRefresh
@Override
protected void onRefresh() {
   try {
   	 //创建并启动Tomcat容器
      createWebServer();
   } catch (Throwable ex) {
   }
}
```

```java
//ServletWebApplicationContext#createWebServer核心代码主干流程
private void createWebServer() {
   WebServer webServer = this.webServer;
   ServletContext servletContext = getServletContext();
   if (webServer == null && servletContext == null) {
      //1. 获取Web的服务器工厂；目的就是创建Web的服务器；默认返回TomcatServletWebServerFactory实例
      ServletWebServerFactory factory = getWebServerFactory();
       
      //2.1 getSelfInitializer 获取服务器的初始化器
      //2.2 获取并创建Web的服务器
      this.webServer = factory.getWebServer(getSelfInitializer());
      
       //3. 容器如果优化关闭、则会调用WebServerGracefulShutdownLifecycle实例
      getBeanFactory().registerSingleton("webServerGracefulShutdown",
            new WebServerGracefulShutdownLifecycle(this.webServer));
       //4. 容器如果停止、则调用WebServerStartStopLifecycle实例
      getBeanFactory().registerSingleton("webServerStartStop",
            new WebServerStartStopLifecycle(this, this.webServer));
   }
}
```

```java
//ServletWebApplicationContext#getWebServerFactory：从容器中获取Servlet的Web服务器工厂；
//ServletWebServerFactory：接口实现类有：TomcatServletWebServerFactory、JettyServletWebServerFactory、 UndertowServletWebServerFactory; 默认情况下使用TomcatServletWebServerFactory作为容器；
protected ServletWebServerFactory getWebServerFactory() {
   //从容器中获取类型为ServletWebServerFactory的Bean名称
   String[] beanNames = getBeanFactory().getBeanNamesForType(ServletWebServerFactory.class);
	//...
    
   //根据Bean名称从容器中获取beanNames的Bean实例
   return getBeanFactory().getBean(beanNames[0], ServletWebServerFactory.class);
}
```

```java
//TomcatServletWebServerFactory#getWebServer:创建Tomcat、设置Tomcat参数，并启动
//WebServer: 负责提供服务器的操作、启动、停止， 此处的WebServer为TomcatWebServer
@Override
public WebServer getWebServer(ServletContextInitializer... initializers) {
   if (this.disableMBeanRegistry) {
      Registry.disableRegistry();
   }
   //创建Tomcat实例
   Tomcat tomcat = new Tomcat();

   //...省略
    
   //准备TomcatEbContext; initailizers会被包装为TomcatStarter启动器、 设置进TomcatEmbeddedContext嵌入式Tomcat上下文中； Tomcat启动会调用该启动器；
   prepareContext(tomcat.getHost(), initializers);
    
   //将Tomcat包装为TomcatWebServer, 并启动Tomcat
   return getTomcatWebServer(tomcat);
}

protected TomcatWebServer getTomcatWebServer(Tomcat tomcat) {
   return new TomcatWebServer(tomcat, getPort() >= 0, getShutdown());
}
```



```java
//TomcatWebServer
public class TomcatWebServer implements WebServer {
	private final Tomcat tomcat;
	private final GracefulShutdown gracefulShutdown;

    public TomcatWebServer(Tomcat tomcat, boolean autoStart, Shutdown shutdown) {
       this.tomcat = tomcat;
       this.autoStart = autoStart;
       this.gracefulShutdown = (shutdown == Shutdown.GRACEFUL) ? new GracefulShutdown(tomcat) : null;
       initialize();
    }
}
//启动Tomcat
private void initialize() throws WebServerException {
   synchronized (this.monitor) {
      try 

         Context context = findContext();
         context.addLifecycleListener((event) -> {
            if (context.equals(event.getSource()) && Lifecycle.START_EVENT.equals(event.getType())) {
               // Remove service connectors so that protocol binding doesn't
               // happen when the service is started.
               removeServiceConnectors();
            }
         });

         // 启动Tomcat， 并触发所有监听器
         this.tomcat.start();

         // We can re-throw failure exception directly in the main thread
         rethrowDeferredStartupExceptions();

         try {
            ContextBindings.bindClassLoader(context, context.getNamingToken(), getClass().getClassLoader());
         }
         catch (NamingException ex) {
            // Naming is not enabled. Continue
         }

       	//不像Jetty, 所有的Tomcat都是镜像线程、我们需要创建一个非守护的阻塞线程阻塞Tomcat的立即关闭；
         startDaemonAwaitThread();
      } catch (Exception ex) {
      }
   }
}    

//创建一个等待线程、 让Tomcat一直处于运行状态、不停止；
private void startDaemonAwaitThread() {
   Thread awaitThread = new Thread("container-" + (containerCounter.get())) {
      @Override
      public void run() {
         TomcatWebServer.this.tomcat.getServer().await();
      }
   };
   awaitThread.setContextClassLoader(getClass().getClassLoader());
   awaitThread.setDaemon(false);
   awaitThread.start();
}
```

#### 总结

1. SpringBoot启动过程中，创建AnnotationConfigServletWebApplicationContext、该类继承于ServletWebApplicationContext、 
2. 在IOC容器加载过程中、调用onfresh方法、ServletWebApplicationContext实现了该扩展方法；
3. ServletWebApplicationContext会从IOC容器中获取类型ServletWebServerFactory的实例、默认为TomcayServletWebServerFactory;
4. 调用TomcatServletWebServerFactory#getWebServer中、创建Tomcat实例、 并将Tomcat包装为TomcatWebServer
5.  在TomcatWebServer构造方法中、 调用了Tomcat#start方法启动Tomcat服务器、 并创建一个阻塞的非守护线程、使Tomcat一起处于运行状态、避免Tomcat服务器停止运行；

###  Tomcat涉及的组件

- WebServer : 代表Web服务器

```java
public interface WebServer {
   void start() throws WebServerException;
   void stop() throws WebServerException;
   int getPort();
   default void shutDownGracefully(GracefulShutdownCallback callback) {
      callback.shutdownComplete(GracefulShutdownResult.IMMEDIATE);
   }
}
```

Web服务器默认支持：Tomcat、 Jetty、 UnderTow;

- ServletWebServerFactory: 代表生产Web服务器的工厂

```java
@FunctionalInterface
public interface ServletWebServerFactory {
   WebServer getWebServer(ServletContextInitializer... initializers);
}
```

​		生产Web服务器的工厂有：

1. TomcatServletWebServerFactory：负责生产Tomcat服务器
2. JettyServletWebServerFactory: 负责生产Jetty服务器
3. UndertowServletWebServerFactory: 负责生产Undertow服务器；



问题 ： 这三个类都没有@Component注解，那是谁负责注入容器的呢？

```java
/*
 * Copyright 2012-2020 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      https://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.springframework.boot.autoconfigure.web.servlet;

import java.util.stream.Collectors;

import javax.servlet.Servlet;

import io.undertow.Undertow;
import org.apache.catalina.startup.Tomcat;
import org.apache.coyote.UpgradeProtocol;
import org.eclipse.jetty.server.Server;
import org.eclipse.jetty.util.Loader;
import org.eclipse.jetty.webapp.WebAppContext;
import org.xnio.SslClientAuthMode;


@Configuration(proxyBeanMethods = false)
class ServletWebServerFactoryConfiguration {
	//存在Servlet, org.apache.catalina.startup.Tomcat, org.apache.coyote.UpgradeProtocol
    //类，则该类生效、本身是一个Bean；内部又创建TomcatServletWebServerFactory实例返回；
    //由于默认情况spring-boot-starter-web中依赖了tomcat依赖， 因此IOC启动过程中，会注入一个TomcatServletWebServerFactory
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass({ Servlet.class, Tomcat.class, UpgradeProtocol.class })
	@ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
	static class EmbeddedTomcat {
		@Bean
		TomcatServletWebServerFactory tomcatServletWebServerFactory(
				ObjectProvider<TomcatConnectorCustomizer> connectorCustomizers,
				ObjectProvider<TomcatContextCustomizer> contextCustomizers,
				ObjectProvider<TomcatProtocolHandlerCustomizer<?>> protocolHandlerCustomizers) {
			TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
			
			return factory;
		}
	}

	
    //Servlet，org.eclipse.jetty.server.Server，org.eclipse.jetty.util.Loader，org.eclipse.jetty.webapp.WebAppContext时、该Bean配置生效；
    //创建EmbeddedJetty实例、 内部创建了JettyServletWebServerFactory实例Bean
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass({ Servlet.class, Server.class, Loader.class, WebAppContext.class })
	@ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
	static class EmbeddedJetty {
		@Bean
		JettyServletWebServerFactory JettyServletWebServerFactory(
				ObjectProvider<JettyServerCustomizer> serverCustomizers) {
			JettyServletWebServerFactory factory = new JettyServletWebServerFactory();
			return factory;
		}

	}

    //Servlet，io.undertow.Undertow，org.eclipse.jetty.util.Loader，org.xnio.SslClientAuthMode时、该Bean配置生效；
    //创建EmbeddedUndertow实例、 内部创建了UndertowServletWebServerFactory, UndertowServletWebServerFactoryCustomizer类型的Bean实例；
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass({ Servlet.class, Undertow.class, SslClientAuthMode.class })
	@ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
	static class EmbeddedUndertow {

		@Bean
		UndertowServletWebServerFactory undertowServletWebServerFactory(
				ObjectProvider<UndertowDeploymentInfoCustomizer> deploymentInfoCustomizers,
				ObjectProvider<UndertowBuilderCustomizer> builderCustomizers) {
			UndertowServletWebServerFactory factory = new UndertowServletWebServerFactory();
			factory.getDeploymentInfoCustomizers()
					.addAll(deploymentInfoCustomizers.orderedStream().collect(Collectors.toList()));
			factory.getBuilderCustomizers().addAll(builderCustomizers.orderedStream().collect(Collectors.toList()));
			return factory;
		}

		@Bean
		UndertowServletWebServerFactoryCustomizer undertowServletWebServerFactoryCustomizer(
				ServerProperties serverProperties) {
			return new UndertowServletWebServerFactoryCustomizer(serverProperties);
		}
	}
}
```

ServletWebServerFactoryAutoConfiguration自动配置类中：通过@Import方式、导入了EmbeddedTomcat,EmbeddedJetty，EmbeddedUndertow三个类

```java
@Configuration(proxyBeanMethods = false)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@ConditionalOnClass(ServletRequest.class)
@ConditionalOnWebApplication(type = Type.SERVLET)
@EnableConfigurationProperties(ServerProperties.class)
@Import({ ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class,
      ServletWebServerFactoryConfiguration.EmbeddedTomcat.class,
      ServletWebServerFactoryConfiguration.EmbeddedJetty.class,
      ServletWebServerFactoryConfiguration.EmbeddedUndertow.class })
public class ServletWebServerFactoryAutoConfiguration {
    
}
```

### 切换Web服务器

只需要将spring-boot-starter-web中的tomcat依赖排除、再引入其他服务器的依赖、就可生效；

```java
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

### 定制Servlet容器

1.  实现WebServerFactoryCustomizer<ConfigurableServletWebServerFactory>
   - 把配置文件的值和`ServletWebServerFactory`进行绑定
2.  修改server.xxx的值；
3. 直接实现ServletWebServerFactory的子类ConfigurableServletWebServerFactory， 重新定制Servlet容器；

```java
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.boot.web.servlet.server.ConfigurableServletWebServerFactory;
import org.springframework.stereotype.Component;

@Component
public class CustomizationBean implements WebServerFactoryCustomizer<ConfigurableServletWebServerFactory> {

    @Override
    public void customize(ConfigurableServletWebServerFactory server) {
        server.setPort(9000);
    }

}
```

## 定制化原理-SpringBoot定制化组件的几种方式

1. 修改配置文件
2. 实现xxxcustomizer 自定义器
3. 使用@Configuration + @Bean方式 增加容器中默认组件
4. Web应用 编写一个配置类实现 `WebMvcConfigurer` 即可定制化web功能 + `@Bean`给容器中再扩展一些组件

```java
@Configuration
public class AdminWebConfig implements WebMvcConfigurer{
}
```

5.  @EnableWebMvc` + `WebMvcConfigurer` — `@Bean` 可以全面接管SpringMVC，所有规则全部自己重新配置； 实现定制和扩展功能

   - 原理：

     - WebMvcAutoConguration里、定义了Web运行所需要的大部分组件；

     - 生效条件之一为：容器中不存在WebMvcConfigurationSupport类型的Bean实例；

       ```java
       @ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
       public class WebMvcAutoConfiguration {
       }
       ```

     - 使用@EnableWebMvc注解、会@Import(DelegatingWebMvcConfiguration.class)

       ```java
       @Import(DelegatingWebMvcConfiguration.class)
       public @interface EnableWebMvc {
       }
       ```

     - 因此使用@EnableWebMvc注解、就代表着WebMvcAutoConfigruation失效， 而@EnableWebMvc只会保证基本使用、因此我们需要重新实现定义很多基础组件；

### 引入Starter一般分析

引入Starter => 包含多个autoConguration自动配置类 => 绑定xxxProperties配置 => 绑定配置文件项yml