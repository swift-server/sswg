# Swift Service Lifecycle

* Proposal: SSWG-0015
* Authors: [Tomer Doron](https://github.com/tomerd)
* Review Manager: [Peter Adams](https://github.com/PeterAdams-A)
* Status: Active Review 7 Aug 2020 ..< 22 Aug 2020
* Implementation: [Swift Service Lifecycle](https://github.com/swift-server/swift-service-lifecycle)

## Package Description

Swift Service Lifecycle provides a basic mechanism to cleanly start up and shut down the application, freeing resources in order before exiting.
It also provides a `Signal`-based shutdown hook, to shut down on signals like `TERM` or `INT`.

|  |  |
|--|--|
| **Package Name** | `swift-service-lifecycle` |
| **Module Name** | `ServiceLifecycle` |
| **Proposed Maturity Level** | [Sandbox](https://github.com/swift-server/sswg/blob/master/process/incubation.md#process-diagram) |
| **License** | [Apache License 2.0](https://github.com/swift-server/swift-backtrace/blob/master/LICENSE.txt) |
| **Dependencies** | [Backtrace](https://github.com/swift-server/swift-backtrace), [SwiftLog](https://github.com/apple/swift-log), [SwiftMetrics](https://github.com/apple/swift-metrics)  |

## Introduction & Motivation

Most services have startup and shutdown workflow-logic which is often sensitive to failure and hard to get right. 
Startup sequences include actions like initializing thread pools, running data migrations, warming up caches, and other forms of state initialization before 
taking traffic or accepting events. 
Shutdown sequences include freeing up resources that hold on to file descriptors or other system resources that may leak if not cleared correctly.

Today, server applications and frameworks must find ways to address the need on their own, which could be error prone. 
To make things safer and easier, Service Lifecycle codifies this common need in a safe, reusable and framework-agnostic way. 
It is designed to be integrated with any server framework or directly in a server application’s main.

## Proposed solution

The main types in the library are `ServiceLifecycle` and `ComponentLifecycle`.

`ServiceLifecycle` is the most commonly used type.
It is designed to manage the top-level Application (Service) lifecycle,
and in addition to managing the startup and shutdown flows it can also set up `Signal` trap for shutdown and install backtraces.

`ComponentLifecycle` manages a state machine representing the startup and shutdown logic flow.
In larger Applications (Services) `ComponentLifecycle` can be used to manage the lifecycle of subsystems, such that `ServiceLifecycle` can start and shutdown `ComponentLifecycle`s.

### Registering items

`ServiceLifecycle` and `ComponentLifecycle` are containers for `LifecycleTask`s which need to be registered using a `LifecycleHandler` - a container for synchronous or asynchronous closures.

Synchronous handlers are defined as  `() throws -> Void`.

Asynchronous handlers defined are as  `(@escaping (Error?) -> Void) -> Void`.

`LifecycleHandler` comes with static helpers named `async` and `sync` designed to help simplify the registration call to:

```swift
let foo = ...
lifecycle.register(
    label: "foo",
    start: .sync(foo.syncStart),
    shutdown: .sync(foo.syncShutdown)
)
```

Or the async version:

```swift
let foo = ...
lifecycle.register(
    label: "foo",
    start: .async(foo.asyncStart),
    shutdown: .async(foo.asyncShutdown)
)
```

or, just shutdown:

```swift
let foo = ...
lifecycle.registerShutdown(
    label: "foo",
    .sync(foo.syncShutdown)
)
```
Or the async version:

```swift
let foo = ...
lifecycle.registerShutdown(
    label: "foo",
    .async(foo.asyncShutdown)
)
```

you can also register a collection of `LifecycleTask`s (less typical) using:

```swift
func register(_ tasks: [LifecycleTask])

func register(_ tasks: LifecycleTask...)
```

### Configuration

`ServiceLifecycle` initializer takes optional `ServiceLifecycle.Configuration` to further refine the `ServiceLifecycle` behavior:

* `logger`: Defines the `Logger` to work with.  By default, `Logger(label: "Lifecycle")` is used.

* `callbackQueue`: Defines the `DispatchQueue` on which startup and shutdown handlers are executed. By default, `DispatchQueue.global` is used.

* `shutdownSignal`: Defines what, if any, signals to trap for invoking shutdown. By default, `INT` and `TERM` are trapped.

* `installBacktrace`: Defines if to install a crash signal trap that prints backtraces. This is especially useful for applications running on Linux since Swift does not provide backtraces on Linux out of the box. This functionality is provided via the [Swift Backtrace](https://github.com/swift-server/swift-backtrace) library.

### Starting the lifecycle

Use the `start` function to start the application.
Start handlers passed using the `register` function will be called in the order the items were registered in.

`start` is an asynchronous operation.
If a startup error occurred, it will be logged and the startup sequence will halt on the first error, and bubble it up to the provided completion handler.

```swift
lifecycle.start { error in
    if let error = error {
        logger.error("failed starting \(self) ☠️: \(error)")
    } else {
        logger.info("\(self) started successfully 🚀")
    }
}
```

### Shutdown

The typical use of the library is to call on `wait` after calling `start`.

```swift
lifecycle.start { error in
  ...   
}
lifecycle.wait() // <-- blocks the thread
```

If you are not interested in handling start completion, there is also a convenience method:

```swift
lifecycle.startAndWait() // <-- blocks the thread
```

Both `wait` and `startAndWait` are blocking operations that wait for the lifecycle library to finish the shutdown sequence.
The shutdown sequence is typically triggered by the `shutdownSignal` defined in the configuration. By default, `INT` and `TERM` are trapped.

During shutdown, the shutdown handlers passed using the `register` or `registerShutdown` functions are called in the reverse order of the registration. E.g.

```
lifecycle.register("1", ...)
lifecycle.register("2", ...)
lifecycle.register("3", ...)
```

startup order will be 1, 2, 3 and shutdown order will be 3, 2, 1.

If a shutdown error occurred, it will be logged and the shutdown sequence will *continue* to the next item, and attempt to shut it down until all registered items that have been started are shut down.

In more complex cases, when `Signal`-trapping-based shutdown is not appropriate, you may pass `nil` as the `shutdownSignal` configuration, and call `shutdown` manually when appropriate. This is designed to be a rarely used pressure valve.

`shutdown` is an asynchronous operation. Errors will be logged and bubbled up to the provided completion handler.

### Complex Systems and Nesting of Subsystems

In larger Applications (Services) `ComponentLifecycle` can be used to manage the lifecycle of subsystems, such that `ServiceLifecycle` can start and shutdown `ComponentLifecycle`s.

In fact, since `ComponentLifecycle` conforms to `LifecycleTask`,
it can start and stop other `ComponentLifecycle`s, forming a tree. E.g.:

```swift
struct SubSystem {
    let lifecycle = ComponentLifecycle(label: "SubSystem")
    let subsystem: SubSubSystem

    init() {
        self.subsystem = SubSubSystem()
        self.lifecycle.register(self.subsystem.lifecycle)
    }

    struct SubSubSystem {
        let lifecycle = ComponentLifecycle(label: "SubSubSystem")

        init() {
            self.lifecycle.register(...)
        }
    }
}

let lifecycle = ServiceLifecycle()
let subsystem = SubSystem()
lifecycle.register(subsystem.lifecycle)

lifecycle.start { error in
    ...
}
lifecycle.wait()
```

### Compatibility with SwiftNIO Futures

[SwiftNIO](https://github.com/apple/swift-nio) is a popular networking library that among other things provides Future abstraction named `EventLoopFuture`.

Swift Service Lifecycle comes with a compatibility module designed to make managing SwiftNIO based resources easy.

Once you import `LifecycleNIOCompat` module, `LifecycleHandler` gains a static helper named `eventLoopFuture` designed to help simplify the registration call to:

```swift
let foo = ...
lifecycle.register(
    label: "foo",
    start: .eventLoopFuture(foo.start),
    shutdown: .eventLoopFuture(foo.shutdown)
)
```

or, just shutdown:

```swift
let foo = ...
lifecycle.registerShutdown(
    label: "foo",
    .eventLoopFuture(foo.shutdown)
)
```


## Maturity Justification

This is a new package.

## Alternatives considered

None
