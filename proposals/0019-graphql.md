# GraphQL & Graphiti

* Proposal: [SSWG-0019](0019-graphql.md)
* Authors: [Jay Herron](https://github.com/NeedleInAJayStack)
* Review Manager: [Konrad 'ktoso' Malawski](https://github.com/ktoso)
* Status: **Accepted**, https://forums.swift.org/t/sswg-0019-graphql-graphiti/59799/12
* Implementation: 
    * [GraphQL](https://github.com/GraphQLSwift/GraphQL)
    * [Graphiti](https://github.com/GraphQLSwift/Graphiti)
* Forum Threads: 
    *  [Pitch](https://forums.swift.org/t/pitch-graphql/59554)
    *  [Review](https://forums.swift.org/t/sswg-0019-graphql-graphiti/59799)

## Package Description
These libraries are a Swift port of the [JavaScript reference GraphQL implementation](https://github.com/graphql/graphql-js), written on top of Swift NIO. They provides full server-side instrumentation for defining GraphQL schemas, resolving queries, and encoding/decoding GraphQL messages.

|  |  |
|--|--|
| **Package Name** | `GraphQL`/`Graphiti` |
| **Module Name** | `GraphQL`/`Graphiti` |
| **Proposed Maturity Level** | [Incubation](https://github.com/swift-server/sswg/blob/main/process/incubation.md#process-diagram) |
| **License** | [MIT](https://mit-license.org/) |
| **Dependencies** | [swift-nio](https://github.com/apple/swift-nio), [swift-collections](https://github.com/apple/swift-collections) |

## Introduction

GraphQL is an increasingly common web API technology that improves client overfetching and reduces network communication between clients and servers.

These libraries are a Swift port of the [JavaScript reference GraphQL implementation](https://github.com/graphql/graphql-js), written on top of Swift NIO. They provide full server-side instrumentation for defining GraphQL schemas, resolving queries, and encoding/decoding GraphQL messages.

## Motivation

With GraphQL's growing popularity, the server-side Swift community should have a robust backend implementation to reduce the overhead of adding GraphQL APIs to webserver projects. This overhead is significant; the GraphQL spec defines a rich type system, message formatting requirements, and multiple resolution patterns that require a substantial amount of logic. 

A single implementation would be ideal to maximize the benefits of increased attention and community engagement, including faster bug reports and fixes, improved performance, and quicker feature adoption.

## Proposed solution

The proposed solution is a Swift port of the JavaScript reference implementation, built on NIO event loops.

### Hello World

Here is an example of a basic `"Hello world"` GraphQL schema:

```swift
import Graphiti
import NIO

let eventLoopGroup = MultiThreadedEventLoopGroup(numberOfThreads: 1)

struct HelloResolver {
    func hello(context: NoContext, arguments: NoArguments) -> String {
        return "world"
    }
}

struct HelloAPI : API {
    typealias ContextType = NoContext
    let resolver = HelloResolver()
    let schema = try! Schema<HelloResolver, NoContext> {
        Query {
            Field("hello", at: HelloResolver.hello)
        }
    }
}
```

This schema can be queried using the `graphql` function:

```swift
let api = HelloApi()
let result = try await api.execute(
    request: "{ hello }",
    eventLoopGroup: eventLoopGroup
)
```

The result of this query is a `GraphQLResult` that encodes to the following JSON:

```json
{ "hello": "world" }
```

### Swift Type Usage

Graphiti includes support for using Swift types in the schema itself. To connect the Swift type with the GraphQL one, include a `Type` block in the API declaration, composed of `Field`s. For example, we can integrate a `Person` object into the API:

```swift
import Graphiti
import NIO

let eventLoopGroup = MultiThreadedEventLoopGroup(numberOfThreads: 1)

struct Person: Codable {
    let name: String
    let age: Int
    let height: Float
}

let characters = [
    Person(name: "Johnny Utah", age: 23, height: 1.85),
    Person(name: "Bodhi", age: 27, height: 1.8),
]

struct PersonResolver {
    func people(context: NoContext, arguments: NoArguments) -> [Person] {
        return characters
    }
}

struct PersonAPI : API {
    typealias ContextType = NoContext
    let resolver = PersonResolver()
    let schema = try! Schema<PersonResolver, NoContext> {
        Type(Person.self) {
            Field("name", at: \.name)
            Field("age", at: \.age)
            Field("height", at: \.height)
        }
        Query {
            Field("people", at: PersonResolver.people)
        }
    }
}

let result = try await PersonAPI().execute(
    request: """
    {
      people {
        name
        age
      }
    }
    """,
    context: NoContext(),
    on: eventLoopGroup
)
```

The `result` above could be decoded to a JSON of the form:

```json
{
  "people": [
    "name": "Johnny Utah",
    "name": "Bodhi"
  ]
}
```

### Arguments

Arguments can be defined within an API using the `Argument` initializer in a `Field` builder. Adjusting our previous example, we can add in an argument to filter people by their age.

```swift
struct PeopleArguments: Codable {
    let olderThan: Int
}

struct PersonResolver {
    func people(context: NoContext, arguments: PeopleArguments) -> [Person] {
        return characters.filter { $0.age > arguments.olderThan }
    }
}

struct PersonAPI : API {
    typealias ContextType = NoContext
    let resolver = PersonResolver()
    let schema = try! Schema<PersonResolver, NoContext> {
        Type(Person.self) {
            Field("name", at: \.name)
            Field("age", at: \.age)
            Field("height", at: \.height)
        }
        Query {
            Field("people", at: PersonResolver.people) {
                Argument("olderThan", at: \.olderThan)
            }
        }
    }
}
```

A request string for this might be:

```graphql
{
  people(olderThan: 25) {
    name
  }
}
```

which would generate the response:

```json
{
  "people": [
    {
      "name": "Bodhi"
    }
  ]
}
```

### Mutations

Mutations are defined using a `Mutation` block in the API, and are typically used to change an underlying dataset. We can expand our example to include a mutation that creates a new person:

```swift
struct NewPersonArguments: Codable {
    let name: String
    let age: Int
    let height: Float
}

struct PersonResolver {
    func newPerson(context: NoContext, arguments: NewPersonArguments) -> Person {
        return Person(
            name: arguments.name,
            age: arguments.age,
            height: arguments.height
        )
    }
}

struct PersonAPI : API {
    typealias ContextType = NoContext
    let resolver = PersonResolver()
    let schema = try! Schema<PersonResolver, NoContext> {
        Type(Person.self) {
            Field("name", at: \.name)
            Field("age", at: \.age)
            Field("height", at: \.height)
        }
        Mutation {
            Field("newPerson", at: PersonResolver.newPerson) {
                Argument("name", at: \.name)
                Argument("age", at: \.age)
                Argument("height", at: \.height)
            }
        }
    }
}
```

A request string for this might be:

```graphql
mutation {
  newPerson(name: "Tyler Endicott", age: 22, height: 1.63) {
    name
  }
}
```

which would generate the response:

```json
{
  "newPerson": {
    "name": "Tyler Endicott"
  }
}
```

### Input Objects

Sometimes we'd like to pass a complex argument. `Input`s allow us to do this and are declared by including an `Input` block in the API declaration, composed of `InputField`s. Our example can be changed to include a mutation that creates multiple new people, each passed as an input object:

```swift
struct NewPeopleArguments: Codable {
    let individuals: [Person]
}

struct PersonResolver {
    func newPeople(context: NoContext, arguments: NewPeopleArguments) -> [Person] {
        return arguments.individuals.map { person in
            Person(
                name: person.name,
                age: person.age,
                height: person.height
            )
        }
    }
}

struct PersonAPI : API {
    typealias ContextType = NoContext
    let resolver = PersonResolver()
    let schema = try! Schema<PersonResolver, NoContext> {
        Type(Person.self) {
            Field("name", at: \.name)
            Field("age", at: \.age)
            Field("height", at: \.height)
        }
        Input(Person.self) {
            InputField("name", at: \.name)
            InputField("age", at: \.age)
            InputField("height", at: \.height)
        }
        Mutation {
            Field("newPeople", at: PersonResolver.newPeople) {
                Argument("individuals", at: \.individuals)
            }
        }
    }
}
```

A request might look like:

```graphql
mutation {
  newPeople(individuals: [
    {name: "Tyler Endicott", age: 22, height: 1.63},
    {name: "Angelo Pappas", age: 45, height: 1.91},
  ]) {
    name
  }
}
```

which would generate the response:

```json
{
  "newPeople": [
    {
      "name": "Tyler Endicott",
      "name": "Angelo Pappas"
    }
  ]
}
```

### Subscriptions

Subscriptions are reactive queries that return a result whenever an event occurs. This functionality is built on Swift Concurrency using `AsyncThrowingStream`. To create a subscription, include a `Subscription` block in the API declaration composed of `SubscriptionFields`. We can change our example API to include a subscription alert:

```swift
import Foundation
import GraphQL

struct PersonResolver {
    func fiftyYearStormAlert(context: NoContext, arguments: NoArguments) -> ConcurrentEventStream<String> {
        let asyncStream = AsyncThrowingStream<String, Error> { continuation in
            let timer = Timer.scheduledTimer(
                withTimeInterval: 60 * 60 * 24 * 365 * 50,
                repeats: true
            ) { _ in
                continuation.yield("A 50-year storm is occurring!")
            }
        }
        return ConcurrentEventStream<String>.init(asyncStream)
    }
}

struct PersonAPI : API {
    typealias ContextType = NoContext
    let resolver = PersonResolver()
    let schema = try! Schema<PersonResolver, NoContext> {
        Type(Person.self) {
            Field("name", at: \.name)
            Field("age", at: \.age)
            Field("height", at: \.height)
        }
        Subscription {
            SubscriptionField(
              "fiftyYearStormAlert",
              as: String.self,
              atSub: PersonResolver.fiftyYearStormAlert
            )
        }
    }
}
```

When a subscription request is made:

```graphql
subscription {
  fiftyYearStormAlert
}
```

The following message will be generated every fifty years:

```json
{
  "fiftyYearStormAlert": "A 50-year storm is occurring!"
}
```

### Additional Examples

For additional examples, see the [repository readme](https://github.com/GraphQLSwift/GraphQL/blob/master/README.md)

The packages also include support for GraphQL subscriptions.

## Detailed design

The proposed libraries aims to be a [spec-compliant](https://spec.graphql.org/October2021/#sec-Overview) implementation of GraphQL, so it's design follows the requirements in that document. Where flexibility is allowed by the spec, the libraries aims to match the JavaScript reference implementation. However, this is not always possible, generally due to Swift/JavaScript language inconsistencies. A few examples of Swift-specific design choices are listed below:

### Concurrency

Until recently, Swift did not include language features to implement event-loop based concurrency like JavaScript. As a result, these libraries use the popular [`swift-nio`](https://github.com/apple/swift-nio) package to perform operations concurrently. Swift `async`/`await` support has been added in a backwards-compatible way for clients that are running Swift 5.5 and greater.

### Ordered Encoding/Decoding

The GraphQL spec [requires that results match the ordering of the request](https://spec.graphql.org/October2021/#sec-Serialized-Map-Ordering). However, the Swift Dictionary and JSONEncoder do not guarantee ordering. To resolve this, these packages makes use of `OrderedDictionary` almost everywhere the Javascript implementation uses a normal `object`, and provide a custom [`GraphQLJSONEncoder`](https://github.com/GraphQLSwift/GraphQL/blob/master/Sources/GraphQL/Map/GraphQLJSONEncoder.swift) to ensure that encoded JSON results match the underlying ordered dictionary ordering, while still appearing as a simple JSON object.

### Schema Definition

Schema definition in JavaScript takes advantage of the language's relaxed type system, whereas defining schemas in Swift is more difficult because of Swift's type safety. [Graphiti](https://github.com/GraphQLSwift/Graphiti) was created to ease this difficulty by exposing a result-builder-based DSL that closely resembles the schema definition style in JavaScript.

## Maturity Justification

We are proposing these packages at the "Incubation" level of maturity. I've listed the requirements for this maturity level and responses below:

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
  * Prefer to adopt code formatting tools and integrate them into the CI: **Satisfied**

Incubating Requirements:

* Document that it is being used successfully in production by at least two independent end users which, in the SSWG judgement, are of adequate quality and scope.
  * **[Jay Herron](https://github.com/NeedleInAJayStack) is using the packages in a commercial product at [PassiveLogic](https://passivelogic.com/). They are also used in multiple actively maintained open source projects, including [Pioneer](https://github.com/d-exclaimation/pioneer).**
* Must have 2+ maintainers and/or committers. In this context, a committer is an individual who was given write access to the codebase and actively writes code to add new features and fix any bugs and security issues. A maintainer is an individual who has write access to the codebase and actively reviews and manages contributions from the rest of the project's community. In all cases, code should be reviewed by at least one other individual before being released.
  * **Pull requests over the past year have been authored by 4 different contributors. The GraphQLSwift organization has 7 maintainers. The repos have `master` branch protection rules that require that at least 1 maintainer approve a pull request before it may be merged.**
* Packages must have more than one person with admin access. This is to avoid losing access to any packages. For packages hosted on GitHub and GitLab, the packages must live in an organization with at least two administrators. If you don't want to create an organization for the package, you can host them in the [Swift Server Community](https://github.com/swift-server-community) organization.
  * **We have 2 active administrators in the GraphQLSwift organization - [Paulo Faria](https://github.com/paulofaria) and [Jay Herron](https://github.com/NeedleInAJayStack). We both have admin access to the GraphQL and Graphiti repos.**
* Demonstrate an ongoing flow of commits and merged contributions, or issues addressed in timely manner, or similar indication of activity.
  * **16 pull requests have been merged in the past year between the two packages.**
* Receive a supermajority vote from the SSWG to move to Incubation stage.

## Alternatives considered

As far as I can tell, this is the only open-source GraphQL spec implementation in Swift. All other implementations I could find are either forks or depend upon the proposed package. The repository averages about 70 clones per week, and underpins other popular server-side GraphQL packages, such as [GraphZahl](https://github.com/nerdsupremacist/GraphZahl) and [Pioneer](https://github.com/d-exclaimation/pioneer). [Graphiti](https://github.com/GraphQLSwift/Graphiti) is listed in the [graphql.org's Swift server language support section](https://graphql.org/code/#swift-objective-c).
