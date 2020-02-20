# Swift Crypto

* Proposal: [SSWG-0009](0009-swift-crypto.md)
* Authors: [Cory Benfield](https://github.com/Lukasa), [Frederic Jacobs](https://github.com/FredericJacobs)
* Sponsor(s): Apple
* Review Manager: TBD
* Status: [**Implemented**](https://github.com/apple/swift-crypto)

## Package Description

Swift Crypto is an open-source implementation of a substantial portion of the API of [Apple CryptoKit](https://developer.apple.com/documentation/cryptokit) suitable for use on Linux platforms. It enables cross-platform or server applications with the advantages of CryptoKit.

|  |  |
|--|--|
| **Package Name** | `swift-crypto` |
| **Module Name** | `Crypto` |
| **Proposed Maturity Level** | [Sandbox](https://github.com/swift-server/sswg/blob/master/process/incubation.md#process-diagram) |
| **License** | [Apache 2.0](https://choosealicense.com/licenses/apache-2.0/) |
| **Dependencies** | None (Vendored copy of [BoringSSL](https://boringssl.googlesource.com/boringssl/)) |

## Introduction

An extremely wide range of applications rely on accessing some form of cryptographic primitive in order to function safely and correctly. Swift Crypto brings the API of Apple CryptoKit, a framework focused on providing solutions to common cryptographic problems with safe, composable, high-level interfaces, into the wider Swift ecosystem. As server-side Swift is a vital part of the wider Swift ecosystem, Swift Crypto aims to be an important building block for a wide range of server-side applications and frameworks, bringing the same powerful APIs that are available on Apple platforms to server-side use-cases.

## Motivation

The number of server-side use-cases for cryptography is enormous, and affects almost all server-side applications in one form or another. Often these use-cases can be quite simple, such as needing to verify message hashes. However, in many cases the work required is substantially more complex, such as performing key agreement operations to establish shared encryption secrets, or performing message signing operations.

Today there are a wide range of cryptographic libraries available to server developers to allow them to perform cryptographic operations. Most of these libraries, however, are quite low-level. This leads to users needing to interact with raw pointers, or to manually manage memory, or to have to make complex choices about exactly what cryptographic technologies to use. In the best case this can lead to decision paralysis, but in the worst case users can inadvertently introduce security critical vulnerabilities into their applications.

We believe that there is an important space in the ecosystem for a library that focuses on providing high-quality interfaces to a limited number of cryptographic primitives. The goal is to provide safe, high-performance APIs to a collection of cryptographic "good choices". In the vast majority of cases, developers will not need anything other than what Swift Crypto provides, and will be able to write safe Swift code to interact with these high-level APIs without needing to be experts in their use. In some cases developers will be developing brand new systems that use cryptography: in these cases, they can safely pick anything from Swift Crypto (that doesn't include the word `Insecure` in its API!) and be confident that their choice is a good one.

## Detailed design

Swift Crypto has an extremely large API surface, so an exhaustive exploration of that API would make this document impractically long. We recommend consulting [the Apple CryptoKit documentation](https://developer.apple.com/documentation/cryptokit) for an outline of this API. However, we can provide a high level overview of the functions provided by Swift Crypto, and some examples of the API.

### Cryptographic Hash Functions

Cryptographic hash functions are commonly used to provide a "message digest": essentially, it maps an arbitrarily sized chunk of data into a fixed size chunk of data. A cryptographic hash function must be a "one-way function", which means that given a specific digest it must be very hard to produce a message that would generate that digest. Note that cryptographic hash functions are not suitable for password hashing: at this time, Swift Crypto does not contain any hash functions that _are_ suitable for this use-case.

Swift Crypto provides 5 cryptographic hash functions:

- `SHA256`
- `SHA384`
- `SHA512`
- `Insecure.SHA1`
- `Insecure.MD5`

Each of these hash functions supports two usage modes: one-shot and incremental/streaming. The one shot mode is extremely straightforward to use:

```swift
static func hash<D>(data: D) -> Self.Digest where D : DataProtocol
```

They can also be used incrementally with the following functions:

```swift
init()

mutating func update<D>(data: D) where D : DataProtocol

mutating func update(bufferPointer: UnsafeRawBufferPointer)

func finalize() -> Self.Digest
```

Swift Crypto's cryptographic hash functions and digest types are all value types. This allows interesting use-cases built on top of value semantics, such as rapidly calculating the digest for a number of different values that have the same prefix and different suffixes.

Digest's are simple types: they are sequences of UInt8, and they are able to be compared with both themselves and any arbitrary sequence of bytes.

### Message Authentication Codes

A message authentication code (MAC) is a short chunk of information that is used to confirm that a message was sent by a specific sender, and has not been modified in transit. These are heavily used in any circumstance where information is transferred between multiple peers where it is possible to ensure that these peers have a shared secret, or in constructions where data is handed to an untrusted party that will eventually send it back to you (e.g. macaroons).

Swift Crypto supports one such MAC, HMAC. HMAC is parameterised over the hash functions above. Like the hash functions above, HMACs can be calculated using both a one-shot API and a streaming API. The streaming API looks essentially identical to the hash function API, but substitutes `HashFunction.Digest` for `HMAC<HashFunction>.MAC`. The one-shot API is:

```swift
static func authenticationCode<D>(for data: D, using key: SymmetricKey) -> HMAC<H>.MAC where D : DataProtocol
```

Once again, this is a generic API that allows a wide range of data types to be used, including SwiftNIO's `ByteBufferView` and Foundation's `Data`.

This API also provides an extra helper for users that are already holding a `HMAC<HashFunction>.MAC`:

```swift
static func isValidAuthenticationCode<D>(_ authenticationCode: HMAC<H>.MAC, authenticating authenticatedData: D, using key: SymmetricKey) -> Bool where D : DataProtocol
```

This allows users to express the high level question ("is this MAC valid for that data") without needing to directly calculate the MAC over the data they want to verify.

### Ciphers

Swift Crypto provides two ciphers for use. In both cases these are Authenticated Encryption with Additional Data (AEAD) ciphers. These are modern constructions that are extremely resilient against attacks that can be launched by modifying the ciphertext of encrypted data. As AEAD ciphers cannot operate in a streaming mode, both ciphers offer one-shot APIs. There are four functions provided:

```swift
static func seal<Plaintext>(_ message: Plaintext, using key: SymmetricKey, nonce: Cipher.Nonce? = nil) throws -> Cipher.SealedBox where Plaintext : DataProtocol
static func seal<Plaintext, AuthenticatedData>(_ message: Plaintext, using key: SymmetricKey, nonce: Cipher.Nonce? = nil, authenticating authenticatedData: AuthenticatedData) throws -> Cipher.SealedBox where Plaintext : DataProtocol, AuthenticatedData : DataProtocol

static func open(_ sealedBox: Cipher.SealedBox, using key: SymmetricKey) throws -> Data
static func open<AuthenticatedData>(_ sealedBox: Cipher.SealedBox, using key: SymmetricKey, authenticating authenticatedData: AuthenticatedData) throws -> Data where AuthenticatedData : DataProtocol
```

These APIs provide simple options for the common case where there is no authenticated additional data, while allowing users to provide that data when needed. They also choose sensible random values for nonces when users don't need to provide a specific one.

Both ciphers also offer a `SealedBox` construction. This construction is an association of plaintext, nonce, and tag, the complete union of things that need to be preserved for decryption of the data. `SealedBox` supports a default serialization mode as well, so in many cases users can simply use `SealedBox.combined` to emit the data and `SealedBox.init(combined:)` to parse it.

### Public Key Cryptography

The gold standard of modern public key cryptography is elliptic curve cryptography. Swift Crypto supports the most widely-used elliptic curve constructions. In particular, it supports both key agreement and signing using the three major NIST elliptic curves (P-256, P-384, and P-521) and Curve 25519. These curves cover the vast majority of elliptic curve uses today, and offer a wide range of compatibility with other implementations.

The curves provide extensive APIs for signing and verifying signatures, as well as performing key exchanges. These APIs follow the patterns above: they provide generic interfaces over `DataProtocol`, they return typed objects (e.g. `ECDSASignature`) with value semantics, and they have very little configuration. There are also helpers for generating keys from random bytes.

## Maturity Justification

While CryptoKit has wide usage on Apple platforms, Swift Crypto has up until it was released on February 3rd 2020 only been used by Apple internally. For this reason it does not meet the criterion for moving directly to **Incubating**. Due to the importance of this use-case and the value of a shared API between client and server platforms, we expect that Swift Crypto will rapidly acquire sufficient use to move to **Incubating** and **Graduated**, but at this time **Sandbox** is the most appropriate case.


