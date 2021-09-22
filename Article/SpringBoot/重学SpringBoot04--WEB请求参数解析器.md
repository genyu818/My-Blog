# SpringBoot WEB开发常用注解参数解析器

### 1.	常用请求参数注解

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





### 2.	参数解析器原理

下面来详细看一下，SpringMVC是如何获取到参数的，回想一下上一章，SpringMVC执行HTTP请求映射原理的核心方法放在了``doDispatch()``中，这节的请求参数原理的核心方法依旧放在``doDispatch()``中

#### 2.1	基本参数绑定原理

首先，针对两种较为简单的注解@PathVariable,@RequestParam(写进URL的参数)，来看一下SpringMVC如何解析参数。

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

- **ServletModelAttributeMethodProcessor**
- **是否为简单类型。**
- **WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name);**
- bindRequestParam
- **GenericConversionService：在设置每一个值的时候，找它里面的所有converter那个可以将这个数据类型（request带来参数的字符串）转换到指定的类型（JavaBean -- Integer）**