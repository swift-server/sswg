# MultipartKit

* Proposal: [SSWG-NNNN](NNNN-multipart-kit.md)
* Authors: [Siemen Sikkema](https://github.com/siemensikkema), [Tim Condon](https://github.com/0xTim)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: [repo name](https://github.com/vapor/multipart-kit)
* Forum Threads: [Pitch](https://forums.swift.org/t/multipartkit), [Discussion](https://forums.swift.org/), [Review](https://forums.swift.org/)

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
| **License** | [TBD](https://choosealicense.com/licenses/mit/) |
| **Dependencies** | [SwiftNIO >= 2.2.0](https://github.com/apple/swift-nio) |

## Introduction

MultipartKit offers both low level parsing and serializing of Multipart data as well as high level Codable support for encoding and decoding Multipart form data.

## Motivation

Multipart is a commonly used standard for transmitting data, often originating from HTML forms. It is particularly well-suited for binary data (like file uploads) but also supports structured data or a combination of the two.

There exists currently no standard package for multipart in the Swift server ecosystem. MultipartKit was originally developed for Vapor but is independent of it. Lifting MultipartKit's status as an officially supported package would increase its visibility so people are less likely to reinvent the wheel and can contribute to the features, stability, and performance of MultipartKit instead.

## Proposed solution

Describe your solution to the problem. Provide examples and describe
how they work. Show how your solution is better than current
workarounds: is it cleaner, safer, or more efficient?

## Detailed design

Describe the design of the solution in detail. If it's a new API, show the full API and its documentation
comments detailing what it does. The detail in this section should be
sufficient for someone who is *not* one of the authors to be able to
reasonably re-implement the feature.

## Maturity Justification

Explain why this solution should be accepted at the proposed maturity level.

## Alternatives considered

Describe alternative approaches to addressing the same problem, and
why you chose this approach instead.
