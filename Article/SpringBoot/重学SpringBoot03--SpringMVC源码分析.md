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



### 1.	HttpServlet初始化

在我们第一次请求的时候会进行Servlet的初始化，主要用来初始化资源。HttpServlet的init方法由父类GenericServlet声明，由子类HttpServletBean实现。



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

### 3.	返回值原理

