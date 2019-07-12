# Micrometer Tips

* [Know Your Gauges](#know-your-gauges)
* [Tags Hell](#tags-hell)

## Know Your Gauges

## Tags Hell

```kotlin
fun executeWithMetrics(path: String,
					   method: String): Try<Response<T>> {
        return metrics.measureTime(
            "http.outgoing",
            "path" to path,
            "method" to method
        ) {...}
}
```