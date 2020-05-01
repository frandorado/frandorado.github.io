---
layout: post
title:  "AsyncRestTemplate returns 404 (Site Not Found) with Apache HttpComponents"
author: frandorado
categories: [spring]
tags: [spring, asyncresttemplate, apache, factory, HttpComponentsAsyncClientHttpRequestFactory, site not found, "404"]
image: assets/images/posts/2018-12-17/asyncresttemplate-apache-404.png
toc: true
---

When you create an `AsyncRestTemplate`, Spring uses `SimpleClientHttpRequestFactory` by default which depends on default configuration of `HttpURLConnection`. This implementation doesn't have connection pooling.

Apache HttpComponents provides the `HttpComponentsAsyncClientHttpRequestFactory` with connection pooling support and using NIO (Non-blocking Input/Output) asynchronous http client, called `HttpAsyncClient`.


## The problem

In a recent project, we decided to change the the Spring's default `SimpleClientHttpRequestFactory` to the Apache Components factory `HttpComponentsAsyncClientHttpRequestFactory` by the advantages of connection pooling. The solution seemed easy, you should only add the dependency:

```xml
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpasyncclient</artifactId>
</dependency>
```
and change the Spring Bean's definition from this:

```java
@Bean
public AsyncRestTemplate asyncRestTemplate() {
    return new AsyncRestTemplate();
}
```
to this:

```java
@Bean
public AsyncRestTemplate asyncRestTemplate() {
    return new AsyncRestTemplate(new HttpComponentsAsyncClientHttpRequestFactory());
}
```

But when we did a request from Postman to our controller propagating the received headers to a new call using for example `https://httpstat.us/200` (GET), we received this error:

```
Async GET request for "https://httpstat.us/200" resulted in 404 (Site Not Found); invoking error handler
```

The same request with `SimpleClientHttpRequestFactory` returned "200 OK".

## The solution

After a time trying to resolve the error we found that Postman includes a header called `host` that provokes a 404 with the Apache Factory implementation.  

> host=localhost:8080

So, the solution was to delete this header from the request. You could be a example of this error in [1]. In this example we make four calls with and without headers and using the two factories. The log result:

```
===== Execution without host header =====
Factory=[SimpleClientHttpRequestFactory] ResponseBody=["200 OK"]
Factory=[HttpComponentsAsyncClientHttpRequestFactory] ResponseBody=["200 OK"]

===== Execution with host header =====
Factory=[SimpleClientHttpRequestFactory] ResponseBody=["200 OK"]
Factory=[HttpComponentsAsyncClientHttpRequestFactory] Exception=[org.springframework.web.client.HttpClientErrorException: 404 Site Not Found]

```


## References

[1] Link to the project in [Github][github-link]


[github-link]: https://github.com/frandorado/spring-projects/tree/master/async-rest-template-apache
