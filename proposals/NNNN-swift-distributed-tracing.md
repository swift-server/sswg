# Swift Distributed Tracing

* Proposal: [SSWG-NNNN](https://github.com/swift-server/sswg/blob/main/proposals/NNNN-tracing-api.md)
* Authors: [Moritz Lang](https://github.com/slashmo), [Konrad 'ktoso' Malawski](https://github.com/ktoso)
* Review Manager: TBD
* Status: **Implemented**
* Implementation: [apple/swift-distributed-tracing](https://github.com/apple/swift-distributed-tracing)
* Forum Threads:
  * [Pitch](https://forums.swift.org/t/pitch-swift-distributed-tracing/66679)

## Package Description

A Distributed Tracing API for Swift, enabling the instrumentation of server side applications
using tools such as tracers.

|    |    |
| -- | -- |
| **Package Name** | `swift-distributed-tracing` |
| **Module Names** | `Tracing`, `Instrumentation` |
| **Proposed Maturity Level** | [Incubating](https://github.com/swift-server/sswg/blob/main/process/incubation.md#process-diagram) |
| **License** | [Apache 2.0](https://github.com/apple/swift-distributed-tracing/blob/main/LICENSE.txt) |
| **Dependencies** | [swift-service-context](https://github.com/apple/swift-service-context) |

## Introduction

Swift Distributed Tracing provides a common foundation that allows to freely choose how to instrument systems
with minimal changes to your actual code.

## Motivation

Whereas [Logging](https://github.com/apple/swift-log) and [Metrics](https://github.com/apple/swift-metrics) enables
you to instrument specific parts of an application,
[Distributed Tracing](https://github.com/apple/swift-distributed-tracing) provides a holistic view on an entire
distributed system. Together with these two, Distributed Tracing will complete the "three pillars of observability".

As with Logging and Metrics, the community will benefit most from Distributed Tracing if it is implemented in
libraries and frameworks directly using one common API. End-users should be able to freely choose a suitable backend
implementation without having to change the libraries or frameworks they're using.

## Proposed Solution

Swift Distributed Tracing revolves around creating **spans**, which together form a tree-like structure. A trace can be
made up by spans recorded in a single service, or propagated across multiple services. Swift Distributed Tracing uses
the task-local based [Swift Service Context](https://github.com/apple/swift-service-context) to implement propagation
transparently, without the need of manual context passing.

Our proposed solution is a library which targets three "roles":

- End-users
- Library & framework authors
- Tracer backend implementations

### End-users

End-users are the ones benefiting from Distributed Tracing. They choose a Tracing backend that suits their needs, use
libraries that have built-in Swift Distributed Tracing support, and augment their own code with manual instrumentation.

### Library & framework authors

Libraries/Frameworks such as HTTP servers/clients, database libraries, etc. know best how to instrument their library's
internals. They implement generic tracing support using the Swift Distributed Tracing API without a specific tracing
backend in mind.

#### Examples

- [Hummingbird](https://github.com/hummingbird-project/hummingbird/releases/tag/1.6.0)
- [Soto](https://github.com/soto-project/soto-core/pull/575)

### Tracer backend implementations

The final piece of the puzzle are tracer backend implementations. They provide vendor-specific support for exporting
tracing spans.

#### Examples

[Swift OTel](https://github.com/slashmo/swift-otel) exposes a tracer that exports to
an [OpenTelemetry Collector](https://opentelemetry.io/docs/collector). This already allows adopters of this tracing
library to export to popular backends compatible with OpenTelemetry, such as Zipkin, Jaeger, Honeycomb, and more.

## Detailed design

Both library/framework authors and end-users use the same API to create tracing spans. Adding a span to functions can
be achieved by using the `withSpan(operationName:context:body:)` API:

```swift
func performOperation() -> String {
    withSpan("operation") {
        return "handled"
    }
}
```

### Customizing the span

A span can be further customized using `withSpan(_:context:ofKind:function:file:line:operation:)` as well. In the example below we're adding a custom attribute (`"http.method"`) to the span, so when viewing it in a trace visualization backend we can have more detailed information about what kind of request this was:

```swift
func handleRequest(_ request: Request) -> Response {
    withSpan(request.endpointPath, ofKind: .server) { span in
        span.attributes["http.method"] = request.method.rawValue
        return respond(to: request)
    }
}
```

### Recording errors

Throwing can be handled by either recording errors manually into a span by calling `span.recordError(error:)`,
or by wrapping a potentially throwing operation using the `withSpan(operationName:context:body:)` function,
which automatically records any thrown error.

```swift
func handleRequest(_ request: Request) throws -> Response {
    try withSpan(request.endpointPath, ofKind: .server) {
        try respond(to: request)
    }
}
```

### Swift Concurrency

Swift Distributed Tracing ships both synchronous and asynchronous versions of `withSpan(operationName:context:body:)`.
In both cases, spans are automatically ended at the end of the body closure scope:

```swift
func handleRequest(_ request: Request) async throws -> Response {
    try await withSpan(request.endpointPath, ofKind: .server) {
        try await respond(to: request)
    }
}
```

### Child Spans

A distributed tracing span may have one or more child spans. While Swift Distributed Tracing also allows for manually
linking spans, creating a child span is easiest when using the
task-local [`ServiceContext`](https://github.com/apple/swift-service-context) behind the scenes:

```swift
func handleRequest(_ request: Request) async throws -> Response {
    try await withSpan(request.endpointPath, ofKind: .server) {
        let user = try await fetchUser(byID: request.userID)
        return try await respond(to: request)
    }
}

func fetchUser(byID id: String) async throws -> User {
    try await withSpan("fetchUser") { span in
        span.attributes["user.id"] = id
    }
}
```

In this example, the `withSpan(operationName:context:body:)` call within `handleRequest(_:)` creates a span.
When creating the span, the tracer implementation sets trace and span identifiers on the task-local
[ServiceContext](https://github.com/apple/swift-service-context). `fetchUser(id:)` will then automatically "inherit"
these identifiers, indicating to the tracer implementation that the "fetchUser" span should be a child of the request
span.

### Low-level API

When using Swift NIO `EventLoopFuture`s or similar patterns, we need to control the spans lifetime manually. 
The service context will also need to be propagated manually by passing it to the context parameter of startSpan, 
because event loop futures cannot automatically propagate task local values.
This can be
achieved using the lower-level `startSpan(operationName:context:)` API:

```swift
func handleRequest(_ request: Request, context: ServiceContext) -> EventLoopFuture<Response> {
    // Start the span manually with startSpan passing in the `context` explicitly 
    let span = startSpan(request.endpointPath, context: context, ofKind: .server)

    // Chain an "always" onto the response future since we must ALWAYS end() a span
    return response(to: request).always { result in
        switch result {
        case .success(let response):
            span.attributes["http.status_code"] = response.status.code
        case .failure(let error):
            // If the response has failed, mark this span as failed by recording an error
            span.recordError(error)
        }

        // ALWAYS end a span, as otherwise its resources held by the tracer for this span may leak
        span.end()
    }
}
```

### Context Propagation

Distributed Tracing really shines when being used in a distributed system, having traces that span across multiple
nodes. We support this in Swift Distributed Tracing via the `Injector`/`Extractor` protocol.
Libraries/frameworks that send data across asynchronous boundaries such as HTTP servers/clients create
transport-specific injectors and extractors to pass the current
[`ServiceContext`](https://github.com/apple/swift-service-context) around the system:

```swift
struct HTTPHeadersInjector: Injector {
    func inject(_ value: String, forKey name: String, into headers: HTTPHeaders) {
        headers.add(name: name, value: value)
    }
}
```

```swift
struct HTTPHeadersExtractor: Extractor {
    func extract(key name: String, from headers: HTTPHeaders) -> String? {
        headers.first(name: name)
    }
}
```

The libraries/frameworks then call these injectors/extractors right before or after crossing an asynchronous boundary:

```swift
func execute(_ request: HTTPRequest) async throws -> HTTPResponse {
    try await withSpan("HTTP \(request.method)", ofKind: .client) { span in
        var request = request
        InstrumentationSystem.instrument.inject(
            span.context,
            into: request.headers,
            using: HTTPHeadersInjector()
        )
        return try await execute(request)
    }
}
```

```swift
func handleRequest(_ request: Request) async throws -> Response {
    var context = ServiceContext.topLevel
    InstrumentationSystem.instrument.extract(
        request.headers,
        into: context,
        using: HTTPHeadersExtractor()
    )
    try await withSpan(request.endpointPath, context: context) { span in
        try await respond(to: request)
    }
}
```

In this client/server example, the server span would automatically become a child of the client span, both of which
belong to a single trace.

### Log Correlation

Distributed Tracing uses identifiers such as trace- and span-ids to identify individual spans.
Swift Distributed Tracing supports the correlation of [swift-log](https://github.com/apple/swift-log) logs via
[Logging MetadataProviders](https://github.com/apple/swift-log/blob/main/proposals/0001-metadata-providers.md) by
including these identifiers in log statements.

Log correlation allows you to easily go from the bigger picture to very detailed log statements and vice versa.

```swift
func handleRequest(request: Request) -> Response {
    withSpan(request.operationName, ofKind: .server) {
        logger.info("Handle request", metadata: ["route": "\(request.route)"])
        // 2023-08-08T09:41:00 INFO [route=/example trace_id=123 span_id=456] Handle request
    }
}
```

## Maturity Justification

We are proposing this package at the "Incubation" level of maturity. We believe this package to be an important building
block of the server ecosystem, in the same way [swift-log](https://github.com/apple/swift-log) and
[swift-metrics](https://github.com/apple/swift-metrics) are adopted across many server and client libraries.

The project has matured for over 3 years, has multiple active maintainers and
fulfills adoption requirements in production.

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
  * Must be from a team that has more than one public repository (or similar indication of experience): **Satisfied: hosted under Apple organization on GitHub**
  * SSWG should have access / authorization to graduated repositories in case of emergency: **Satisfied (Konrad, Franz, Tomer)**
  * Adopt the [SSWG Security Best Practices](../security/README.md)): **Pending**
* Testing, CI and Release
  * Have unit tests for Linux: **Satisfied**
  * CI setup, including testing PRs and the main branch: **Satisfied**
  * Follow semantic versioning, with at least one published pre-release (e.g. 0.1.0, 1.0.0-beta.1) or release (e.g. 1.0.0): **Satisfied**
* Licensing
  * Apache 2, MIT, or BSD (Apache 2 recommended): **Satisfied: Apache 2**
* Conventions and Style
  * Adopt [Swift API Design Guidelines](https://swift.org/documentation/api-design-guidelines/)
  * Follow [SSWG Technical Best Practices](#technical-best-practices) when applicable.
  * Prefer to adopt code formatting tools and integrate them into the CI: **Satisfied**

Incubating Requirements:

* Document that it is being used successfully in production by at least two independent end users which, in the SSWG judgment, are of adequate quality and scope.
  * We are aware of 3+ production use-cases in large deployments, using [swift-otel](https://github.com/slashmo/swift-otel) as a backend.
* Must have 2+ maintainers and/or committers. In this context, a committer is an individual who was given write access to the codebase and actively writes code to add new features and fix any bugs and security issues. A maintainer is an individual who has write access to the codebase and actively reviews and manages contributions from the rest of the project's community. In all cases, code should be reviewed by at least one other individual before being released.
  * **Actively maintained by Moritz ([@slashmo](https://github.com/slashmo) and Konrad ([@ktoso](https://github.com/ktoso))**
* Packages must have more than one person with admin access. This is to avoid losing access to any packages. For packages hosted on GitHub and GitLab, the packages must live in an organization with at least two administrators. If you don't want to create an organization for the package, you can host them in the [Swift Server Community](https://github.com/swift-server-community) organization.
  * **SSWG members Franz, Konrad have admin rights, and the project is under the Apple organization which has multiple admins.**
* Demonstrate an ongoing flow of commits and merged contributions, or issues addressed in timely manner, or similar indication of activity.
  * **Active development for years, see changelog. Project expected to have less changes now that a 1.0 has been announced.**

## Alternatives considered

Not building a tracing API was considered but we see this as a fundamental building block for the server ecosystem,
in the same style as [swift-log](https://github.com/apple/swift-log) and
[swift-metrics](https://github.com/apple/swift-metrics).
