# Swift Service Context

* Proposal: [SSWG-0024](0024-service-context.md)
* Authors: [Konrad 'ktoso' Malawski](https://github.com/ktoso) / [Moritz Lang](https://github.com/slashmo)
* Review Manager: Adam Fowler
* Status: **Implemented**
* Implementation: 
  * [swift-service-context](https://github.com/apple/swift-service-context)
* Forum Threads: 
  * [Pitch](https://forums.swift.org/t/pitch-swift-service-context/66687)
  * [Proposal](https://forums.swift.org/t/sswg-0024-service-context/67121)

## Package Description

ServiceContext is a minimal (zero-dependency) context propagation container, intended to "carry" items for purposes of cross-cutting tools to be built on top of it.

It is modeled after the concepts explained in W3C Baggage and in the spirit of Tracing Plane's "Baggage Context" type, although by itself, it does not define a specific serialization format.

See https://github.com/apple/swift-distributed-tracing for actual instrument types and implementations, which can be used to deploy various cross-cutting instruments, all reusing the same service context type. 
Refer to distributed tracing [documentation](https://swiftpackageindex.com/apple/swift-distributed-tracing/main/documentation/tracing) to learn more about how it is used in practice.

| | |
|--|--|
| **Package Name** | `swift-service-context` |
| **Module Name** | `ServiceContextModule` |
| **Proposed Maturity Level** | [Incubating](https://www.swift.org/sswg/incubation-process.html#process-diagram) |
| **License** | [Apache v2](https://github.com/apple/swift-distributed-tracing/blob/main/LICENSE.txt) |
| **Dependencies** | - |

## Introduction

Service context is a type-safe dictionary that is keyed using types and intended to be used for contextual information propagation across asynchronous code.

It declares a well-known Task Local key for the service context, and patterns how to add keys. 

It is a core building block of [swift-distributed-tracing](https://github.com/apple/swift-distributed-tracing) which uses it to propagate trace information across swift concurrency tasks, though it can be used independently as well.

## Overview

Service context's API surface is relatively small and is all based around the service context type which is a type-safe dictionary.

One can create a service context using the `.topLevel` factory method which intends to clarify that this should not be used UNLESS intending to start a fresh "top level" context that does not inherit any values from the current context:

```swift
var context = ServiceContext.topLevel
```

In most situations, developers should prefer to "pick up" the context from the current context:

```swift
var context: ServiceContext? = ServiceContext.current
```

which inspects the task-local values for the presence of a service context. It is by design that it is possible to determine if no context was set, or if an empty context is set. When intending to "carry existing, or create a new" context, the following pattern is used:

```swift
var context = ServiceContext.current ?? ServiceContext.topLevel
```

Once obtained, one can set values in by using defined keys, like this:

```swift
/// Keys should be defined private and only exposed though accessors, as seen below.
private enum FirstTestKey: ServiceContextKey {
  typealias Value = Int
}

/// Define convenience accessors, in order to keep the key type private.
extension ServiceContext {
    public var firstTest: String? {
        set {
            self[FirstTestKey.self] = newValue
        }
        get {
            self[FirstTestKey.self]
        }
    }
}

var context = ServiceContext.topLevel
context.firstTest = 42
```

Keys declared as types conforming to `ServiceContextKey` allow for future extension where we might want to configure specific keys using additional behavior, like for example defining the `static let nameOverride` of a key. It is also important that a Key type may be private to whomever declares it, allowing only such library to set these values. 

Service context is used primarily as a task-local value read and modified like this:

```swift
func exampleFunction() async -> String {
  guard let context = ServiceContext.current {
    return "no-service-context"
  }
  guard let value = context.firstTest {
    return "no test value"
  }
  print("test value = \(value)") // test value = test-value
  return value
}

// ----------------------------------------

var context = ServiceContext.topLevel
context.firstTest = "test-value"

let ok = ServiceContext.withValue(context) {
    await exampleFunction()
    assert(ServiceContext.current?.firstTest == Optional("test-value"))
    return "ok" // withValue can return values
}
assert(ServiceContext.current?.firstTest == Optional.none) // value is not set outside withValue block
assert(c == "ok")
```

Swift's [task local values](https://developer.apple.com/documentation/swift/tasklocal) are used to automatically propagate the context value to any child tasks that may be created from this code, such that context is automatically propagated through to them, e.g. like this:

```swift
func testMeMore() -> String {
  guard let context = ServiceContext.current {
    return "no-service-context"
  }
  guard let value = context.firstTest {
    return "no test value"
  }
  return value
}

func test() async -> String {
  async let v = testMeMore() // read context from child task
  return await v
}

var context = ServiceContext.topLevel
context.firstTest = "test-value"
ServiceContext.withValue(context) {
    assert(test() == "test-value")
}
```

It is recommended to follow this pattern to declare an accessor on the context, rather than exposing the `Key` type directly. This also allows to add validation and change the Key type used to persist the value in the future, in addition to being also a nicer user-experience for developers.

```swift
extension ServiceContext { 
  public var testValue: String? {
    set { 
      self[TestKey.self] = newValue
    }
    get {
      self[TestKey.self]
    }
  }
}
```

## Maturity Justification

We are proposing this package at the "Incubation" level of maturity. We believe this package to be an important building block of the server ecosystem, in the same way [swift-log](https://github.com/apple/swift-log) and [swift-metrics](https://github.com/apple/swift-metrics) are adopted across many server and client libraries.

The project has matured for over 3 years, has multiple active maintainers and fulfills adoption requirements in production.

Minimum Requirements:

* General
  * Has relevance to Swift on Server specifically: **Yes, and is planned to be adopted by core server and client libraries**
  * Publicly accessible source managed by an SCM such as github.com or similar: **Satisfied: Repository is stored on github.com**
  * Prefer to use `main` as the default branch name, in line with [Swift's guidelines](https://forums.swift.org/t/moving-default-branch-to-main/38515): **Satisfied**
  * Adopt the [Swift Code of Conduct](https://swift.org/community/#code-of-conduct): **Satisfied**
* Ecosystem
  * Uses SwiftPM: **Satisfied**
  * Integrated with critical SSWG ecosystem building blocks, e.g., Logging and Metrics APIs, SwiftNIO for IO: **It is such a building block.**
* Longevity
  * Must be from a team that has more than one public repository (or similar indication of experience): **Satisfied: hosted under Apple organization on GitHub; Active maintainers are @ktoso and @slashmo**
  * SSWG should have access / authorization to graduated repositories in case of emergency: **Satisfied (Konrad, Franz, Tomer)**
  * Adopt the [SSWG Security Best Practices](../security/README.md)): **Pending**
* Testing, CI and Release
  * Have unit tests for Linux: **Satisfied**
  * CI setup, including testing PRs and the main branch: **Satisfied**
  * Follow semantic versioning, with at least one published pre-release (e.g. 0.1.0, 1.0.0-beta.1) or release (e.g. 1.0.0): **Satisfied**
* Licensing
  * Apache 2, MIT, or BSD (Apache 2 recommended): **Satisfied: Apache 2**
* Conventions and Style
  * Adopt [Swift API Design Guidelines](https://swift.org/documentation/api-design-guidelines/): **Satisfied**
  * Follow [SSWG Technical Best Practices](https://www.swift.org/sswg/incubation-process.html#technical-best-practices) when applicable: **Satisfied**
  * Prefer to adopt code formatting tools and integrate them into the CI: **Satisfied**
Incubating Requirements:
* Document that it is being used successfully in production by at least two independent end users which, in the SSWG judgment, are of adequate quality and scope.
  * **We are aware of 3+ production use-cases in large deployments using [swift-distributed-tracing](https://github.com/apple/swift-distributed-tracing).**
* Must have 2+ maintainers and/or committers. 
  * **Actively maintained by Moritz ([@slashmo](https://github.com/slashmo)) and Konrad ([@ktoso](https://github.com/ktoso))**
* Packages must have more than one person with admin access. This is to avoid losing access to any packages. For packages hosted on GitHub and GitLab, the packages must live in an organization with at least two administrators. If you don't want to create an organization for the package, you can host them in the [Swift Server Community](https://github.com/swift-server-community) organization.
  * **SSWG members Franz, Konrad have admin rights, and the project is under the Apple organization which has multiple admins.**
* Demonstrate an ongoing flow of commits and merged contributions, or issues addressed in timely manner, or similar indication of activity.
  * **Active development for years, see changelog. Project expected to have less changes now that a 1.0 has been announced.**

## Alternatives considered

Not building a tracing API was considered but we see this as a fundamental building block for the server ecosystem, in the same style as swift-log and swift-metrics. 
