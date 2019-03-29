# Server Metrics API

* Proposal: [SSWG-0002](https://github.com/swift-server/sswg/blob/master/proposals/SSWG-0002.md)
* Authors: [Tomer Doron](https://github.com/tomerd)
* Sponsors: Apple
* Review Manager: [Tanner Nelson](https://github.com/tanner0101)
* Status: [Accepted as Sandbox Maturity](https://forums.swift.org/t/april-4th-2019/22704)
* Implementation: [tomerd/swift-server-metrics-api-proposal](https://github.com/tomerd/swift-server-metrics-api-proposal/)
* Pitch: [Server/Pitches/Metrics](https://forums.swift.org/t/metrics/19353)
* Review: [Initial Discussion](https://forums.swift.org/t/discussion-server-metrics-api/19600/9), [Feedback Review](https://forums.swift.org/t/feedback-server-metrics-api/21353)

## Package Description

|  |  |
|--|--|
| **Package Name** | `swift-metrics` |
| **Proposed Maturity Level** | [Sandbox](https://github.com/swift-server/sswg/blob/master/process/incubation.md#process-diagram) |
| **License** | [Apache 2](https://www.apache.org/licenses/LICENSE-2.0.html) |
| **Dependencies** | *none* |

## Introduction

Almost all production server software needs to emit metrics information for observability. The SSWG aims to provide a number of packages that can be shared across the whole Swift Server ecosystem so we need some amount of standardization. Because it's unlikely that all parties can agree on one full metrics implementation, this proposal is attempting to establish a metrics API that can be implemented by various metrics backends which then post the metrics data to backends like prometheus, graphite, publish over statsd, write to disk, etc.

## Motivation

As outlined above, we should standardize on an API that if well adopted would allow application owners to mix and match libraries from different parties with a consistent metrics collection solution.

## Proposed solution

The proposed solution is to introduce the following types that encapsulate metrics data:

`Counter`: A counter is a cumulative metric that represents a single monotonically increasing counter whose value can only increase or be reset to zero on restart. For example, you can use a counter to represent the number of requests served, tasks completed, or errors.

```swift
counter.increment(100)
```

`Recorder`: A recorder collects observations within a time window (usually things like response sizes) and can provides aggregated information about the data sample, for example count, sum, min, max and various quantiles.

```swift
recorder.record(100)
```

`Gauge`: A Gauge is a metric that represents a single numerical value that can arbitrarily go up and down. Gauges are typically used for measured values like temperatures or current memory usage, but also "counts" that can go up and down, like the number of active threads. Gauges are modeled as `Recorder` with a sample size of 1 and that does not perform any aggregation.

```swift
gauge.record(100)
```

`Timer`: A timer collects observations within a time window (usually things like request durations) and provides aggregated information about the data sample, for example min, max and various quantiles. It is similar to a `Recorder` but specialized for values that represent durations.

```swift
timer.recordMilliseconds(100)
```

How would you use `counter`, `recorder`, `gauge` and `timer` in you application or library? Here is a contrived example for request processing code that emits metrics for: total request count per url, request size and duration and response size:

```swift
func processRequest(request: Request) -> Response {
    let requestCounter = Counter("request.count", ["url": request.url])
    let requestTimer = Timer("request.duration", ["url": request.url])
    let requestSizeRecorder = Recorder("request.size", ["url": request.url])
    let responseSizeRecorder = Recorder("response.size", ["url": request.url])

    requestCounter.increment()
    requestSizeRecorder.record(request.size)

    let start = Date()
    let response = ...
    requestTimer.record(Date().timeIntervalSince(start))
    responseSizeRecorder.record(response.size)
}
```

## Detailed design

As seen above, the constructor functions `Counter`, `Timer`, `Gauge` and `Recorder` provides a concrete metric object. This raises the question of what metrics backend will you actually get? The answer is that it's configurable _per application_. The application sets up the metrics backend it wishes the whole application to use when it first starts. Libraries should never change the metrics implementation as that is something owned by the application. Configuring the metrics backend is straightforward:

```swift
MetricsSystem.bootstrap(MyFavoriteMetricsImplementation())
```

This instructs the `MetricsSystem` to install `MyFavoriteMetricsImplementation` as the metrics backend to use. This can only be done once at the beginning of the program.

### Metrics Types

#### Counter

Following is the user facing `Counter` API. It must have reference semantics, and its behavior depends on the `CounterHandler` implementation.

```swift
public class Counter: CounterHandler {
    var handler: CounterHandler

    public let label: String
    public let dimensions: [(String, String)]

    public init(label: String, dimensions: [(String, String)], handler: CounterHandler)

    /// Increments by 1
    public func increment()
    public func increment<DataType: BinaryInteger>(_ value: DataType)
}
```

#### Recorder

Following is the user facing `Recorder` API. It must have reference semantics, and its behavior depends on the `RecorderHandler` implementation.

```swift
public class Recorder: RecorderHandler {
    var handler: RecorderHandler

    public let label: String
    public let dimensions: [(String, String)]
    public let aggregate: Bool

    public init(label: String, dimensions: [(String, String)], aggregate: Bool, handler: RecorderHandler)

    public func record<DataType: BinaryInteger>(_ value: DataType)
    public func record<DataType: BinaryFloatingPoint>(_ value: DataType)
}
```

#### Gauge

`Gauge` is a specialized `Recorder` that does not preform aggregation.

```swift
public class Gauge: Recorder {
    public convenience init(label: String, dimensions: [(String, String)] = []) {
        self.init(label: label, dimensions: dimensions, aggregate: false)
    }
}
```

#### Timer

Following is the user facing `Timer` API. It must have reference semantics, and its behavior depends on the `TimerHandler` implementation.

```swift
public class Timer: TimerHandler {
    var handler: TimerHandler
    
    public let label: String
    public let dimensions: [(String, String)]

    public init(label: String, dimensions: [(String, String)], handler: TimerHandler)

    public func recordNanoseconds(_ duration: Int64)
}
```

## Implementing a metrics backend (eg prometheus client library)

An implementation of a metric backend needs to conform to the `MetricsFactory` protocol:

```swift
public protocol MetricsFactory {
    func makeCounter(label: String, dimensions: [(String, String)]) -> CounterHandler
    func makeRecorder(label: String, dimensions: [(String, String)], aggregate: Bool) -> RecorderHandler
    func makeTimer(label: String, dimensions: [(String, String)]) -> TimerHandler
}
```

Having `CounterHandler`, `TimerHandler` and `RecorderHandler` define the metric capturing API:

```swift
public protocol CounterHandler: AnyObject {
    func increment<DataType: BinaryInteger>(_ value: DataType)
}

public protocol TimerHandler: AnyObject {
    func recordNanoseconds(_ duration: Int64)
}

public protocol RecorderHandler: AnyObject {
    func record<DataType: BinaryInteger>(_ value: DataType)
    func record<DataType: BinaryFloatingPoint>(_ value: DataType)
}
```

Here is an example of contrived in-memory implementation:

```swift
class SimpleMetrics: MetricsFactory {
    init() {}

    func makeCounter(label: String, dimensions: [(String, String)]) -> CounterHandler {
        return ExampleCounter(label, dimensions)
    }

    func makeRecorder(label: String, dimensions: [(String, String)], aggregate: Bool) -> RecorderHandler {
        let maker:(String,  [(String, String)]) -> Recorder = aggregate ? ExampleRecorder.init : ExampleGauge.init
        return maker(label, dimensions)
    }

    func makeTimer(label: String, dimensions: [(String, String)]) -> TimerHandler {
        return ExampleTimer(label, dimensions)
    }

    private class ExampleCounter: CounterHandler {
        init(_: String, _: [(String, String)]) {}

        let lock = NSLock()
        var value: Int64 = 0
        func increment<DataType: BinaryInteger>(_ value: DataType) {
            self.lock.withLock {
                self.value += Int64(value)
            }
        }
    }

    private class ExampleRecorder: RecorderHandler {
        init(_: String, _: [(String, String)]) {}

        private let lock = NSLock()
        var values = [(Int64, Double)]()
        func record<DataType: BinaryInteger>(_ value: DataType) {
            self.record(Double(value))
        }

        func record<DataType: BinaryFloatingPoint>(_ value: DataType) {
            // this may loose precision, but good enough as an example
            let v = Double(value)
            // TODO: sliding window
            lock.withLock {
                values.append((Date().nanoSince1970, v))
                self._count += 1
                self._sum += v
                self._min = min(self._min, v)
                self._max = max(self._max, v)
            }
        }

        var _sum: Double = 0
        var sum: Double {
            return self.lock.withLock { _sum }
        }

        private var _count: Int = 0
        var count: Int {
            return self.lock.withLock { _count }
        }

        private var _min: Double = 0
        var min: Double {
            return self.lock.withLock { _min }
        }

        private var _max: Double = 0
        var max: Double {
            return self.lock.withLock { _max }
        }
    }

    private class ExampleGauge: RecorderHandler {
        init(_: String, _: [(String, String)]) {}

        let lock = NSLock()
        var _value: Double = 0
        func record<DataType: BinaryInteger>(_ value: DataType) {
            self.record(Double(value))
        }

        func record<DataType: BinaryFloatingPoint>(_ value: DataType) {
            // this may loose precision but good enough as an example
            self.lock.withLock { _value = Double(value) }
        }
    }

    private class ExampleTimer: ExampleRecorder, TimerHandler {
        func recordNanoseconds(_ duration: Int64) {
            super.record(duration)
        }
    }
}
```
