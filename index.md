[Micrometer](https://micrometer.io/) is an application metrics facade which is widely used and adopted by N26 teams.
It goes with a plenty of instrumentations for various metric backend datastores, bindings to different components of your services and is a default metrics collector in Spring Boot 2.x.

This page is a small collection of findings and pitfalls you may find useful during Micrometer integration. 

* [Bindings](#bindings)
* [Know Your Gauges](#know-your-gauges)
* [Tags Hell](#tags-hell)

## Bindings

Micrometer goes with a bunch of handy pre-configured [bindings](https://github.com/micrometer-metrics/micrometer/tree/master/micrometer-core/src/main/java/io/micrometer/core/instrument/binder)
that can provide insights on your application internals on system, datatabase, jvm etc. with minimum configuration required. 
Many of these metrics are registered [out-of-the-box]((https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready-metrics-meter)) with Spring Boot.

`ExecutorService` instrumentation:
```java
ExecutorService monitorExecutor(MeterRegistry meterRegistry, ExecutorService executor, String executorName) {
    return ExecutorServiceMetrics.monitor(meterRegistry, executor, executorName);
}
```

## Know Your Gauges

During the migration from `StatsDClient` to `Micrometer` we found how different the latter handles gauge-metrics.

```java
statsDClient.gauge("metrics", value);
```

From Micrometer [documentation](https://micrometer.io/docs/concepts#_gauges):
> A gauge can be made to track any java.lang.Number subtype that is settable, 
> such as AtomicInteger and AtomicLong found in java.util.concurrent.atomic 
> and similar types like Guava’s AtomicDouble.

![Image of Gauge Warning](/assets/img/gauge-warning.png)

```java
class GaugesCache {

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
     * similarly how metric time-series are being persisted and identified
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