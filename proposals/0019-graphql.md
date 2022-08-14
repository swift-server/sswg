# Solution name

* Proposal: [SSWG-0019](0019-graphql.md)
* Authors: [Jay Herron](https://github.com/NeedleInAJayStack)
* Review Manager: TBD
* Status: **Implemented**
* Implementation: [GraphQL](https://github.com/GraphQLSwift/GraphQL)
* Forum Threads: [Pitch](https://forums.swift.org/t/pitch-graphql/59554)

## Package Description
This library is a Swift port of the [JavaScript reference GraphQL implementation](https://github.com/graphql/graphql-js), written on top of Swift NIO. It provides full server-side instrumentation for defining GraphQL schemas, resolving queries, and encoding/decoding GraphQL messages.

|  |  |
|--|--|
| **Package Name** | `GraphQL` |
| **Module Name** | `GraphQL` |
| **Proposed Maturity Level** | [Incubation](https://github.com/swift-server/sswg/blob/main/process/incubation.md#process-diagram) |
| **License** | [MIT](https://mit-license.org/) |
| **Dependencies** | [swift-nio](https://github.com/apple/swift-nio), [swift-collections](https://github.com/apple/swift-collections) |

## Introduction

GraphQL is an increasingly common web API technology that improves client overfetching and reduces network communication between clients and servers.

This library is a Swift port of the [JavaScript reference GraphQL implementation](https://github.com/graphql/graphql-js), written on top of Swift NIO. It provides full server-side instrumentation for defining GraphQL schemas, resolving queries, and encoding/decoding GraphQL messages.

## Motivation

With GraphQL's growing popularity, the server-side Swift community should have a robust backend implementation to reduce the overhead of adding GraphQL APIs to webserver projects. This overhead is significant; the type system and resolution patterns of GraphQL require a 

A single implementation would be ideal to maximize the benefits of increased attention and community engagement, including faster bug reports and fixes, improved performance, and quicker feature adoption.

## Proposed solution

The proposed solution is a Swift port of the JavaScript reference implementation, built on NIO event loops.

Here is an example of a basic `"Hello world"` GraphQL schema:

```swift
let schema = try GraphQLSchema(
    query: GraphQLObjectType(                   // Defines the special "query" type
        name: "Query",
        fields: [
            "hello": GraphQLField(              // Users may query 'hello'
                type: GraphQLString,            // The result is a string type
                resolve: { _, _, _, _ in
                    "world"                     // The result of querying 'hello' is "world"
                }
            )
        ]
    )
)
```

This schema can be queried using the `graphql` function:

```swift
let result = try await graphql(
    schema: schema,
    request: "{ hello }",
    eventLoopGroup: eventLoopGroup
)
```

The result of this query is a `GraphQLResult` that encodes to the following JSON:

```json
{ "hello": "world" }
```

For more in-depth examples, see the [repository readme](https://github.com/GraphQLSwift/GraphQL/blob/master/README.md)

The package also includes support for GraphQL subscriptions and Swift `async`/`await` concurrency features.

## Detailed design

The proposed library aims to be a [spec-compliant](https://spec.graphql.org/October2021/#sec-Overview) implementation of GraphQL, so it's design follows the requirements in that document. Where flexibility is allowed by the spec, this library aims to match the JavaScript reference implementation. However, this is not always possible, generally due to Swift/JavaScript language inconsistencies. A few examples of Swift-specific design choices are listed below:

### Concurrency

Until recently, Swift did not include language features to implement event-loop based concurrency like JavaScript. As a result, this library uses the popular [`swift-nio`](https://github.com/apple/swift-nio) package to perform operations concurrently. Swift `async`/`await` support has been added in a backwards-compatible way for clients that are running Swift 5.5 and greater.

### Ordered Encoding/Decoding

The GraphQL spec [requires that results match the ordering of the request](https://spec.graphql.org/October2021/#sec-Serialized-Map-Ordering). However, the Swift Dictionary and JSONEncoder do not guarantee ordering. To resolve this, the package makes use of `OrderedDictionary` almost everywhere the Javascript implementation uses a normal `object`, and provides a custom [`GraphQLJSONEncoder`](https://github.com/GraphQLSwift/GraphQL/blob/master/Sources/GraphQL/Map/GraphQLJSONEncoder.swift) to ensure that encoded JSON results match the underlying ordered dictionary ordering, while still appearing as a simple JSON object.

## Maturity Justification

We are proposing this package at the "Incubation" level of maturity. I've listed the requirements for this maturity level and responses below:

Minimum Requirements:

* General
  * Has relevance to Swift on Server specifically: **Argued in Motivation section**
  * Publicly accessible source managed by an SCM such as github.com or similar: **Satisfied: Repository is stored on github.com**
    * Prefer to use `main` as the default branch name, in line with [Swift's guidelines](https://forums.swift.org/t/moving-default-branch-to-main/38515): **Satisfied**
  * Adopt the [Swift Code of Conduct](https://swift.org/community/#code-of-conduct): **Satisfied**
* Ecosystem
  * Uses SwiftPM: **Satisfied**
  * Integrated with critical SSWG ecosystem building blocks, e.g., Logging and Metrics APIs, SwiftNIO for IO: **Satisfied**
* Longevity
  * Must be from a team that has more than one public repository (or similar indication of experience): **Satisfied: See the [GraphQLSwift Organization](https://github.com/GraphQLSwift)**
  * SSWG should have access / authorization to graduated repositories in case of emergency: **This can be arranged**
  * Adopt the [SSWG Security Best Practices](../security/README.md)): **Satisfied**
* Testing, CI and Release
  * Have unit tests for Linux: **Satisfied**
  * CI setup, including testing PRs and the main branch: **Satisfied**
  * Follow semantic versioning, with at least one published pre-release (e.g. 0.1.0, 1.0.0-beta.1) or release (e.g. 1.0.0): **Satisfied**
* Licensing
  * Apache 2, MIT, or BSD (Apache 2 recommended): **Satisfied: MIT**
* Conventions and Style
  * Adopt [Swift API Design Guidelines](https://swift.org/documentation/api-design-guidelines/)
  * Follow [SSWG Technical Best Practices](#technical-best-practices) when applicable.
  * Prefer to adopt code formatting tools and integrate them into the CI: **No code formatting tools are integrated currently**

Incubating Requirements:

* Document that it is being used successfully in production by at least two independent end users which, in the SSWG judgement, are of adequate quality and scope.
  * **[Jay Herron](https://github.com/NeedleInAJayStack) is using it in a commercial product at [PassiveLogic](https://passivelogic.com/). This package is also used in multiple actively maintained open source projects, including [Graphiti](https://github.com/GraphQLSwift/Graphiti), and [Pioneer](https://github.com/d-exclaimation/pioneer).**
* Must have 2+ maintainers and/or committers. In this context, a committer is an individual who was given write access to the codebase and actively writes code to add new features and fix any bugs and security issues. A maintainer is an individual who has write access to the codebase and actively reviews and manages contributions from the rest of the project's community. In all cases, code should be reviewed by at least one other individual before being released.
  * **Pull requests over the past year have been authored by 4 different contributors. The GraphQLSwift organization has 7 maintainers. The repo `master` branch protection rules require that at least 1 maintainer approve a pull request before it may be merged.**
* Packages must have more than one person with admin access. This is to avoid losing access to any packages. For packages hosted on GitHub and GitLab, the packages must live in an organization with at least two administrators. If you don't want to create an organization for the package, you can host them in the [Swift Server Community](https://github.com/swift-server-community) organization.
  * **We have 2 active administrators in the GraphQLSwift organization - [Paulo Faria](https://github.com/paulofaria) and [Jay Herron](https://github.com/NeedleInAJayStack). We both have admin access to the GraphQL repo.**
* Demonstrate an ongoing flow of commits and merged contributions, or issues addressed in timely manner, or similar indication of activity.
  * **11 pull requests have been merged in the past year.**
* Receive a supermajority vote from the SSWG to move to Incubation stage.

## Alternatives considered

As far as I can tell, this is the only open-source GraphQL spec implementation in Swift. All other implementations I could find are either forks or depend upon the proposed package. The repository averages about 70 clones per week, and underpins other popular server-side GraphQL packages, such as [Graphiti](https://github.com/GraphQLSwift/Graphiti) and [GraphZahl](https://github.com/nerdsupremacist/GraphZahl), both of which are listed in the [graphql.org's Swift server language support section](https://graphql.org/code/#swift-objective-c).
