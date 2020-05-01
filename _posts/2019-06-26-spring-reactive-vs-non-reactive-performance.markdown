---
layout: post
title:  "Reactive vs Non-Reactive Spring Performance"
author: frandorado
categories: [spring]
tags: [spring, reactive, performance, mvc, webflux, mono, flux, stack]
image: assets/images/posts/2019-06-26/spring-mvc-webflux-chart.png
toc: true
---

The goal of this article is to compare the performance between Spring MVC (Non-Reactive) and Spring WebFlux (Reactive). For this task we have taken into account three differents implementations:

* Using Spring MVC synchronous
* Using Spring MVC asynchronous
* Using Spring WebFlux


## Implementations
### Using Spring MVC synchronous

For this implementation the next endpoints have been defined:

```java
@RequestMapping("/mvcsync/{id}")
public Message findById(@PathVariable(value = "id") String id) {
    return nonReactiveRepository.findById(id).orElse(null);
}
    
@PostMapping("/mvcsync")
public Message post(@Valid @RequestBody Message message) {
    return nonReactiveRepository.save(message);
}
```

### Using Spring MVC asynchronous

Java 8 CompletableFuture library has been used to get asynchronous methods.

```java
@RequestMapping("/mvcasync/{id}")
public CompletableFuture<Message> findById(@PathVariable(value = "id") String id) {
    return CompletableFuture.supplyAsync(() -> nonReactiveRepository.findById(id).orElse(null));
}
    
@PostMapping("/mvcasync")
public CompletableFuture<Message> post(@Valid @RequestBody Message message) {
    return CompletableFuture.supplyAsync(() -> nonReactiveRepository.save(message));
}
```

### Using Spring WebFlux

The reactive version including MongoDB Reactive Repository.

```java
@RequestMapping("/reactive/{id}")
public Mono<Message> findByIdReactive(@PathVariable(value = "id") String id) {
    return reactiveRepository.findById(id);
}
    
@PostMapping("/reactive")
public Mono<Message> postReactive(@Valid @RequestBody Message message) {
    return reactiveRepository.save(message);
}
```

## The Test

Using [JMeter](https://jmeter.apache.org) we have created a test  with the next characteristics:

* Each iteration will be composed by one create operation (POST) and one find operation (GET) to MongoDB.
* For each test, we are defining the number of concurrent users/threads

Hardware:

* MacBook Pro (15-inch, 2016)
* 2,6 GHz Intel Core i7
* 16 GB 2133 MHz LPDDR3

## Results

The results shown in the following table refer to a one-minute execution with a one-second ramp-up period.

![Result Table]({{site.url}}/assets/images/posts/2019-06-26/spring-mvc-webflux-table.png "Result Table")

Now, we can extract the following conclusions based in our tests:

* Spring MVC Sync vs Async are similar in performance but with the asynchronous implementation we have a less error rate at beginning of the execution
* Spring WebFlux allows an increment in performance greater than **60%** respect to Spring MVC

## References

[1] Link to the project in [Github][github-link]


[github-link]: https://github.com/frandorado/spring-projects/tree/master/spring-reactive-nonreactive