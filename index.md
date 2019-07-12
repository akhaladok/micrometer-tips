[Micrometer](https://micrometer.io/) is an application metrics facade which is widely used and adopted by N26 teams.
It goes with a plenty of instrumentations for various metric backend datastores, bindings to different components of your services and is a default metrics collector in Spring Boot 2.x.
This page is a small collection of findings and pitfalls you may find useful during Micrometer integration. 

* [Bindings](#bindings)
* [Know Your Gauges](#know-your-gauges)
* [Tags Hell](#tags-hell)

## Bindings

Micrometer goes with a bunch of pre-configured [bindings](https://github.com/micrometer-metrics/micrometer/tree/master/micrometer-core/src/main/java/io/micrometer/core/instrument/binder)
that can provide insights on your application internals with minimum configuration required: system, database, jvm etc. 
[Many](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready-metrics-meter) of these metrics are ready out-of-the-box with Spring Boot.

```java
ExecutorService monitorExecutorService(
	MeterRegistry meterRegistry, 
	ExecutorService executor, 
	String executorName) {
    return ExecutorServiceMetrics.monitor(
    	meterRegistry, 
    	executor, 
    	executorName);
}
```

## Know Your Gauges

## Tags Hell

```kotlin
fun executeWithMetrics(path: String, method: String, metrics: Metrics): Try<Response<T>> {
        return metrics.measureTime(
            "http.outgoing",
            "path" to path,
            "method" to method
        ) {...}
}
```