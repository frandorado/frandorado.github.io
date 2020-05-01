---
layout: post
title:  "Logging of Requests and Responses in Spring with Undertow (no body)"
author: frandorado
categories: [spring]
tags: [spring, logging, request, response, java, body, undertow, RequestDumpingHandler, undertow]
image: assets/images/posts/2018-11-04/header.jpg
toc: true
---

In this post we're going to configure Undertow as embedded server in our Spring Boot Application. Then we'll trace in logs all the requests and responses of our controllers.

## Configuring Undertow 

Add the Undertow dependency to our Spring Boot Application and exclude `tomcat` in your pom.xml

```xml
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
	<artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

Enable RequestDumpingHandler adding the configuration bean of Undertow

In Spring boot < 2.0

```java
@Bean
public UndertowEmbeddedServletContainerFactory undertowEmbeddedServletContainerFactory() {
    UndertowEmbeddedServletContainerFactory factory = new UndertowEmbeddedServletContainerFactory();
    factory.addDeploymentInfoCustomizers(deploymentInfo -> 
           	deploymentInfo.addInitialHandlerChainWrapper(handler -> {
    	return new RequestDumpingHandler(handler);
    }));
        
    return factory;
}
```

In Spring boot >= 2.0

```java
@Bean
public UndertowServletWebServerFactory undertowServletWebServerFactory() {
    UndertowServletWebServerFactory factory = new UndertowServletWebServerFactory();
    factory.addDeploymentInfoCustomizers(deploymentInfo -> 
           	deploymentInfo.addInitialHandlerChainWrapper(handler -> {
    	return new RequestDumpingHandler(handler);
    }));
        
    return factory;
}
```


## The result

Now when we invoke our controller we will see a result like this:

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

## References

[1] Link to the project in [Github][github-link]


[github-link]: https://github.com/frandorado/spring-projects/tree/master/log-request-response-undertow
