[Micrometer](https://micrometer.io/) is a metrics facade for JVM-based applications which is widely used and adopted by N26 teams.
It goes with a plenty of instrumentations for various metric backend datastores, bindings to different components of 
your services and is a default metrics collector in Spring Boot 2.x (available backport for 1.x).

This page is a small collection of findings and pitfalls discovered during Micrometer integration. 

* [Handy Bindings](#handy-bindings)
* [Know Your Gauges](#know-your-gauges)
* [Tags Hell](#tags-hell)

## Handy Bindings

* * *

Micrometer goes with a bunch of handy pre-configured 
[bindings](https://github.com/micrometer-metrics/micrometer/tree/master/micrometer-core/src/main/java/io/micrometer/core/instrument/binder) 
that are not very well covered in official documentation. 
Binders can provide insights about application internals (system, database, jvm etc.) with minimum configuration required. 

As an example, the following `ExecutorService` instrumentation will provide metrics for internal tasks-queue and thread-pool:
```kotlin
fun monitorExecutor(meterRegistry: MeterRegistry, 
                    executor: ExecutorService, 
                    executorName: String): ExecutorService = 
    ExecutorServiceMetrics.monitor(meterRegistry, executor, executorName)
```

Many of these metrics are available and registered 
[out-of-the-box]((https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready-metrics-meter)) 
using Spring Boot.

## Know Your Gauges

* * *

If you migrate to Micrometer from StatsDClient you might expect that all you need to do is to replace `statsDClient` with 
`meterRegistry`:

```kotlin
// StatsDClient
fun incCounter(metricName: String, value: Double) = 
    statsDClient.increment(metricName, value)

// Micrometer
fun incCounter(metricName: String, value: Double) = 
    Counter.builder(metricName)
           .register(meterRegistry)
           .increment(value);
```

This works for counters and histograms, but not for gauges. 
You can find a warning in Micrometer documentation that says you can not use primitive numbers with Micrometer gauges:
![Image of Gauge Warning](/assets/img/gauge-warning.png)

```kotlin
fun main() {
    val metricName = "my-gauge"
    val meterRegistry = SimpleMeterRegistry()

    meterRegistry.gauge(metricName, 12.0)
    println(meterRegistry.find(metricName).gauge()!!.value()) // 12.0

    meterRegistry.gauge(metricName, 11.0)
    println(meterRegistry.find(metricName).gauge()!!.value()) // expected 11.0 but got 12.0
}
```

From Micrometer [documentation](https://micrometer.io/docs/concepts#_gauges):
> A gauge can be made to track any java.lang.Number subtype that is settable, 
> such as AtomicInteger and AtomicLong found in java.util.concurrent.atomic 
> and similar types like Guava’s AtomicDouble.

This means gauge metric value can be changed only via instance of settable type (atomic) that was used during first metric registration. 
Compare the following scenarious:

```kotlin
fun main() {
    val metricName = "my-gauge"
    val meterRegistry = SimpleMeterRegistry()

    meterRegistry.gauge(metricName, AtomicDouble(12.0))
    println(meterRegistry.find(metricName).gauge()!!.value()) // 12.0

    // reference to the original value is not preserved 
    // gauge value change ignored
    meterRegistry.gauge(metricName, AtomicDouble(11.0))
    println(meterRegistry.find(metricName).gauge()!!.value()) // expected 11.0 but still got 12.0
}
```

```kotlin
fun main() {
    val metricName = "my-gauge"
    val meterRegistry = SimpleMeterRegistry()
    val value = AtomicDouble()

    meterRegistry.gauge(metricName, value)

    value.set(12.0)
    println(meterRegistry.find(metricName).gauge()!!.value()) // 12.0

    // change gauge value via original reference
    value.set(11.0)
    println(meterRegistry.find(metricName).gauge()!!.value()) // 11.0
}

```

For long-running services we can use the following gauges cache to keep mapping between metric and appropriate atomic-value reference:
```kotlin
class GaugesCache {

    /**
     * Update or create a gauge metric with provided 
     * value associated with metric name-tags pair.
     */
    fun upsert(metricName: String, 
               metricTags: Tags = Tags.empty(), 
               value: Double): AtomicDouble {
        val atomic = gaugesCache.computeIfAbsent(GaugeCacheKey(metricName, metricTags)) { AtomicDouble() }
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

```kotlin
fun main() {
    val metricName = "my-gauge"
    val meterRegistry = SimpleMeterRegistry()
    val gaugesCache = GaugesCache()

    meterRegistry.gauge(metricName, gaugesCache.upsert(metricName, value = 12.0))
    println(meterRegistry.find(metricName).gauge()!!.value()) // 12.0

    meterRegistry.gauge(metricName, gaugesCache.upsert(metricName, value = 11.0))
    println(meterRegistry.find(metricName).gauge()!!.value()) // 11.0
}
```

## Tags Hell

* * *

The following memory-leak was observed in one of our services: 
![Memory Leak](/assets/img/memory-leak.png)

After investigation we found the following code that tracks http-call response durations:
```kotlin
fun trackResponseDuration(path: String, method: String, duration: Duration) =
    meterRegistry.timer(
        "http.response_duration",
        Tags.of(
            Tag.of("path", path),
            Tag.of("http_method", method)))
        .record(duration)
```

Within every invocation of `trackResponseDuration` Micrometer creates a histogram based on 
metric name (`http.response_duration`) and provided tags (`path` and `method`). 
Each histogram is uniquely identified by metric name and set of tag key/values, 
therefore any new unique combination of tag key/values creates a new instance of histogram 
that remains in memory as long as service instance is running (Micrometer design).

Having unbounded set of tag values results into constantly increasing memory consumption. 
In method above some paths contained UUIDs, e.g. `/api/internal/cards/{cards_uuid}/status`. 
This caused rapid growth of heap memory consumption (linear to number of unique UUIDs in the path) 
and eventually resulted into OOM and application crash.

![Unbounded Metrics Set](/assets/img/unbounded_metrics.png)

Keep in mind metrics are not intended to track fine-grained information or statistics of a service 
(such as specific requests durations etc.). Use logs or distributed tracing instead. 
Metrics must provide aggregated view on service internals.

Unbounded number of histograms results into unbounded number of time series in durable metrics storage. 
Some storage providers may bill you for every serie created. Big number of time series also complicates real-time aggregations.

Worth to mention that everything mentioned above also applies to counters and gauges.
