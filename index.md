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

`ExecutorService` instrumentation:
```java
ExecutorService monitorExecutor(MeterRegistry meterRegistry, ExecutorService executor, String executorName) {
    return ExecutorServiceMetrics.monitor(meterRegistry, executor, executorName);
}
```

## Know Your Gauges

```java
class GaugesCache {

    /*
     * From Micrometer docs (https://micrometer.io/docs/concepts#_gauges):
     *
     *    Attempting to construct a gauge with a primitive number or one of its java.lang object forms
     *    is always incorrect. These numbers are immutable, and thus the gauge cannot ever be changed.
     *    Attempting to "re-register" the gauge with a different number won’t work, as the registry only
     *    maintains one meter for each unique combination of name and tags.
     */
    private static final Map<GaugeCacheKey, AtomicDouble> gaugesCache = new ConcurrentHashMap<>();

    /**
     * Update or create a gauge metric with provided value associated with metric name-tags pair.
     */
    AtomicDouble upsert(String metricName, Tags metricTags, double value) {
        AtomicDouble atomic = gaugesCache.computeIfAbsent(
                GaugeCacheKey.from(metricName, metricTags),
                __ -> new AtomicDouble());
        atomic.set(value);
        return atomic;
    }

    /**
     * Combination of metric name and tags should result into unique tuple,
     * similarly how metric time-series are being * persisted and identified
     * in a long-term storage.
     */
    private static class GaugeCacheKey {

        private final String metricName;
        private final Tags tags;

        private GaugeCacheKey(String metricName,
                              Tags tags) {
            this.metricName = metricName;
            this.tags = tags;
        }

        static GaugeCacheKey from(String metricName, Tags tags) {
            return new GaugeCacheKey(metricName, tags);
        }

        @Override
        public boolean equals(Object o) { ... }

        @Override
        public int hashCode() { ... }
    }
}
```

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