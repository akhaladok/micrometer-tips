[Micrometer](https://micrometer.io/) is an application metrics facade which is widely used and adopted by N26 teams.
It goes with a plenty of instrumentations for various metric backend datastores, bindings to different components of 
your services and is a default metrics collector in Spring Boot 2.x.

This page is a small collection of findings and pitfalls we faced during Micrometer integration. 

* [Bindings](#bindings)
* [Know Your Gauges](#know-your-gauges)
* [Tags Hell](#tags-hell)

## Bindings

Micrometer goes with a bunch of handy pre-configured [bindings](https://github.com/micrometer-metrics/micrometer/tree/master/micrometer-core/src/main/java/io/micrometer/core/instrument/binder)
that can provide insights on your application internals (system, database, jvm etc.) with minimum configuration required. 
Many of these metrics are registered [out-of-the-box]((https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready-metrics-meter)) with Spring Boot.

The following `ExecutorService` instrumentation will provide metrics for internal tasks-queue and thread-pool:
```kotlin
fun monitorExecutor(meterRegistry: MeterRegistry, 
                    executor: ExecutorService, 
                    executorName: String): ExecutorService = 
    ExecutorServiceMetrics.monitor(meterRegistry, executor, executorName)
```

## Know Your Gauges

If you migrate to `Micrometer` from `StatsDClient` 
During the migration from `StatsDClient` to `Micrometer` we found how different the latter handles gauge-metrics.

```kotlin
statsDClient.gauge("metrics", value)
```

From Micrometer [documentation](https://micrometer.io/docs/concepts#_gauges):
> A gauge can be made to track any java.lang.Number subtype that is settable, 
> such as AtomicInteger and AtomicLong found in java.util.concurrent.atomic 
> and similar types like Guava’s AtomicDouble.

![Image of Gauge Warning](/assets/img/gauge-warning.png)

```kotlin
class GaugesCache {

    /**
     * Update or create a gauge metric with provided 
     * value associated with metric name-tags pair.
     */
    fun upsert(metricName: String, 
               metricTags: Tags = Tags.empty(), 
               value: Double): AtomicDouble {
        val atomic = gaugesCache.computeIfAbsent(
            GaugeCacheKey(metricName, metricTags)) { AtomicDouble() }
        atomic.set(value)
        return atomic
    }

    /**
     * Combination of metric name and tags should result into unique tuple,
     * similarly how metric time-series are being persisted and identified
     * in a long-term storage.
     */
    data class GaugeCacheKey(
        private val metricName: String,
        private val tags: Tags
    )

    companion object {
        private val gaugesCache = ConcurrentHashMap<GaugeCacheKey, AtomicDouble>()
    }
}
```

## Tags Hell

![Image of Memory Leak](/assets/img/memory-leak.png)

```kotlin
Timer.builder(metricName)
    .tags(tags.map { (k, v) -> Tag.of(k, v.toString()) })
    .register(registry)
    .record(duration)
```