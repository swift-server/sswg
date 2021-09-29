# MultipartKit

* Proposal: [SSWG-NNNN](NNNN-multipart-kit.md)
* Authors: [Siemen Sikkema](https://github.com/siemensikkema), [Tim Condon](https://github.com/0xTim)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: [MultipartKit](https://github.com/vapor/multipart-kit)
* Forum Threads: [Pitch](https://forums.swift.org/t/multipartkit)

<!--
*During the review process, add the following fields as needed:*

* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Previous Revision(s): [1](https://github.com/swift-server/sswg/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal(s): [SSWG-XXXX](XXXX-filename.md)
-->

## Package Description

Multipart parser and serializer with `Codable` support for Multipart Form Data.

|  |  |
|--|--|
| **Package Name** | `multipart-kit` |
| **Module Name** | `MultipartKit` |
| **Proposed Maturity Level** | [Sandbox](https://github.com/swift-server/sswg/blob/main/process/incubation.md#process-diagram) |
| **License** | [MIT](https://choosealicense.com/licenses/mit/) |
| **Dependencies** | [SwiftNIO >= 2.2.0](https://github.com/apple/swift-nio) |

## Introduction

MultipartKit offers both low level parsing and serializing of Multipart data as well as high level Codable support for encoding and decoding Multipart form data.

## Motivation

Multipart is a commonly used standard for transmitting data, often originating from HTML forms. It is particularly well-suited for binary data (like file uploads) but also supports structured data or a combination of the two.

There exists currently no standard package for multipart in the Swift server ecosystem. MultipartKit was originally developed for Vapor but is independent of it. Lifting MultipartKit's status as an officially supported package would increase its visibility so people are less likely to reinvent the wheel and can contribute to the features, stability, and performance of MultipartKit instead.

## Proposed solution

We propose that the existing MultipartKit implementation be promoted to an officially supported package as is.

## Detailed design

### Form Data

The high level Form Data API revolves around `FormDataEncoder` and `FormDataDecoder`. 

Encoding some encodable value `foo` as multipart using a boundary value of `abc123` can be achieved as follows:

```swift
let encoded = try FormDataEncoder().encode(foo, boundary: "abc123")
```

Similarly to decode this value into the type `Foo` we can do:

```swift
let foo = try FormDataDecoder().decode(Foo.self, from: encoded, boundary: "abc123")
```

### Nesting and Collections

Nested data and collections can be represented using an angle bracket notation convention for the part names as follows:

- `root[branch]`: a value named `branch` nested inside `root`
- `root[branch][leaf]`: like the above but with `leaf` nested inside `branch`
- `array[0]`: first value of `array`
- `array[1][leaf]`: a value named `leaf` nested inside the second value of `array`

### Low level parsing of Multipart data

MultipartKit comes with a callback-based `MultipartParser` that forms the basis for the Form Data coders.

```swift
let parser = MultipartParser(boundary: "abc123")
```

Set up the callbacks in order to process the parsed data:

```swift
parser.onHeader = { (field: String, value: String) in
    // gets called after each multipart header that is parsed
}
parser.onBody = { (buffer: ByteBuffer) in
    // gets called when (a part of) a multipart body is parsed
}
parser.onPartComplete = {
    // gets called when a part is fully parsed
    // the body along with any collected headers for the current part can now be considered complete
}
```

Feed data into the parser:
```swift
let data: ByteBuffer = ...
parser.execute(data)
```

In case the data is arriving in chunks. The `execute` method can be called once for each chunk.

## Maturity Justification

Although it has been used in production successfully alongside Vapor it would be good to see it used and tested in the wider Swift on the Server ecosystem.

## Alternatives considered

None