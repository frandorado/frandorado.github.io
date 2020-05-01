---
layout: post
title:  "Circuit Breaker with Resilience4j and Spring"
author: frandorado
categories: [spring]
tags: [spring, resilience4j, circuit breaker]
image: assets/images/posts/2019-01-04/circuitbreaker-resilience4j-spring.png
toc: true
---

When a remote service is down the [Circuit Breaker][circuitbreaker-martinfowler-link] pattern prevents a cascade of failures. After a number of failed attempts, we can consider that the service is unavailable/overloaded and reject all subsequent requests to it. The Circuit Breaker acts like a switch that opens or closes a circuit.

In this post we'll talk about the `resilience4j` library that allows us to apply this pattern.

## Dependencies

Add the next dependencies to your `pom.xml`

```xml
<properties>
    <resilience4j.version>0.13.2</resilience4j.version>
</properties>

<dependencies>
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-spring-boot</artifactId>
        <version>${resilience4j.version}</version>
    </dependency>
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-metrics</artifactId>
        <version>${resilience4j.version}</version>
    </dependency>
</dependencies>
```

## Configuration

For the next configuration we have defined two circuit breaker configurations, `default` and `serviceA`.

```yaml
resilience4j.circuitbreaker:
  backends:
    default:
      ringBufferSizeInClosedState: 3
      ringBufferSizeInHalfOpenState: 3
      waitInterval: 1000
      failureRateThreshold: 20
    serviceA:
      ringBufferSizeInClosedState: 2
      ringBufferSizeInHalfOpenState: 2
      waitInterval: 1000
      failureRateThreshold: 50
#      registerHealthIndicator: false
#      recordExceptions:
#      - org.springframework.web.client.HttpServerErrorException
#      ignoreExceptions:
#      - org.springframework.web.client.HttpClientErrorException
```
* **ringBufferSizeInClosedState**: The size of ring buffer when the CircuitBreaker is closed
* **ringBufferSizeInHalfOpenState**: The size of ring buffer when the CircuitBreaker is half open
* **waitInterval**: The wait duration in millis which defines how long the CircuitBreaker should stay open before it switches to half open
* **failureRateThreshold**: The failure rate threshold above which the CircuitBreaker opens and starts short-circuiting calls

## Example of Circuit Breaker
### The test

We'll call to a Consumer that iterate 5 times executing a method that will throw an exception. We will log:
* The begin of method "Entering in service ..."
* The exception that will return the service "Exception in method"
* When a CircuitBreaker is opened "Circuit breaker applied"

```java
private static final Consumer<Runnable> consumer = runnable -> IntStream.range(0, 5).forEach(value -> {
    try {
        runnable.run();
    } catch (CircuitBreakerOpenException e) {
        log.warn("Circuit breaker applied");
    } catch (Exception e) {
        log.warn("Exception in method");
    }
});
```

### Circuit Breaker using annotations

You could annotate the method or the entire class with `@CircuitBreaker` annotation. Example for method annotation with `serviceA` configuration.

```java
@CircuitBreaker(name = "serviceA")
public void runWithCircuitBreaker() {
    // This code will throw an exception and will not be executed when applies Circuit Breaker
}
```
An example of 5 invocations to a method that always throws exception, the Circuit Breaker mechanism will be applied. The method will be invoked 2 times and the next 3 times

```console
2019-01-02 20:15:26.936  INFO ... : Running with circuit breaker using annotations ...
2019-01-02 20:15:26.943  INFO ... : Entering in service ...
2019-01-02 20:15:27.431  WARN ... : Exception in method
2019-01-02 20:15:27.432  INFO ... : Entering in service ...
2019-01-02 20:15:28.106  WARN ... : Exception in method
2019-01-02 20:15:28.107  WARN ... : Circuit breaker applied
2019-01-02 20:15:28.107  WARN ... : Circuit breaker applied
2019-01-02 20:15:28.107  WARN ... : Circuit breaker applied
```

### Circuit Breaker with direct invocation

You can decorate any Supplier / Runnable / Function or CheckedRunnable / CheckedFunction function with CircuitBreaker.decorateCheckedSupplier(), CircuitBreaker.decorateCheckedRunnable() or CircuitBreaker.decorateCheckedFunction()

```java
CircuitBreaker defaultCircuitBreaker = circuitBreakerRegistry.circuitBreaker("default");
Runnable decoratedRunnable = CircuitBreaker.decorateRunnable(defaultCircuitBreaker, () -> circuitBreakerTestService.run(false));
```

The same example of the previous case, 5 invocations but applying the `default` configuration:

```console
2019-01-02 20:15:28.107  INFO ... : Running with default circuit breaker using manual invocation ...
2019-01-02 20:15:28.108  INFO ... : Entering in service ...
2019-01-02 20:15:28.298  WARN ... : Exception in method
2019-01-02 20:15:28.298  INFO ... : Entering in service ...
2019-01-02 20:15:28.485  WARN ... : Exception in method
2019-01-02 20:15:28.486  INFO ... : Entering in service ...
2019-01-02 20:15:28.670  WARN ... : Exception in method
2019-01-02 20:15:28.670  WARN ... : Circuit breaker applied
2019-01-02 20:15:28.670  WARN ... : Circuit breaker applied
```

## References

[1] Link to the project in [Github][github-link]

[2] CircuitBreaker of Martin Fowler [Circuit Breaker][circuitbreaker-martinfowler-link]

[3] Resilience4j in Baeldung [Resilience4j][resilience4j-baeldung-link]

[4] Resilience4j in Github [Resilience4j Github][resilience4j-official]

[github-link]: https://github.com/frandorado/spring-projects/tree/master/resilience4j-spring
[circuitbreaker-martinfowler-link]: https://martinfowler.com/bliki/CircuitBreaker.html
[resilience4j-baeldung-link]: https://www.baeldung.com/resilience4j
[resilience4j-official]: https://resilience4j.github.io/resilience4j

