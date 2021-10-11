# 返回值解析器

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













