# SpringMVC源码分析



来看一下SpringBoot针对WEB应用程序的源码，简单想一下设计一个Spring的WEB应用程序，都会有哪些功能？

1. 根据请求地址匹配到JAVA方法
2. 获取到请求的参数
3. 返回请求参数



Notes : SpringBoot是集成了很多框架使用，为开发者提供简单遍历的一个框架集合，其中就包括了SpringMVC(SpringBoot的WEB请求处理依旧是SpringMVC中的DispatcherServlet类来进行处理)。



首先我们先来看一下DispatcherServlet的继承关系，DispatcherServlet的父类FrameworkServlet继承与HttpServerlet

```java
public class DispatcherServlet extends FrameworkServlet
```



![image-20211011201020714](D:\Blog\My-Blog\pic\image-20211011201020714.png)











继承于HttpServerlet一定要重写doGet(), doPost()等方法，我们来搜索一下，发现HttpServerletBean中并没有重写这些方法，我们再来FrameworkServlet查看在此类中重写了这些方法，且这些方法都调用了FrameworkServlet中的 processRequest(request, response)。

```java
protected final void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        this.processRequest(request, response);
    }

    protected final void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        this.processRequest(request, response);
    }

    protected final void doPut(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        this.processRequest(request, response);
    }

    protected final void doDelete(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException 	{
        this.processRequest(request, response);
    }


```

查看processRequest方法中又是调用doService方法去执行一个HTTP请求

```java
protected final void processRequest(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
    //这些都是初始化获取过程    
    long startTime = System.currentTimeMillis();
        Throwable failureCause = null;
        LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
        LocaleContext localeContext = this.buildLocaleContext(request);
        RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
        ServletRequestAttributes requestAttributes = this.buildRequestAttributes(request, response, previousAttributes);
        WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
        asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new FrameworkServlet.RequestBindingInterceptor());
        this.initContextHolders(request, localeContext, requestAttributes);

        try {
            // doService(request, response) 这个才是核心,就是去执行一个HTTP请求
            this.doService(request, response);
        } catch (IOException | ServletException var16) {
            failureCause = var16;
            throw var16;
        } catch (Throwable var17) {
            failureCause = var17;
            throw new NestedServletException("Request processing failed", var17);
        } finally {
            this.resetContextHolders(request, previousLocaleContext, previousAttributes);
            if (requestAttributes != null) {
                requestAttributes.requestCompleted();
            }

            this.logResult(request, response, (Throwable)failureCause, asyncManager);
            this.publishRequestHandledEvent(request, response, startTime, (Throwable)failureCause);
        }
    }
```

紧跟着去看一下doService方法，发现这是一个抽象方法，所以饶了一圈,还是绕道了DispatcherServlet中的doService方法

```java
// 查看到DispatcherServlet中的DoService方法后看到核心还是使用doDispatch方法去做Http请求的派发
try {        
    this.doDispatch(request, response);        
	}
	finally {
        if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted() && attributesSnapshot != null) {
            this.restoreAttributesAfterInclude(request, attributesSnapshot);
        }
        if (this.parseRequestPath) {
            ServletRequestPathUtils.setParsedRequestPath(previousRequestPath, request);
        }
    }

```



综上所述DispatcherServlet中的请求处理原理的核心就是doDispatch方法

![image-20210919192208909](D:\Blog\My-Blog\pic\image-20210919192208909.png)



最终我们得到SpringMVC HTTP请求处理都从org.springframework.web.servlet.DispatcherServlet中 --> **doDispatch()**进行处理，那么我们就逐步分析源码去求解之前的三个问题。



### 1.	请求映射原理



首先我们先写一个最简单的WEB接口，进行一个HTTP请求 http://localhost:8080/test

```java
@RequestMapping("/test")
public String test(){
    return "test";
}
```



我们来打个断点来进行逐行分析一下doDispatch方法，

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
  	// 解析我们的Request请求, 包含我们的请求路径,端口,HTTP方法等	
   	HttpServletRequest processedRequest = request;
        
    HandlerExecutionChain mappedHandler = null;
    //	是否需要上传功能,默认为false    
    boolean multipartRequestParsed = false;
    //	是否有异步处理请求    
    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

        try {
            try {
                //	初始化一些数据
                ModelAndView mv = null;
                Object dispatchException = null;

                try {
                  	//	判断需要上传功能
                    processedRequest = this.checkMultipart(request);
                    
                    multipartRequestParsed = processedRequest != request;
                    //	核心:决定哪个Controller可以来处理/test这个请求
                    //	放行之后 com.springbootstudy.springbootstudy.controller.test#Hello()
                    //	发现获得到Hello()方法来处理/test这个请求,和处理类进行半丁映射
                    mappedHandler = this.getHandler(processedRequest);
               ...

```

根据上述源码发现 **mappedHandler**的handler属性包含到了com.springbootstudy.springbootstudy.controller.test#test()，它将我们的请求处理和处理方法进行绑定映射(即我们处理/test请求的方法)。那就进一步来看**this.getHandler(processedRequest)**方法

```java
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    if (this.handlerMappings != null) {
        // handlerMappings包含了Spring中所有的请求种类
 		//	RequestMappingHandlerMapping,BeanNameUrlHandlerMapping,RouterFunctionMapping, 
        //	SimpleUrlHandlerMapping, WelcomePagHandlerMapping
        Iterator var2 = this.handlerMappings.iterator();		
        //	遍历所有的HandlerMapping,        
        while(var2.hasNext()) {                             
            //	RequestMappingHandlerMapping 中的 mappingRegistry属性,            
            //	保存着所有Controller中 经过@RequestMapping注解的方法和秦桧去路径            
            HandlerMapping mapping = (HandlerMapping)var2.next();            
            HandlerExecutionChain handler = mapping.getHandler(request);           
            if (handler != null) {           
                return handler;                
            }            
        }        
    }   
    return null;   
}


```

执行发现getHandler方是调用getHandlerInternal方法

```java
protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {        
    //	获取请求路径 /test
    String lookupPath = this.initLookupPath(request);   
    this.mappingRegistry.acquireReadLock();        
    HandlerMethod var4;    
    try {    
        //	核心方法，根据请求路径匹配Controller中的方法
        HandlerMethod handlerMethod = this.lookupHandlerMethod(lookupPath, request);        
        //	将与路径匹配的方法与其他需要的属性封装成Bean返回
        var4 = handlerMethod != null ? handlerMethod.createWithResolvedBean() : null;        
    } finally { 
        this.mappingRegistry.releaseReadLock();        
    }    
    return var4;    
}
```



根据上述源码我们得到lookupHandlerMethod方法是将请求路径与Controller中的方法进行映射，来看一下lookupHandlerMethod源码

```java
// lookupPath请求路径 /test
protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {      
    List<AbstractHandlerMethodMapping<T>.Match> matches = new ArrayList();
    //	获得和路径匹配的所有方法   
    //	本案例中只有一个test方法	
    List<T> directPathMatches = this.mappingRegistry.getMappingsByDirectPath(lookupPath);    
    if (directPathMatches != null) {            
       	// 根据请求方式将路径和请求方法的映射塞进matches中    
        this.addMatchingMappings(directPathMatches, matches, request);        
    }
    if (matches.isEmpty()) {      
        this.addMatchingMappings(this.mappingRegistry.getRegistrations().keySet(), matches, request)        
    }      
    if (matches.isEmpty()) {    
        return this.handleNoMatch(this.mappingRegistry.getRegistrations().keySet(), lookupPath, request);   
    } else {                            
        AbstractHandlerMethodMapping<T>.Match bestMatch = (AbstractHandlerMethodMapping.Match)matches.get(0)            
        //	对多请求映射进行处理            
        if (matches.size() > 1) {
            // 如果matches中元素大于1,即一个路径有多个请求方法,例如GET, POST,则根据请求方法进行排序,选择最符合的请求方法 
            Comparator<AbstractHandlerMethodMapping<T>.Match> comparator = new AbstractHandlerMethodMapping.MatchComparator(this.getMappingComparator(request));                
            matches.sort(comparator);               
            bestMatch = (AbstractHandlerMethodMapping.Match)matches.get(0);             
            if (this.logger.isTraceEnabled()) {             
                this.logger.trace(matches.size() + " matching mappings: " + matches);                
            }                
            if (CorsUtils.isPreFlightRequest(request)) {                  
                Iterator var7 = matches.iterator();                  
                while(var7.hasNext()) {                  
                    AbstractHandlerMethodMapping<T>.Match match = (AbstractHandlerMethodMapping.Match)var7.next()               
                    if (match.hasCorsConfig()) {                         
                       return PREFLIGHT_AMBIGUOUS_MATCH;   
                    }  
                }  
            } 
            else {       
                AbstractHandlerMethodMapping<T>.Match secondBestMatch = (AbstractHandlerMethodMapping.Match)matches.get(1)          			
                //	如果存在多个匹配结果，就报错

                if (comparator.compare(bestMatch, secondBestMatch) == 0) {                
                    Method m1 = bestMatch.getHandlerMethod().getMethod();                  
                    Method m2 = secondBestMatch.getHandlerMethod().getMethod();         
                    String uri = request.getRequestURI();             
                    throw new IllegalStateException("Ambiguous handler methods mapped for '" + uri + "': {" + m1 + ", " + m2 + "}");       
                }       
            }      
        }           
        request.setAttribute(BEST_MATCHING_HANDLER_ATTRIBUTE, bestMatch.getHandlerMethod());           
        this.handleMatch(bestMatch.mapping, lookupPath, request);       
        return bestMatch.getHandlerMethod();    
    }    
}
```



来进行一个简单的总结

* DispatcherServlet继承于HttpServlet, 请求处理的最重要核心是doDispatch()
* HandlerMapping中保存了Spring中所有的请求种类
  * 请求进来会遍历HandlerMapping看是否有处理当前请求的handler
  * 如果没有就进入下一个handler(通过@RequestMapping标注的Controller会保存在RequestMappingHandlerMapping)
* 如果一个请求路径对应多个Controller, 会根据请求类型进行排序选取最优的Controller





### 2.	请求参数注解原理WEB开发常用注解参数解析器

#### 2.1	常用请求参数注解

1. @PathVariable
   * @RequestMapping("/test/``{id}``"), 通过 请求http:...../test/``id``获取变量
2. @RequestParam
   * @RequestParam(value = "age") Integer age, 通过http:...../test?``age=232``获取
3. @RequestBody
   * @RequestBody Map<String, String> grades，通过POST 请求传入JSON数据来获取
   * 实际开发中更多是使用一个实体类来承接RequestBody 



我们通过写一个简单的Controller来看一下这三个注解。

```java
@Controller
@ResponseBody
public class test {
    @RequestMapping("/test/{id}")
    public Map Hello(@PathVariable("id") Integer id,
                     @RequestParam(value = "age") Integer age,
                     @RequestBody Map<String, String> grades){
        Map<String, Object> res = new HashMap<>();
        res.put("ID", id);
        res.put("Age", age);
        res.put("Grades", grades);
        return res;
    }
}
```

使用IDEA下的HTTP Client发送一个如下请求

```http
###
POST http://localhost:8080/test/3?age=32
Content-Type: application/json

{
  "math": 99,
  "Chinese": 100
}

```

获得如下结果，可以看到参数都能正常获取

```http
HTTP/1.1 200 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Tue, 21 Sep 2021 05:59:33 GMT
Keep-Alive: timeout=60
Connection: keep-alive
{
  "Grades": {
    "math": "99",
    "Chinese": "100"
  },
  "ID": 3,
  "Age": 32
}
```



#### 2.2	参数解析器原理

下面来详细看一下，SpringMVC是如何获取到参数的，回想一下上一章，SpringMVC执行HTTP请求映射原理的核心方法放在了``doDispatch()``中，这节的请求参数原理的核心方法依旧放在``doDispatch()``中



之前已经说过在**mappedHandler = this.getHandler(processedRequest);**，mappedHandler获取到请求路径和Controller方法中的映射。

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
        HttpServletRequest processedRequest = request;
        HandlerExecutionChain mappedHandler = null;
        boolean multipartRequestParsed = false;
        WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
		// 上一章讲到的，自动请求参数处理
        try {
            try {
                ModelAndView mv = null;
                Object dispatchException = null;

                try {
                    processedRequest = this.checkMultipart(request);
                    multipartRequestParsed = processedRequest != request;
                    //	getHandler获取到请求路径和Controller方法中的映射
                    mappedHandler = this.getHandler(processedRequest);
                    if (mappedHandler == null) {
                        this.noHandlerFound(processedRequest, response);
                        return;
                    }
					//	HandlerAdapter字面上的意思就是处理适配器
                    //	将断点打到这个地方发现mappedHandler中仍未有我们的参数信息
                    HandlerAdapter ha = this.getHandlerAdapter(mappedHandler.getHandler());
         			...
                }
            }
        }
}
//
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
    // this.handlerAdapters 包含了所有请求种类的处理器 
    //	0 = {RequestMappingHandlerAdapter@6382} 
	//	2 = {HttpRequestHandlerAdapter@6384} 
	//	1 = {HandlerFunctionAdapter@6383} 
	//	3 = {SimpleControllerHandlerAdapter@6385}   
    if (this.handlerAdapters != null) {
        	//	增强遍历
            Iterator var2 = this.handlerAdapters.iterator();

            while(var2.hasNext()) {
                HandlerAdapter adapter = (HandlerAdapter)var2.next();
                //	核心， 判断SpringMVC中的Handler是否支持我们当前请求的 handler
                if (adapter.supports(handler)) {
                    return adapter;
                }
            }
        }

        throw new ServletException("No adapter for handler [" + handler + "]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
}



public final boolean supports(Object handler) {
    // 使用@RequestMapping()注解的handler会都会返回True
    return handler instanceof HandlerMethod && this.supportsInternal((HandlerMethod)handler);
}   
```

执行完 HandlerAdapter ha = this.getHandlerAdapter(mappedHandler.getHandler());发现HandlerAdapter已经初始化完成（即判断出我们当前的请求的路径以及Controller符合SpringMVC的支持）。继续向下看

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
       				...
                    //	获取到适配器
                    HandlerAdapter ha = this.getHandlerAdapter(mappedHandler.getHandler());
                    //	匹配HTTP请求方法
    				String method = request.getMethod();
                    boolean isGet = HttpMethod.GET.matches(method);
                    if (isGet || HttpMethod.HEAD.matches(method)) {
                        long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                        if ((new ServletWebRequest(request, response)).checkNotModified(lastModified) && isGet) {
                            return;
                        }
                    }

                    if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                        return;
                    }
					//	执行目标方法
    				//	actually invoke the handler
                    mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
                    if (asyncManager.isConcurrentHandlingStarted()) {
                        return;
                    }

                    this.applyDefaultViewName(processedRequest, mv);
                    ...
                }
            }
        }
}


public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
    return this.handleInternal(request, response, (HandlerMethod)handler);
}


protected ModelAndView handleInternal(HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod) 		throws Exception {
    	// 检查是否支持当前HTTP方法
        this.checkRequest(request);
        ModelAndView mav;
    	//	如果需要保证用户能够在多次请求中正确的访问同一个 session ，就要将 synchronizeOnSession 设置为 TRUE 。
    	//	默认为False
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
            //	执行Handle方法
            mav = this.invokeHandlerMethod(request, response, handlerMethod);
        }

        if (!response.containsHeader("Cache-Control")) {
            if (this.getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
                this.applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
            } else {
                this.prepareResponse(response);
            }
        }

        return mav;
    }

protected ModelAndView invokeHandlerMethod(HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
    //	初始化过程    
    ServletWebRequest webRequest = new ServletWebRequest(request, response);   
    Object result;
    try {
        WebDataBinderFactory binderFactory = this.getDataBinderFactory(handlerMethod);
        ModelFactory modelFactory = this.getModelFactory(handlerMethod, binderFactory);
        ServletInvocableHandlerMethod invocableMethod = this.createInvocableHandlerMethod(handlerMethod);
        //	神奇的发现argumentResolvers有27个解析器
        //	0 = {RequestParamMethodArgumentResolver@6872} 
		//	1 = {RequestParamMapMethodArgumentResolver@6873} 
		//	2 = {PathVariableMethodArgumentResolver@6874} 
		//	3 = {PathVariableMapMethodArgumentResolver@6875}
        //	...
        if (this.argumentResolvers != null) {
            invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
        }
        //	同理是返回值处理器
        if (this.returnValueHandlers != null) {
            invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
        }
        //	设定一些值
        ...
        //执行参数	
        invocableMethod.invokeAndHandle(webRequest, mavContainer, new Object[0]);

    }
   
}


// 经过以上代码，发现SpringMVC中的定义的参数都有参数解析器去匹配，继续打开参数解析器的源码
public interface HandlerMethodArgumentResolver {
    //	是否支持参数
    boolean supportsParameter(MethodParameter var1);
	//	解析参数
    @Nullable
    Object resolveArgument(MethodParameter var1, @Nullable ModelAndViewContainer var2, NativeWebRequest var3, @Nullable WebDataBinderFactory var4) throws Exception;
}


public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
    // 真正去执行目标方法    
    Object returnValue = this.invokeForRequest(webRequest, mavContainer, providedArgs);
    ...
        }
    }

    @Nullable
public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
    //	获取方法的所有值    
    Object[] args = this.getMethodArgumentValues(request, mavContainer, providedArgs);
        if (logger.isTraceEnabled()) {
            logger.trace("Arguments: " + Arrays.toString(args));
        }
		//	利用反射执行方法
        return this.doInvoke(args);
    }

```



最终我们发现，SpringMVC获取请求参数值是通过getMethodArgumentValues()获取值的

```java
======
protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
    	
    //	获取方法的全部参数声明(索引位置，类型等)
    MethodParameter[] parameters = this.getMethodParameters();
        if (ObjectUtils.isEmpty(parameters)) {
            return EMPTY_ARGS;
        } else {
            Object[] args = new Object[parameters.length];

            for(int i = 0; i < parameters.length; ++i) {
                MethodParameter parameter = parameters[i];
                parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
                args[i] = findProvidedArgument(parameter, providedArgs);
                if (args[i] == null) {
                    //	使用IDEA的Evaluate得到  this.resolvers = argumentResolvers
                    // 	很神奇的这是我们之前看到的HandlerMethodArgumentResolver中的supportsParameter方法
                    if (!this.resolvers.supportsParameter(parameter)) {
                        throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
                    }
					
                    try {
                        //	使用IDEA的Evaluate得到  this.resolvers = argumentResolvers
                    	// 	很神奇的这是我们之前看到的HandlerMethodArgumentResolver中的resolveArgument方法
                        args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
                    } catch (Exception var10) {
                        if (logger.isDebugEnabled()) {
                            String exMsg = var10.getMessage();
                            if (exMsg != null && !exMsg.contains(parameter.getExecutable().toGenericString())) {
                                logger.debug(formatArgumentError(parameter, exMsg));
                            }
                        }

                        throw var10;
                    }
                }
            }

            return args;
        }
    }

//	判断是否之前当前Param
private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
    HandlerMethodArgumentResolver result = (HandlerMethodArgumentResolver)this.argumentResolverCache.get(parameter);
    if (result == null) {
        Iterator var3 = this.argumentResolvers.iterator();
		//	遍历所有的argumentResolvers，找到支持当前参数的argumentResolver
        while(var3.hasNext()) {
            
            HandlerMethodArgumentResolver resolver = (HandlerMethodArgumentResolver)var3.next();
            if (resolver.supportsParameter(parameter)) {
                result = resolver;
                //	注意当我们判断一次参数正确之后吗，会将该参数以及参数参数处理器放到缓存中，以便于我们下次使用
                //	这就代表一个参数在SpringMVC中只需要判断一些词即可
                this.argumentResolverCache.put(parameter, resolver);
                break;
            }
        }
    }

    return result;
}


@Nullable
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
    
    //	获取到支持改参数的参数解析器
    HandlerMethodArgumentResolver resolver = this.getArgumentResolver(parameter);
    if (resolver == null) {
        throw new IllegalArgumentException("Unsupported parameter type [" + parameter.getParameterType().getName() + "]. supportsParameter should be called first.");
    } else {
        //	解析参数
        return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
    }
}

public final Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
    
    //	获取参数信息
    AbstractNamedValueMethodArgumentResolver.NamedValueInfo namedValueInfo = this.getNamedValueInfo(parameter);
    //	获取参数的声明    
    MethodParameter nestedParameter = parameter.nestedIfOptional();
    //	获取参数的名字 即 @PathVariable("id") 中的id    
    Object resolvedName = this.resolveEmbeddedValuesAndExpressions(namedValueInfo.name);
        
    if (resolvedName == null) {
            throw new IllegalArgumentException("Specified name must not resolve to null: [" + namedValueInfo.name + "]");
        } 
    else {
        //	获取到参数值
        Object arg = this.resolveName(resolvedName.toString(), nestedParameter, webRequest);    
       	...
        return arg;
    }
}

//	根据支持参数的参数解析器去获取参数

//	@PathVariable, resolveName直接从URL中进行解析，获取参数
protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception {
    Map<String, String> uriTemplateVars = (Map)request.getAttribute(HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE, 0);
    return uriTemplateVars != null ? uriTemplateVars.get(name) : null;
}

//	@RequestParam,	 resolveName直接从原生 HTTP request.getParameterValues(name);获取参数
 protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception {
        HttpServletRequest servletRequest = (HttpServletRequest)request.getNativeRequest(HttpServletRequest.class);
        Object arg;
        if (servletRequest != null) {
            arg = MultipartResolutionDelegate.resolveMultipartArgument(name, parameter, servletRequest);
            if (arg != MultipartResolutionDelegate.UNRESOLVABLE) {
                return arg;
            }
        }

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



#### 2.2	自定义参数绑定原理

上面已经分析了两种简单注解的参数获取原理，但是实际在开发过程中最经常使用的还是定义一个实体类，并且使用实体类来接受请求的JSON参数，那就需要使用到@RequestBody这个注解，先简单来看一段代码

```java
//	定义一个实体类来接受请求参数
@Data
public class Person {
    private String name;

    private int age;

}

//	原地返回请求参数
@PostMapping("/person")
public Person testPerson(@RequestBody Person person){
    return person;
}
```

发出Post请求

```http
###

POST http://localhost:8080/person
Content-Type: application/json
{
  "name": "wang",
  "age": 100
}

```

请求结果如下

```http
{
    "name": "wang",
    "age": 100
}
```

那就来看一下SpringMVC是如何处理用户自定义的实体类来接收参数，上节我们说过SpringMVC是通过getMethodArgumentValues()去获取参数，那就回到这个方法继续去看

```java
protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer, Object... 			providedArgs) throws Exception {
	···
    // 判断是否resolvers是否支持当前参数
    if (!this.resolvers.supportsParameter(parameter)) {
                        throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
    }
	
    try {
        args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
               ···
        }
}    
public boolean supportsParameter(MethodParameter parameter) {
	// 判断是否有@RequestParam
    if (parameter.hasParameterAnnotation(RequestParam.class)) {
            if (!Map.class.isAssignableFrom(parameter.nestedIfOptional().getNestedParameterType())) {
                return true;
            } else {
                RequestParam requestParam = (RequestParam)parameter.getParameterAnnotation(RequestParam.class);
                return requestParam != null && StringUtils.hasText(requestParam.name());
            }
        } 
    // 判断是否有@RequestPartm
    else if (parameter.hasParameterAnnotation(RequestPart.class)) {
            return false;
        } 
    else {    
        parameter = parameter.nestedIfOptional();
        // 判断是否有上传
            
        if (MultipartResolutionDelegate.isMultipartArgument(parameter)) {
         
            return true;      
        } else {
            //	判断当前类是否是isSimpleProperty  
            return this.useDefaultResolution ? BeanUtils.isSimpleProperty(parameter.getNestedParameterType()) : false;
        }    
    }    
}


```



根据上一节所讲进入到getArgumentResolver()看是哪一种HandlerResolver来处理当前参数。把断电打在``return result;``上，发现``resolver``会去处理当前参数。

```java
private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
    HandlerMethodArgumentResolver result = (HandlerMethodArgumentResolver)this.argumentResolverCache.get(parameter);
    if (result == null) {
        Iterator var3 = this.argumentResolvers.iterator();
        while(var3.hasNext()) {
            
            HandlerMethodArgumentResolver resolver = (HandlerMethodArgumentResolver)var3.next();
            //	判断是否支持当前参数处理器
            if (resolver.supportsParameter(parameter)) {
                result = resolver;
            
                this.argumentResolverCache.put(parameter, resolver);
                break;
            }
        }
    }
	
    return result;
}

// 	RequestResponseBodyMethodProcessor
//	判断是否有RequestBody注解
public boolean supportsParameter(MethodParameter parameter) {
    return parameter.hasParameterAnnotation(RequestBody.class);
}

//	处理参数
@Nullable
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
    //	拿到参数解析器
    HandlerMethodArgumentResolver resolver = this.getArgumentResolver(parameter);
    if (resolver == null) {
        throw new IllegalArgumentException("Unsupported parameter type [" + parameter.getParameterType().getName() + "]. supportsParameter should be called first.");
    } else {
        //	处理参数
        return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
    }
}

public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
    //	获取参数
    parameter = parameter.nestedIfOptional();
    //	参数消息转换器，即从HTTP POST请求中获取参数{"name": "wang","age":13}
    // 	A base class for resolving method argument values by reading from the body of a request with HttpMessageConverters.
    Object arg = this.readWithMessageConverters(webRequest, parameter, parameter.getNestedGenericParameterType());
    //	获取参数命名 Person
    String name = Conventions.getVariableNameForParameter(parameter);
    if (binderFactory != null) {
        //	核心关键：绑定Servelet和请求的实体对象
        WebDataBinder binder = binderFactory.createBinder(webRequest, arg, name);
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

    return this.adaptArgumentIfNecessary(arg, parameter);
}

protected <T> Object readWithMessageConverters(NativeWebRequest webRequest, MethodParameter parameter, Type paramType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {
    // get HTTP Servlet Requests
    HttpServletRequest servletRequest = (HttpServletRequest)webRequest.getNativeRequest(HttpServletRequest.class);
    Assert.state(servletRequest != null, "No HttpServletRequest");
    //	将servletRequest封装成ServletServerHttpRequest
    ServletServerHttpRequest inputMessage = new ServletServerHttpRequest(servletRequest);
    Object arg = this.readWithMessageConverters(inputMessage, parameter, paramType);
    if (arg == null && this.checkRequired(parameter)) {
        throw new HttpMessageNotReadableException("Required request body is missing: " + parameter.getExecutable().toGenericString(), inputMessage);
    } else {
        return arg;
    }
}


//	将POST请求的JSON对象转会Java中的实体对象
@Nullable
protected <T> Object readWithMessageConverters(HttpInputMessage inputMessage, MethodParameter parameter, Type targetType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {
		
    ...
	//	获取Controller类
    Class<?> contextClass = parameter.getContainingClass();
    //	获取Java中的实体类
    Class<T> targetClass = targetType instanceof Class ? (Class)targetType : null;
    ...
	//	获取请求方法
    HttpMethod httpMethod = inputMessage instanceof HttpRequest ? ((HttpRequest)inputMessage).getMethod() : null;
    Object body = NO_VALUE;
    
    AbstractMessageConverterMethodArgumentResolver.EmptyBodyCheckingHttpInputMessage message;
        try {
            label98: {
               	//	获取请求信息的字节流
                message = new AbstractMessageConverterMethodArgumentResolver.EmptyBodyCheckingHttpInputMessage(inputMessage);
                //	遍历messageConverters
                Iterator var11 = this.messageConverters.iterator();

                HttpMessageConverter converter;
                Class converterType;
                GenericHttpMessageConverter genericConverter;
                while(true) {
                    if (!var11.hasNext()) {
                        break label98;
                    }

                    
                    converter = (HttpMessageConverter)var11.next();
                    converterType = converter.getClass();
                    //	找到可以解析当前请求信息的messageConverters， 此刻是MappingJackonMeessageConverter
                    genericConverter = converter instanceof GenericHttpMessageConverter ? (GenericHttpMessageConverter)converter : null;
				···
                if (message.hasBody()) {
                    HttpInputMessage msgToUse = this.getAdvice().beforeBodyRead(message, parameter, targetType, converterType);
                    //	根据Jackson去解析请求的JSON Data字节流
                    body = genericConverter != null ? genericConverter.read(targetType, contextClass, msgToUse) : converter.read(targetClass, msgToUse);
                 	//	Allows customizing the request before its body is read and converted into an Object and also allows for 				//	processing of the resulting Object before it is passed into a controller method as an @RequestBody or an 					//	HttpEntity method argument.
				//	Implementations of this contract may be registered directly with the RequestMappingHandlerAdapter or more 					//	likely annotated with @ControllerAdvice in which case they are auto-detected.
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
    } else if (httpMethod == null || !SUPPORTED_METHODS.contains(httpMethod) || noContentType && !message.hasBody()) {
        return null;
    } else {
        throw new HttpMediaTypeNotSupportedException(contentType, this.getSupportedMediaTypes(targetClass != null ? targetClass : Object.class));
    }
}
```



![image-20210922205715525](D:\Blog\My-Blog\pic\image-20210922205715525.png)



#### 2.4总结

1. @PathVariable,@RequestParam
   1. 通过 getMethodArgumentValues()针对不同的注解使用不同的resolver
      * @PathVariable从URL直接获取参数
      * @RequestParam从request.getParameterValues(name)获取参数
2. @RequestBody
   1. 通过从HTTP请求中获取请求Body的字节流
   2. 然后使用不同的GenericHttpMessageConverter去获取参数



### 3.	返回值原理

上一篇文章我们讲到了SpringMVC如何获取HTTP请求参数，接下来看一下SpringMVC如何处理返回值。

写一个简单的接口即，请求参数是一个Person对象，返回参数依旧是一个Person对象。来Debug进行源码分析。

```java
@PostMapping("/person")
public Person testPerson(@RequestBody Person person){
    return person;
}
```



记得上一章讲到SpringMVC获取请求参数时，在方法**invokeHandlerMethod()**中由参数处理器(argumentResolvers)和返回值处理器)(returnValueHandlers)，

接下来看一下返回值处理器。

```java
 protected ModelAndView invokeHandlerMethod(HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
        ServletWebRequest webRequest = new ServletWebRequest(request, response);
		//	省略代码	
     	...     
         if (this.argumentResolvers != null) {
             invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
         }
     	//
     	if (this.returnValueHandlers != null) {
         invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
	     }
     	//	省略代码	
     	...
            
        //	执行HTTP方法
     	invocableMethod.invokeAndHandle(webRequest, mavContainer, new Object[0]);

     	//	省略代码	
     	...
    }
```

如下图所示**returnValueHandlers**中共有15个值，包括了SpringMVC所有的可以处理的return value类型。

![image-20210926204254572](D:\Blog\My-Blog\pic\image-20210926204254572.png)







```java
 public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
     	
     
     //	获取参数后使用反射执行方法，获取返回值
     Object returnValue = this.invokeForRequest(webRequest, mavContainer, providedArgs);
      
     
     //	省略代码	
     ...     
    //	核心 ： 处理返回值
    try {
            
        this.returnValueHandlers.handleReturnValue(returnValue, this.getReturnValueType(returnValue), mavContainer, webRequest);
        } 
     catch (Exception var6) {
            if (logger.isTraceEnabled()) {
                logger.trace(this.formatErrorForReturnValue(returnValue), var6);
      //省略代码	
     ...     
    }
```



handleReturnValue()就是处理return value。

```java
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType, ModelAndViewContainer mavContainer, 		NativeWebRequest webRequest) throws Exception {
    
    //	选择Handler
    HandlerMethodReturnValueHandler handler = this.selectHandler(returnValue, returnType);
    
    if (handler == null) {
    
        throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
        
    } else {
    	//	处理返回值
        handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
        
    }
    
}
```



```java
 private HandlerMethodReturnValueHandler selectHandler(@Nullable Object value, MethodParameter returnType) {
     //	判断是否有异步请求   
     boolean isAsyncValue = this.isAsyncReturnValue(value, returnType);
     //	将之前拿到的returnValueHandlers生成迭代器
     Iterator var4 = this.returnValueHandlers.iterator();

       
     HandlerMethodReturnValueHandler handler;
     //	遍历，住处支持当前 return value的handler
     do {
         do {
             if (!var4.hasNext()) {
                 return null;
             }
             handler = (HandlerMethodReturnValueHandler)var4.next();
            } while(isAsyncValue && !(handler instanceof AsyncHandlerMethodReturnValueHandler));
     
     }
     //	找出支持当前returnType的Handler，退出循环
     while(!handler.supportsReturnType(returnType));   
     return handler;
    }

//	遍历方法结束，支持当前returnType的是 RequestResponseBodyMethodProcessor
//	当我们的Controller中的方法，使用了@ResponseBody注解即被RequestResponseBodyMethodProcessor处理。
public boolean supportsReturnType(MethodParameter returnType) {
        return AnnotatedElementUtils.hasAnnotation(returnType.getContainingClass(), ResponseBody.class) || returnType.hasMethodAnnotation(ResponseBody.class);    
}
```

此刻SpringMVC已经获取到可以处理当前请求返回值的处理器，接下来看一下是如何处理返回值即**handleReturnValue()**方法。

```java
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType, ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {
    mavContainer.setRequestHandled(true);
    //	获取HTTP请求信息
    ServletServerHttpRequest inputMessage = this.createInputMessage(webRequest);
    //	获取HTTP返回信息
    ServletServerHttpResponse outputMessage = this.createOutputMessage(webRequest);
    //	核心： 使用消息转换器处理返回值
    this.writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
}
```



```java
protected <T> void writeWithMessageConverters(@Nullable T value, MethodParameter returnType, ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage) throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {
        Object body;
        Class valueType;
        Object targetType;
        if (value instanceof CharSequence) {
            body = value.toString();
            valueType = String.class;
            targetType = String.class;
        } else {
            body = value;
            valueType = this.getReturnValueType(value, returnType);
            targetType = GenericTypeResolver.resolveType(this.getGenericType(returnType), returnType.getContainingClass());
        }
		//	是否是Resource
        if (this.isResourceType(value, returnType)) {
           	//	省略代码
            ...
        }
		// 内容协商（浏览器会以请求头的方式告诉服务器能接受什么样的内容
        MediaType selectedMediaType = null;
        MediaType contentType = outputMessage.getHeaders().getContentType();
        boolean isContentTypePreset = contentType != null && contentType.isConcrete();
        if (isContentTypePreset) {
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Found 'Content-Type:" + contentType + "' in response");
            }

            selectedMediaType = contentType;
        } else {
            //	获取请求信息
            HttpServletRequest request = inputMessage.getServletRequest();

            List acceptableTypes;
            try {
                //	获取客户端（浏览器，PostMan）支持的内容类型（获取客户端Accept字段）
                acceptableTypes = this.getAcceptableMediaTypes(request);
            } catch (HttpMediaTypeNotAcceptableException var20) {
                int series = outputMessage.getServletResponse().getStatus() / 100;
                if (body != null && series != 4 && series != 5) {
                    throw var20;
                }

                if (this.logger.isDebugEnabled()) {
                    this.logger.debug("Ignoring error response content (if any). " + var20);
                }

                return;
            }
			//	核心：获取服务端支持的内容类型
            List<MediaType> producibleTypes = this.getProducibleMediaTypes(request, valueType, (Type)targetType);
            if (body != null && producibleTypes.isEmpty()) {
                throw new HttpMessageNotWritableException("No converter found for return value of type: " + valueType);
            }
			
            //	通过匹配客户端和服务端的HTTPMessageConverter，得到两种的交集
            List<MediaType> mediaTypesToUse = new ArrayList();
            Iterator var15 = acceptableTypes.iterator();

            MediaType mediaType;
            while(var15.hasNext()) {
                mediaType = (MediaType)var15.next();
                Iterator var17 = producibleTypes.iterator();

                while(var17.hasNext()) {
                    MediaType producibleType = (MediaType)var17.next();
                    if (mediaType.isCompatibleWith(producibleType)) {
                        mediaTypesToUse.add(this.getMostSpecificMediaType(mediaType, producibleType));
                    }
                }
            }

            if (mediaTypesToUse.isEmpty()) {
                if (body != null) {
                    throw new HttpMediaTypeNotAcceptableException(producibleTypes);
                }

                if (this.logger.isDebugEnabled()) {
                    this.logger.debug("No match for " + acceptableTypes + ", supported: " + producibleTypes);
                }

                return;
            }
			//	如果支持多个Conveter根据HTTP请求权重去排序
            MediaType.sortBySpecificityAndQuality(mediaTypesToUse);
            var15 = mediaTypesToUse.iterator();

            while(var15.hasNext()) {
                mediaType = (MediaType)var15.next();
                if (mediaType.isConcrete()) {
                    selectedMediaType = mediaType;
                    break;
                }

                if (mediaType.isPresentIn(ALL_APPLICATION_MEDIA_TYPES)) {
                    selectedMediaType = MediaType.APPLICATION_OCTET_STREAM;
                    break;
                }
            }

            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Using '" + selectedMediaType + "', given " + acceptableTypes + " and supported " + producibleTypes);
            }
        }
    
     HttpMessageConverter converter;
    
    GenericHttpMessageConverter genericConverter;
    //	再来遍历一次获取MessageConverter
    //	注意来梳理一下writeWithMessageConverters会获取两次Converter
    //	这一次是使用MessageConverter将请求报文转换为JAVA对象
    //	这一次时使用MessageConverter将JAVA对象转换为响应报文
    label183: {
            if (selectedMediaType != null) {
                selectedMediaType = selectedMediaType.removeQualityValue();
                Iterator var23 = this.messageConverters.iterator();

                while(var23.hasNext()) {
                    converter = (HttpMessageConverter)var23.next();
                    genericConverter = converter instanceof GenericHttpMessageConverter ? (GenericHttpMessageConverter)converter : null;
                    if (genericConverter != null) {
                        if (((GenericHttpMessageConverter)converter).canWrite((Type)targetType, valueType, selectedMediaType)) {
                            break label183;
                        }
                    } else if (converter.canWrite(valueType, selectedMediaType)) {
                        break label183;
                    }
                }
            }

            if (body != null) {
                Set<MediaType> producibleMediaTypes = (Set)inputMessage.getServletRequest().getAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);
                if (!isContentTypePreset && CollectionUtils.isEmpty(producibleMediaTypes)) {
                    throw new HttpMediaTypeNotAcceptableException(this.getSupportedMediaTypes(body.getClass()));
                }

                throw new HttpMessageNotWritableException("No converter for [" + valueType + "] with preset Content-Type '" + contentType + "'");
            }

            return;
        }
		
        body = this.getAdvice().beforeBodyWrite(body, returnType, selectedMediaType, converter.getClass(), inputMessage, outputMessage);
        if (body != null) {
            LogFormatUtils.traceDebug(this.logger, (traceOn) -> {
                return "Writing [" + LogFormatUtils.formatValue(body, !traceOn) + "]";
            });
            this.addContentDispositionHeader(inputMessage, outputMessage);
            if (genericConverter != null) {
                // 将返回值写入body中
                genericConverter.write(body, (Type)targetType, selectedMediaType, outputMessage);
            } else {
                converter.write(body, selectedMediaType, outputMessage);
            }
        } else if (this.logger.isDebugEnabled()) {
            this.logger.debug("Nothing to write: null body");
        }

    }
```

![image-20210926212622756](D:\Blog\My-Blog\pic\image-20210926212622756.png)

```java
protected List<MediaType> getProducibleMediaTypes(HttpServletRequest request, Class<?> valueClass, @Nullable Type targetType) {
        Set<MediaType> mediaTypes = (Set)request.getAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);
        if (!CollectionUtils.isEmpty(mediaTypes)) {
            return new ArrayList(mediaTypes);
        } else {
            List<MediaType> result = new ArrayList();
            //	messageConverters如下图所示
            //	HttpMessageConverter: 看是否支持将 此 Class类型的对象，转为MediaType类型的数据。
			//	例如：Person对象转为JSON。或者 JSON转为Person
            Iterator var6 = this.messageConverters.iterator();
			//	遍历当前系统所有MessageConventer，看哪个Conveter支持这个对象(Person)，将其add到producibleTypes中
            while(true) {
                while(var6.hasNext()) {
                    HttpMessageConverter<?> converter = (HttpMessageConverter)var6.next();
                    if (converter instanceof GenericHttpMessageConverter && targetType != null) {
                        if (((GenericHttpMessageConverter)converter).canWrite(targetType, valueClass, (MediaType)null)) {
                            result.addAll(converter.getSupportedMediaTypes(valueClass));
                        }
                    } else if (converter.canWrite(valueClass, (MediaType)null)) {
                        result.addAll(converter.getSupportedMediaTypes(valueClass));
                    }
                }

                return (List)(result.isEmpty() ? Collections.singletonList(MediaType.ALL) : result);
            }
        }
    }

```

![img](D:\Blog\My-Blog\pic\1618420-20200408173711849-1473732014.png)









