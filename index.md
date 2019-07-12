[Micrometer](https://micrometer.io/) is an application metrics facade which is widely used and adopted by N26 teams.
It goes with a plenty of instrumentations for various metric datastores, bindings to different components of your services and is a default metrics collector in Spring Boot 2.x.
This page is a small collection of findings and pitfalls you may face during Micrometer integration. 

* Bindings
* [Know Your Gauges](#know-your-gauges)
* [Tags Hell](#tags-hell)

## Bindings

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