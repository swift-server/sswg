# Codable implementation of OpenAPI Spec v3.x

* Proposal: SSWG-0011
* Authors: [Matt Polzin](https://github.com/mattpolzin)
* Sponsors: TBD
* Review Manager: Tanner Nelson
* Status:  **Implemented**
* Pitch: [OpenAPIKit](https://forums.swift.org/t/openapikit/32572)
* Implementation: [mattpolzin/OpenAPIKit](https://github.com/mattpolzin/OpenAPIKit)

## Package Description

A library containing Swift types that encode to- and decode from [OpenAPI](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.3.md) Documents and their components.

|||
| --- | --- |
|**Package name**|`OpenAPIKit`|
|**Proposed Maturity Level**|[Sandbox ](https://github.com/swift-server/sswg/blob/master/process/incubation.md#process-diagram)|
|**License**|[MIT](https://www.mit.edu/~amini/LICENSE.md)|
|**Library Dependencies**|N/A|
|**Test-only Dependencies**|[Yams](https://github.com/jpsim/Yams) (MIT), [FineJSON](https://github.com/omochi/FineJSON) (MIT)|

## Introduction

OpenAPIKit provides types that parse and serialize OpenAPI documentation using Swift's `Codable` protocols. It aims to stay structurally close enough to the Spec to be easy to understand given familiarity with OpenAPI, but it also takes advantage of Swift's type system to provide a "Swifty" interface to an OpenAPI AST and it additionally enforces the OpenAPI spec by making illegal things largely impossible to represent. The [Project Status](https://github.com/mattpolzin/OpenAPIKit#project-status) can be viewed as a glossary of OpenAPI terminology that references the corresponding OpenAPIKit types.

## Motivation

OpenAPI is a broadly used specification for writing API documentation. OpenAPI documents can be used to [generate interactive documentation ](https://openapi.tools/#documentation), [automate testing](https://openapi.tools/#testing), [generate code ](https://github.com/OpenAPITools/openapi-generator), or just provide a solid source of truth and a contract between a client and a server.

As linked above, a lot of great tooling already exists around the OpenAPI specification. The aforementioned code generator even supports Swift with [improvements ](https://forums.swift.org/t/open-api-tools-swift-personal-experience/31962/18) being actively discussed.

OpenAPIKit fits into the existing ecosystem as a relatively low level library, with the intention of supporting other libraries and tools on top of it. It currently captures nearly all of the specification in Swift  `Codable`  types. Thanks to Swift's type system, OpenAPIKit validates OpenAPI documentation  *simply by decoding it*  and it guarantees that OpenAPI documentation it  *encodes*  meets the spec as well.

In short, OpenAPIKit is a foundation for any Swift code that aims to read or write OpenAPI documentation. My hope is that this spec implementation saves time for others interested in writing tooling or frameworks with a higher level of abstraction. 

There is a more compelling answer to "why?" though: Any Swift developer can harness the substantial power of an OpenAPI document without leaving the comfort and familiarity of the Swift language. This is the benefit I personally take away from it on a regular basis, having authored a handful of tools and libraries using OpenAPIKit already. With the parsing, validating, and serializing out of the way, someone authoring a library or implementing a dev tool or integrating OpenAPI into their SAAS app can jump straight to working with the AST.

## Proposed Solution

### Codable

OpenAPIKit takes advantage of `Codable` at every turn to create a single implementation that easily parses from- and serializes to JSON or YAML and takes advantage of independent advances by any `Encoder`s or `Decoder`s available.

### AST with a Swifty Interface

"Swifty" is an admittedly vague descriptor. OpenAPIKit uses `struct`s and `enum`s to build out a nested structure of types that both encourages discoverability and ensures validity.  

One of the primary goals was to produce a structure that was as easy to use declaratively as it was to traverse post-creation. OpenAPIKit almost reads like YAML (_almost_).

#### Writing OpenAPI documentation in Swift
```swift
// OpenAPI Info Object
// https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.3.md#info-object
let info = OpenAPI.Document.Info(title: "Demo API", version: "1.0")

// OpenAPI Server Object
// https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.3.md#server-object
let server = OpenAPI.Server(url: URL(string: "https://demo.server.com")!)

// OpenAPI Components Object
// https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.3.md#components-object
let components = OpenAPI.Components(
    schemas: [
        "hello_string": .string(allowedValues: "hello")
    ]
)

// OpenAPI Response Object
// https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.3.md#response-object
let successfulHelloResponse = OpenAPI.Response(
    description: "Hello",
    content: [
        .txt: .init(schemaReference: .component(named: "hello_string"))
    ]
)

// OpenAPI Document Object (and several other nested types)
// https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.3.md#openapi-object
let _ = OpenAPI.Document(
    info: info,
    servers: [server],
    paths: [
        "/hello": .init(
            get: OpenAPI.PathItem.Operation(
                tags: ["Greetings"],
                summary: "Get a greeting",
                description: "An endpoint that says hello to you.",
                responses: [
                    200: .init(successfulHelloResponse)
                ]
            )
        )
    ],
    components: components
)
```

More examples can be found in the [ease of use tests](https://github.com/mattpolzin/OpenAPIKit/blob/master/Tests/OpenAPIKitTests/DeclarativeEaseOfUseTests.swift).

#### Traversing OpenAPI documentation in Swift

```swift
let data = ...
let doc = try decoder.decode(OpenAPI.Document.self, from: data)

// Loop over defined routes and print the path and operations defined for each path.
for (route, pathDefinition) in doc.paths {
    print(route.rawValue)
    // map GET, PUT, POST, PATCH, etc. to the operations defined for each
    let operations = OpenAPI.HttpVerb.allCases.compactMap { verb in
        pathDefinition.for(verb).map { (verb, $0) }
    }

    for (verb, operation) in operations {
        print("\(verb) -> \(operation.summary ?? "unknown description")")
    }
}

// Results similar to:
// ------------------
// test/api/endpoint/{param}
// get -> Get Test
// post -> Post Test
```

#### JSON References _[Section updated 03-15-2020 for v0.24.0]_
OpenAPI supports the use of JSON References in many locations. Anywhere this is allowed, OpenAPIKit exposes an `Either` property with one option being a reference and the other option being to directly specify the component in question. In many cases, `Either` has been extended with convenience static constructors allowing you to create either the reference or the component in an ergonomic and concise way.

As an example, `OpenAPI.Response.Map` (named the [Responses Object](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.3.md#responses-object) in the Spec) maps response status codes to either responses or references to responses. The following code defines a status `200` response inline and then refers to a status `404` response that lives at `#/components/responses/notFound`.
```swift
let responses: OpenAPI.Response.Map = [
  200: .response(
    description: "Success!",
    content: [
      .txt: .init(schema: .string)
    ]
  ),
  404: .response(reference: try components.reference(named: "notFound", ofType: OpenAPI.Response.self))
]
```

Improvements to support for references are on the roadmap ([#17](https://github.com/mattpolzin/OpenAPIKit/issues/17)). Currently, references to components living within the Components Object can be created or used to retrieve components.
```swift
let components: OpenAPI.Components = .init(
  schemas: [
    "example": .string
  ]
)
// creating references
let schemaReference: JSONReference<JSONSchema>
schemaReference = try components.reference(named: "example", ofType: JSONSchema.self)

// using references to grab components
let schema = components[schemaReference]
```

### Validation through type-safety

The vast majority of the OpenAPI specification can be represented in the Swift type system. This means that we can create `structs` and `enums` that are incapable of representing ill-formed OpenAPI documentation. This means the structure of OpenAPIKit diverges slightly from that of the OpenAPI specification as documented, but the diversions are generally small and documented (currently part of the "glossary" found in the [Project Status](https://github.com/mattpolzin/OpenAPIKit#project-status)).

### Human-readable Errors

`Swift.DecodingError` packs a lot of information, but it is not always easy to print that information in a very human-readable way. Additionally, in the context of a specification like OpenAPI we can sometimes offer error text that does a better job of pointing to a solution instead of just calling out a generic problem.

Although there's always room for improvement, I have done some work to set OpenAPIKit up for legible error messages.

When you catch an error coming out of OpenAPIKit, you can create an `OpenAPI.Error` from it. The reason OpenAPIKit does not just produce an `OpenAPI.Error` in the first place is because when you ask a `Decoder` to decode something it is going to wrap the result in a `DecodingError` anyway requiring _someone_ to do the work of unwrapping it and that is what `OpenAPI.Error` is all about.

Example use:
```swift
let openAPIDoc: OpenAPI.Document
do {
  try openAPIDoc = JSONDecoder().decode(OpenAPI.Document.self, from: ...)
} catch {
  let prettyError = OpenAPI.Error(from: error)
  print(prettyError.localizedDescription)
  print(prettyError.codingPathString)
}
```

One important note is that the error is not actually localized at the moment. I see this as an area for improvement because these error messages are otherwise a great fit for passing on to the end user of whatever tool is failing to parse the OpenAPI document.

You can see some examples of the error text produced [here](https://forums.swift.org/t/openapikit/32572/11?u=mattpolzin).

### Dependencies

As of OpenAPIKit v0.26.0, there are no external dependencies outside of those used strictly in test targets. Nevertheless, the following types are mentioned to briefly justify their existence seeing as how they are generally applicable types that have not been generally accepted into any core or standard Swift libraries.

#### Library

##### `OpenAPIKit.Either`

The OpenAPI Spec often defines things as one of two options and because encoding, decoding, and error handling logic are always going to be the same for these situations, an `Either` type made a lot of sense.

##### `OpenAPIKit.OrderedDictionary`

The OrderedDictionary library just offers up one type and I bet you can guess what that is. Ordering of hashes is important to OpenAPIKit for two reasons:
1. Users of OpenAPI may attribute meaning to the ordering of things like Path Items.
2. When fed into a UI like Redoc or SwaggerUI, the output of OpenAPIKit _should_ produce a stable view of documentation. It is disconcerting at best and confusing/frustrating at worst if the information you read yesterday is in a different part of the documentation today for no reason other than non-determinism.

##### `OpenAPIKit.AnyCodable`

OpenAPIKit uses AnyCodable anywhere that OpenAPI offers no structural restrictions on the JSON/YAML. This occurs most often where examples are allowed. Examples can be anything from a simple `String` value to a large nested structure containing `Dictionary`s and `Array`s.

#### Test-only

[Yams](https://github.com/jpsim/Yams) and [FineJSON](https://github.com/omochi/FineJSON) are not used by any targets that are not test targets. FineJSON and Yams both support testing of ordered encoding/decoding. Yams is additionally used to run OpenAPIKit against third party OpenAPI Documents in the [Compatibility Suite](https://github.com/mattpolzin/OpenAPIKit/tree/master/Tests/OpenAPIKitCompatibilitySuite) test target.

### Example and Prototype Uses
Largely out of a combination of curiosity and utility at my "real job," I've developed a handful of libraries or showcases of OpenAPIKit integrations into other libraries and tools. I hope that this serves to show the breadth of applications of such a library even though this is by no means a comprehensive list of uses.

- Vapor API documentation generation ([library](https://github.com/mattpolzin/VaporOpenAPI), [example app](https://github.com/mattpolzin/VaporOpenAPIExample))
- JSON:API schema generation ([library](https://github.com/mattpolzin/JSONAPI-OpenAPI#simple-example))
- Writing OpenAPI documentation ([example](https://github.com/mattpolzin/OpenAPIKit/blob/master/Tests/OpenAPIKitTests/DeclarativeEaseOfUseTests.swift#L141))
- Scripting ([tool](https://gist.github.com/mattpolzin/57dbfbc33e9e97b50f63b3e280866bca))
- Generating "API Changes" deliverables ([library/tool](https://github.com/mattpolzin/OpenAPIDiff))

### Next Steps

I am largely done adding to the _surface area_ of the implementation for now. There are holes in what is supported, but I would like to motivate filling them with requests from the community because the remaining holes are increasingly remote corners of the specification. 

I do plan to work on the following in the foreseeable future: 
1. Adding support for [Specification Extensions](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.2.md#specificationExtensions) to more types ([#24](https://github.com/mattpolzin/OpenAPIKit/issues/24)).
2. Improving support for `$ref`s ([#3](https://github.com/mattpolzin/OpenAPIKit/issues/3), [#17](https://github.com/mattpolzin/OpenAPIKit/issues/17)).
3. Improving the flexibility of decoding ([#23](https://github.com/mattpolzin/OpenAPIKit/issues/23)).
4. Improving the ergonomics of using OpenAPIKit types, including adding accessors that retrieve "canonical" information on an API (suggestions appreciated on this one, see [original pitch](https://forums.swift.org/t/openapikit/32572?u=mattpolzin) for more on this topic).

## Maturity Justification

This package meets the following criteria according to the [SSWG Incubation Process ](https://github.com/swift-server/sswg/blob/master/process/incubation.md):

* Follows semantic versioning
* Uses SwiftPM
* Code Style is up to date
* Errors implemented
* Unit testing for Mac OS and Linux in addition to a compatibility suite of tests.
* [Swift Code of Conduct](https://github.com/mattpolzin/OpenAPIKit/blob/master/CODE_OF_CONDUCT.md)

It notably does not meet the requirement of having a team of 2+ developers, although I believe that rule has been a bit flexible in the past. I do have a track record for maintaining open source libraries once they reach a level of maturity where others can begin taking advantage of them (e.g. [JSONAPI](https://github.com/mattpolzin/JSONAPI), [Poly](https://github.com/mattpolzin/Poly)).

## Alternatives Considered

### Existing Solutions

* [SwaggerParser](https://github.com/AttilaTheFun/SwaggerParser) (OpenAPI 2.0 support)
* [SwagGen](https://github.com/yonaskolb/SwagGen) (OpenAPI 3.0 parsing built-in but primarily a code gen. tool, no serializing of OpenAPI specs)
* [Kitura-OpenAPI](https://github.com/IBM-Swift/Kitura-OpenAPI) (OpenAPI 2.0 serializing support)
* [Swiftgger](https://github.com/mczachurski/Swiftgger) (OpenAPI 3.0 serializing, no parsing)