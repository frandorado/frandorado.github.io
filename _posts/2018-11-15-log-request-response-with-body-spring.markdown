---
layout: post
title:  "Logging Requests and Responses in Spring (including body)"
author: frandorado
categories: [spring]
tags: [spring, logging, request, response, java, body]
image: assets/images/posts/2018-11-15/header.jpg
toc: true
---

Recently we have found some problems trying to log a complete Request and Response in a Spring Application. When we talk about a "complete Request and Response" we are indicating that we want to include the content of body. In this post we will talk about how to resolve this problem.

## Alternatives

When you look for differents alternatives to log the request and response you could find the next solutions:

### Using the embedded server (Tomcat, Jetty, Undertow)

This is a good solution if you don't need to log the value of body. If you want an example with Undertow you can see my previous post about this [Logging Request and Response with Undertow](........)

* Result:
    ```
    2018-10-25 19:54:49.900  INFO 81483 --- [  XNIO-2 task-3] io.undertow.request.dump: 
    ----------------------------REQUEST---------------------------
                   URI=/songs
     characterEncoding=null
         contentLength=-1
           contentType=null
                cookie=JSESSIONID=OxiTJHfy8yqLY3yT8scMfirBMRcl-imcPTcCDpYe
                header=Postman-Token=a80517f2-dd94-4f96-ac5d-e859c66343a1
                header=Accept=*/*
                header=Connection=keep-alive
                header=cache-control=no-cache
                header=accept-encoding=gzip, deflate
                header=cookie=JSESSIONID=OxiTJHfy8yqLY3yT8scMfirBMRcl-imcPTcCDpYe
                header=User-Agent=PostmanRuntime/7.3.0
                header=Host=localhost:8080
                locale=[]
                method=GET
              protocol=HTTP/1.1
           queryString=
            remoteAddr=/0:0:0:0:0:0:0:1:58800
            remoteHost=localhost
                scheme=http
                  host=localhost:8080
            serverPort=8080
              isSecure=false
    --------------------------RESPONSE--------------------------
         contentLength=-1
           contentType=application/json;charset=UTF-8
                header=Connection=keep-alive
                header=Transfer-Encoding=chunked
                header=Content-Type=application/json;charset=UTF-8
                header=Date=Thu, 25 Oct 2018 17:54:49 GMT
                status=200

    ==============================================================
    ```
* Disadvantages
    * It doesn't include the body
    * We can't define the template for log

### Using CommonsRequestLoggingFilter

The same than before, there is a [Bug](https://github.com/grails/grails-core/issues/11024) with Spring Boot (< 2.0) that doesn't print the content of body. Furthermore only will log the request.

* Code
    ```java
    @Bean
    public CommonsRequestLoggingFilter logFilter() {
        CommonsRequestLoggingFilter filter = new CommonsRequestLoggingFilter();
        
        filter.setIncludeQueryString(true);
        filter.setIncludePayload(true);
        filter.setMaxPayloadLength(10000);
        filter.setIncludeHeaders(false);
        filter.setAfterMessagePrefix("REQUEST DATA : ");
        
        return filter;
    }
    ```

* Result in Spring Boot > 2.0:
    ```
    2018-10-27 16:53:49.627 DEBUG 88615 --- [nio-8080-exec-2] o.s.w.f.CommonsRequestLoggingFilter      : REQUEST DATA : uri=/greetings;payload={
        "message": "Hello Java world!"
    }]
    ```

* Disadvantages
    * Only log the request
    * The body is not showed with Spring Boot < 2.0



### Using a handler interceptor

You could read the value of body in the Request in `preHandle` method of a `HandlerInterceptor`. This has the problem that the InputStream only can read once. Other solutions that I have found to avoid this is using a `ContentCachingRequestWrapper` but this didn't work for me.

* Code
    ```java
@Component
public class CustomHandlerInterceptorAdapter extends HandlerInterceptorAdapter {
    
    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) {
        
        ServletRequest servletRequest = new ContentCachingRequestWrapper(request);
        servletRequest.getParameterMap();
        
        // Read inputStream from requestCacheWrapperObject and log it
        
        return true;
    }
}
    ```

* Result:

    ```json
    {
        "timestamp": 1500645243383,
        "status": 400,
        "error": "Bad Request",
        "exception": "org.springframework.http.converter.HttpMessageNotReadableException",
        "message": "Could not read document: Stream closed; nested exception is java.io.IOException: Stream closed",
        "path": "/greetings"
    }
    ```

* Disadvantages
    * The Request can't be read twice and we can't log the body.

## Our solution

### Logging requests (GET methods)

The GET methods don't contain body so we'll use a HandlerInterceptor for this case.

```java
@Component
public class LogInterceptor implements HandlerInterceptor {
    
    @Autowired
    LoggingService loggingService;
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, 
                             Object handler) {
        
        if (DispatcherType.REQUEST.name().equals(request.getDispatcherType().name())
                && request.getMethod().equals(HttpMethod.GET.name())) {
            loggingService.logRequest(request, null);
        }
        
        return true;
    }
}
```

### Logging requests (POST, PUT, PATCH, DELETE ...)

For others methods that contains a body, we'll use RequestBodyAdviceAdapter.

```java
@ControllerAdvice
public class CustomRequestBodyAdviceAdapter extends RequestBodyAdviceAdapter {
    
    @Autowired
    LoggingService loggingService;
    
    @Autowired
    HttpServletRequest httpServletRequest;
    
    @Override
    public boolean supports(MethodParameter methodParameter, Type type, 
                            Class<? extends HttpMessageConverter<?>> aClass) {
        return true;
    }
    
    @Override
    public Object afterBodyRead(Object body, HttpInputMessage inputMessage,
                                MethodParameter parameter, Type targetType,
            Class<? extends HttpMessageConverter<?>> converterType) {
        
        loggingService.logRequest(httpServletRequest, body);
        
        return super.afterBodyRead(body, inputMessage, parameter, targetType, converterType);
    }
}
```
## Logging responses

```java
@ControllerAdvice
public class CustomResponseBodyAdviceAdapter implements ResponseBodyAdvice<Object> {
    
    @Autowired
    LoggingService loggingService;
    
    @Override
    public boolean supports(MethodParameter methodParameter,
                            Class<? extends HttpMessageConverter<?>> aClass) {
        return true;
    }
    
    @Override
    public Object beforeBodyWrite(Object o,
                                  MethodParameter methodParameter,
                                  MediaType mediaType,
                                  Class<? extends HttpMessageConverter<?>> aClass,
                                  ServerHttpRequest serverHttpRequest,
                                  ServerHttpResponse serverHttpResponse) {

        if (serverHttpRequest instanceof ServletServerHttpRequest &&
                serverHttpResponse instanceof ServletServerHttpResponse) {
            loggingService.logResponse(
                    ((ServletServerHttpRequest) serverHttpRequest).getServletRequest(),
                    ((ServletServerHttpResponse) serverHttpResponse).getServletResponse(), o);
        }
        
        return o;
    }
}
```

## The Logging Service

You can configure the logging service as you need, implementing this methods:

```java
public interface LoggingService {
    
    void logRequest(HttpServletRequest httpServletRequest, Object body);
    
    void logResponse(HttpServletRequest httpServletRequest, 
                     HttpServletResponse httpServletResponse, 
                     Object body);
}
```

You can see a concrete implementation in the project in github [1]

## References

[1] Link to the project in [Github][github-link]


[github-link]: https://github.com/frandorado/spring-projects/tree/master/log-request-response-with-body
