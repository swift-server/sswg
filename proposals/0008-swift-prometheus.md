# SwiftPrometheus - Prometheus Metrics in Swift

* Proposal: SSWG-0008
* Authors: [Jari Koopman / LotU](https://github.com/MrLotU)
* Review Manager: TBD
* Status:  [Implemented ](https://github.com/MrLotU/SwiftPrometheus/tree/MetricsLib)
* Pitch: [Pitches/Prometheus](https://forums.swift.org/t/client-side-prometheus-implementation/18098/)

## Package Description
Prometheus client side implementation.

|  |  |
|--|--|
| **Package Name** | `SwiftPrometheus` |
| **Module Name** | `Prometheus` |
| **Proposed Maturity Level** | [Sandbox](https://github.com/swift-server/sswg/blob/master/process/incubation.md#process-diagram) |
| **License** | [Apache 2.0](https://github.com/MrLotU/SwiftPrometheus/blob/master/LICENSE) |
| **Dependencies** | swift-nio 1.x.x or 2.x.x<sup>[1](#footnote_1)</sup> - swift-metrics > 1.0.0 |

## Introduction

_For a background on metrics see the metrics proposal [discussion](https://forums.swift.org/t/discussion-server-metrics-api/19600) and [feedback](https://forums.swift.org/t/feedback-server-metrics-api/21353) thread._

Prometheus is one of the most widely used libraries for metrics in the serverside world. SwiftPrometheus is a client side implementation in Swift, with the ability to use it both connected to & separately from swift-metrics.

## Motivation

With Prometheus being one of the most widely used metric reporting tools, it's a buildstone that can not be left out in a serverside ecosystem. This package is created for everyone to use & build upon for their metric reporting.

## Detailed design

SwiftPrometheus works around one base class `PrometheusClient` and some metric types around it. The prometheus metric types are:
(from the prometheus docs)
* Counter - A  *counter*  is a cumulative metric that represents a single [monotonically increasing counter](https://en.wikipedia.org/wiki/Monotonic_function) whose value can only increase or be reset to zero on restart.
* Gauge - A  *gauge*  is a metric that represents a single numerical value that can arbitrarily go up and down.
* Histogram - A  *histogram*  samples observations (usually things like request durations or response sizes) and counts them in configurable buckets. It also provides a sum of all observed values.
* Summary - Similar to a  *histogram* , a  *summary*  samples observations (usually things like request durations and response sizes). While it also provides a total count of observations and a sum of all observed values, it calculates configurable quantiles over a sliding time window.

SwiftPrometheus provides fully featured implementations for all of them, including a thin wrapper around them for integration with `swift-metrics`.

### API Layout
Below section will lay out the public API of this package. For the internal APIs I would suggest you to read through the code on GitHub :smile:. This section is split up into two parts, using this library standalone, or using it integrated with the `swift-metrics` package. 

#### Without swift-metrics
To get started, initialise an instance of `PrometheusClient`
```swift
import Prometheus
let myProm = PrometheusClient()
```
Once done, you can use the `create*` APIs to create any of the above described metric types.

```swift
// MetricLabels is a helper type used to add labeled metrics.
struct MyCodable: MetricLabels {
   var thing: String = "*"
}

// - Counter
let counter = myProm.createCounter(forType: Int.self, named: "my_counter", helpText: "Just a counter", initialValue: 42, withLabelType: MyCodable.self)

counter.inc() // Increment by one
counter.inc(12) // Increment by a value
counter.inc(12, MyCodable(thing: "test")) // Increment a labeled counter

// - Gauge
let gauge = myProm.createGauge(forType: Int.self, named: "my_gauge", helpText: "Just a gauge", initialValue: 42, withLabelType: MyCodable.self)
gauge.inc() // Same APIs as Counter
gauge.dec() // Same APIs as `inc()` but reversed.
gauge.set(42) // Set the gauge to a specific value

// - Histogram
// Histograms use special labels, different than the Counter & Gauge
struct HistogramLabel: HistogramLabels {
   var le: String = ""
   let route: String

   init() {
       self.route = "*"
   }

   init(_ route: String) {
       self.route = route
   }
}

let histogram = myProm.createHistogram(forType: Double.self, named: "my_histogram", helpText: "Just a histogram", labels: HistogramLabel.self)

histogram.observe(123) // Observes a value

// - Summary
// Like Histograms, Summaries use different label types.
struct SummaryLabel: SummaryLabels {
   var quantile: String = ""
   let route: String

   init() {
       self.route = "*"
   }

   init(_ route: String) {
       self.route = route
   }
}

let summary = myProm.createSummary(forType: Double.self, named: "my_summary", helpText: "Just a summary", labels: SummaryLabel.self)

summary.observe(123) // Observes a value
```
Then, after you have some metric types, you can use `.collect()` on your `PromtheusClient` to get your Prometheus formatted string with all the data.
For example, in a Vapor app:
```swift
router.get("/metrics") { req -> String in 
    return myProm.collect()
}
```

#### With swift-metrics
For use with swift-metrics, most of the steps described above work the same. To bootstrap the MetricsSystem you create a client and feed it to `MetricsSystem`:
```swift
let myProm = PrometheusClient()
MetricsSystem.bootstrap(myProm)
```
After that, you can use the metric types used by swift-metrics for your metrics. The mapping is as follows:
| swift-metrics | SwiftPrometheus |
|--|--|
| Counter | Counter |
| Gauge | Gauge |
| Recorder (agg) | Histogram |
| Timer | Summary |

---

To get a hold of your `PrometheusClient` either to:
a) use custom prometheus behaviour; or
b) get your metrics output
there is a utility function on `MetricsSystem`
```swift
let myProm = try MetricsSystem.prometheus()
```
This will either return the `PrometheusClient` used with `.bootstrap()` or throw an error if `MetricsSystem` was not bootstrapped with `PrometheusClient`
*Note: There currently is no support for retrieving `PrometheusClient` when being used with `MultiplexMetricsHandler`* 

## Maturity Justification

The implementation has the full feature set required for production use and meets the minimum requirements set forth by the SSWG (except for the fact that I'm a 1 man army creating this library)

## Alternatives considered

Other than using a different metrics backend than Prometheus, there are not many alternatives to consider. One thing I'd like to point out though:

This library has support for the destroying of metrics in the way set forth by the `swift-metrics` package. However, as described in the Prometheus documentation, once a metric is created with a specific type, so for example a `Counter` named `my_counter` and that counter is destroyed, it's not allowed to, at a later time, re-create a metric named `my_counter` with a DIFFERENT type. (Creating another counter is fine). To keep track of this, `PrometheusClient` will hold a dictionary of metric names & types. (`[String: MetricType]`). This means that even if you destroy your metrics, your memory footprint will (gradually) increase. All of this is process bound and will reset on a process restart.


<a name="footnote_1">1</a>: For NIO 2 use SwiftPrometheus 1.x.x, for NIO 1 use SwiftPrometheus 0.x.x