# Solution name

* Proposal: [SSWG-NNNN](NNNN-swift-asn1.md)
* Authors: [Cory Benfield](https://github.com/Lukasa), [David Nadoba](https://github.com/dnadoba)
* Review Manager: TBD
* Status: **Implemented**
* Implementation: [swift-asn1](https://github.com/apple/swift-asn1)
* Forum Threads: 

*During the review process, add the following fields as needed:*

* Decision Notes:
* Previous Revision(s):
* Previous Proposal(s):

## Package Description
An implementation of ASN.1 types and DER serialization.

|  |  |
|--|--|
| **Package Name** | `swift-asn1` |
| **Module Name** | `SwiftASN1` |
| **Proposed Maturity Level** | [Sandbox](https://www.swift.org/sswg/incubation-process.html#process-diagram) |
| **License** | [Apache 2.0](https://choosealicense.com/licenses/apache-2.0/) |
| **Dependencies** | None |

## Introduction

ASN.1, and the DER encoding scheme, is a commonly used object serialization format. ASN.1 can be used abstractly to describe essentially any kind of object. ASN.1 objects are made up of either primitive or composite (called "constructed") types.   The ASN.1 object description does not define a specific encoding for these objects.  Instead there are a wide range of possible ways to serialize or deserialize an ASN.1 object. This module provides an implementation of a number of ASN.1 types, as well as the DER serialization format for ASN.1.

## Motivation

The most common use-cases for ASN.1 in general computing are in the cryptographic space, but there are a number of use-cases in a wide range of fields.  Having a clean and safe way to serialise/deserialise for these use cases is highly desirable.

## Proposed solution

The solution separates the concerns of data representation from that of encoding enabling multiple encoding methods to be supported.  This solution does not parse ASN.1 documents - the user is expected to encode this information in the swift code directly by hand.

There are examples of encoding and decoding both [DER](https://swiftpackageindex.com/apple/swift-asn1/1.1.0/documentation/swiftasn1/decodingasn1) and [PEM](https://swiftpackageindex.com/apple/swift-asn1/1.1.0/documentation/swiftasn1/pem) in the documentation.

## Detailed design

There is a structure to represent each ASN.1 basic type - for example [ASN1Null](https://swiftpackageindex.com/apple/swift-asn1/1.1.0/documentation/swiftasn1/asn1null). 

ASN.1 documents are represented as an [ASN1node](https://swiftpackageindex.com/apple/swift-asn1/1.1.0/documentation/swiftasn1/asn1node).  Which may contain either a collection of other nodes to form a tree structure ("constructed" type), or primative data.  [ASN1Identifier](https://swiftpackageindex.com/apple/swift-asn1/1.1.0/documentation/swiftasn1/asn1identifier) represents an OID in the node heirarchy - which assigns meaning to the nodes within this tree struture.

A set of DER functions allow parsing serialsed data.  The parse function supplied takes a sequence of bytes and returns the decoded ASN1Node.  Functions are given to allow parsing of structured types such and sets and sequences - these are done by calling a builder which each node in the collection.

The [DER.Serialiser](https://swiftpackageindex.com/apple/swift-asn1/1.1.0/documentation/swiftasn1/der/serializer) structure contains an array a serialised bytes.  This array is appended as nodes are serialised.  This is achieved through member functions for serialising the various node types.

Parsing ASN.1 description documents is not supported.  The user implements their own data model.  This is done by defining structures for the data and implementing [DERParsable](https://swiftpackageindex.com/apple/swift-asn1/1.1.0/documentation/swiftasn1/derparseable) and [DERSerializable](https://swiftpackageindex.com/apple/swift-asn1/1.1.0/documentation/swiftasn1/derserializable).  When deserializing the framework checks the object identifer is as expected.

## Maturity Justification

The project has been in use for a little over a year.  It is used from swift-certificates and has proven robust.

## Alternatives considered
