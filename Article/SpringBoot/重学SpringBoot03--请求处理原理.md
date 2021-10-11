# 请求映射原理--DispatcherServlet源码



首先看到DispatcherServlet继承于FrameworkServlet

```java
public class DispatcherServlet extends FrameworkServlet
```



使用IDEA的的继承插件看一下DispatcherServlet的继承树，我们看到还是继承于HttpServerlet。

<img src="D:\Blog\My-Blog\pic\image-20210919190256617.png" alt="image-20210919190256617" style="zoom:50%;" />

继承于HttpServerlet一定要重写doGet(), doPost()等方法，我们来搜索一下，发现HttpServerletBean中并没有重写这些方法，我们再来FrameworkServlet查看在此类中重写了这些方法，且这些方法都调用了FrameworkServlet中的 processRequest(request, response).

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

    protected final void doDelete(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        this.processRequest(request, response);
    }

```



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

// doService发现是一个抽象方法,并没有实现,所以饶了一圈,还是绕道了DispatcherServlet中的doService方法
protected abstract void doService(HttpServletRequest var1, HttpServletResponse var2) throws Exception;


// 查看到DispatcherServlet中的DoService方法后看到核心还是使用doDispatch方法去做Http请求的派发
   try {
            this.doDispatch(request, response);
        } finally {
            if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted() && attributesSnapshot != null) {
                this.restoreAttributesAfterInclude(request, attributesSnapshot);
            }

            if (this.parseRequestPath) {
                ServletRequestPathUtils.setParsedRequestPath(previousRequestPath, request);
            }

        }

```



如上源码所述,DispatcherServlet中的请求处理原理的核心就是doDispatch方法

![image-20210919192208909](D:\Blog\My-Blog\pic\image-20210919192208909.png)



最终我们得到SpringMVC HTTP请求处理都从org.springframework.web.servlet.DispatcherServlet中 --> doDispatch()进行处理

我们来打个断电来进行逐行分析, 进行之前一个HTTP请求 http://localhost:8080/test

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

根据以上源码解析,请求处理的核心就是getHandler()方法,它将我们的请求处理和处理方法进行绑定映射,继续详细解析这个方法.

首先是handlerMappings, 展开来看,这就是我们Spring中所有的请求种类.

RequestMappingHandlerMapping : 里边就保存了我们所有经过@RequestMapping注册规则,

![image-20210919194652486](C:\Users\MSI-PC\AppData\Roaming\Typora\typora-user-images\image-20210919194652486.png)

```java
    protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
        if (this.handlerMappings != null) {
            Iterator var2 = this.handlerMappings.iterator();
			//	遍历所有的HandlerMapping
            // 	RequestMappingHandlerMapping,BeanNameUrlHandlerMapping,SimpleUrlHandlerMapping,RouterFunctionMapping
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



继续执行发现getHandler() 是调用lookupHandlerMethod()方法

```java
// lookupPath请求路径 /test
protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
        
    List<AbstractHandlerMethodMapping<T>.Match> matches = new ArrayList();
    //	获得和路径匹配的所有方法    
    List<T> directPathMatches = this.mappingRegistry.getMappingsByDirectPath(lookupPath);
        if (directPathMatches != null) {
            // 根据请求方式将路径和请求方法的映射塞进matches中
            this.addMatchingMappings(directPathMatches, matches, request);
        }

        if (matches.isEmpty()) {
            this.addMatchingMappings(this.mappingRegistry.getRegistrations().keySet(), matches, request);
        }

        if (matches.isEmpty()) {
            return this.handleNoMatch(this.mappingRegistry.getRegistrations().keySet(), lookupPath, request);
        } else {
            
            AbstractHandlerMethodMapping<T>.Match bestMatch = (AbstractHandlerMethodMapping.Match)matches.get(0);
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
                        AbstractHandlerMethodMapping<T>.Match match = (AbstractHandlerMethodMapping.Match)var7.next();
                        if (match.hasCorsConfig()) {
                            return PREFLIGHT_AMBIGUOUS_MATCH;
                        }
                    }
                } else {
                    AbstractHandlerMethodMapping<T>.Match secondBestMatch = (AbstractHandlerMethodMapping.Match)matches.get(1);
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





## 总结

来进行简单的总结

* DispatcherServlet继承于HttpServlet, 请求处理的最重要核心是doDispatch()
* HandlerMapping中保存了Spring中所有的请求种类
  * 请求进来会遍历HandlerMapping看是否有处理当前请求的handler
  * 如果没有就进入下一个handler(通过@RequestMapping标注的Controller会保存在RequestMappingHandlerMapping)
* 如果一个请求路径对应多个Controller, 会根据请求类型进行排序选取最优的Controller