---
layout: post
title:  "Undertow metrics with Spring Actuator"
author: frandorado
categories: [spring]
tags: [spring, undertow, metrics, actuator, micrometer]
image: assets/images/posts/2020-03-31/spring-actuator-undertow.png
toc: true
---

This post is intended to show a possible solution to provide metrics of Undertow with Spring Actuator (Micrometer). Undertow provides its own metric collector, but we have encountered one issue where the values of these metrics are not being updated correctly because when we make a request the collector is incrementing the counter in two units instead of one. Consequently, we have created our own solution based on Undertow metrics collector where we providing next metrics:

* Number of requests
* Total number of error requests
* The longest request duration in time
* The shortest request duration in time

## Defining and registering a metrics handler

Undertow provides a class `MetricsHandler` with all the information about the requests. We will create a wrapper about this metrics handler and register it to the handler process in Undertow.

```java
@Component
public class UndertowMetricsHandlerWrapper implements HandlerWrapper {

    private MetricsHandler metricsHandler;

    @Override
    public HttpHandler wrap(HttpHandler handler) {
        metricsHandler = new MetricsHandler(handler);
        return metricsHandler;
    }

    public MetricsHandler getMetricsHandler() {
        return metricsHandler;
    }
}
```

For register the `HandlerWrapper` you must define a Bean of type `UndertowDeploymentInfoCustomizer` setting the handler.

```java
@Bean
UndertowDeploymentInfoCustomizer undertowDeploymentInfoCustomizer(
    UndertowMetricsHandlerWrapper undertowMetricsHandlerWrapper) {

    return deploymentInfo -> 
        deploymentInfo.addOuterHandlerChainWrapper(undertowMetricsHandlerWrapper);
}
```

## The MeterBinder

The next step is connect all MeterRegistry of our application to `MetricsHandler`. This operation should be executed when the application is ready because is the moment where the `MetricsHandler` is initialized

```java
@Component
public class UndertowMeterBinder implements ApplicationListener<ApplicationReadyEvent> {

    private final UndertowMetricsHandlerWrapper undertowMetricsHandlerWrapper;

    public UndertowMeterBinder(UndertowMetricsHandlerWrapper undertowMetricsHandlerWrapper) {
        this.undertowMetricsHandlerWrapper = undertowMetricsHandlerWrapper;
    }

    @Override
    public void onApplicationEvent(ApplicationReadyEvent applicationReadyEvent) {
        bindTo(applicationReadyEvent.getApplicationContext().getBean(MeterRegistry.class));
    }

    public void bindTo(MeterRegistry meterRegistry) {
        bind(meterRegistry, undertowMetricsHandlerWrapper.getMetricsHandler());
    }

    // ...
}
```

Now, we can add the binder methods to our class

```java
public void bind(MeterRegistry registry, MetricsHandler metricsHandler) {
    bindTimer(registry, "undertow.requests", "Number of requests", metricsHandler,
            m -> m.getMetrics().getTotalRequests(), m2 -> m2.getMetrics().getMinRequestTime());
    bindTimeGauge(registry, "undertow.request.time.max", "The longest request duration in time", metricsHandler,
            m -> m.getMetrics().getMaxRequestTime());
    bindTimeGauge(registry, "undertow.request.time.min", "The shortest request duration in time", metricsHandler,
            m -> m.getMetrics().getMinRequestTime());
    bindCounter(registry, "undertow.request.errors", "Total number of error requests ", metricsHandler,
            m -> m.getMetrics().getTotalErrors());

}

private void bindTimer(MeterRegistry registry, String name, String desc, MetricsHandler metricsHandler,
                       ToLongFunction<MetricsHandler> countFunc, ToDoubleFunction<MetricsHandler> consumer) {
    FunctionTimer.builder(name, metricsHandler, countFunc, consumer, TimeUnit.MILLISECONDS)
            .description(desc).register(registry);
}

private void bindTimeGauge(MeterRegistry registry, String name, String desc, MetricsHandler metricResult,
                           ToDoubleFunction<MetricsHandler> consumer) {
    TimeGauge.builder(name, metricResult, TimeUnit.MILLISECONDS, consumer).description(desc)
            .register(registry);
}

private void bindCounter(MeterRegistry registry, String name, String desc, MetricsHandler metricsHandler,
                         ToDoubleFunction<MetricsHandler> consumer) {
    FunctionCounter.builder(name, metricsHandler, consumer).description(desc)
            .register(registry);
}
```


## The project

In [1] it is available the project to test this metrics. We have used JMX Meter Binder and you can check the results using `jconsole`

![JConsole]({{site.url}}/assets/images/posts/2020-03-31/jconsole.png "JConsole")


## References

[1] [Link to the project in Github](https://github.com/frandorado/spring-projects/tree/master/spring-micrometer-undertow)

