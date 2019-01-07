---
layout: post
title:  Why doesn't JSONEncoder conform to the Encoder protocol? 🤔
date:   2019-01-07 12:00:00 -0400
categories:
---

A common point of confusion in the Swift encoding and decoding ecosystem is that there is an [`Encoder` protocol](https://developer.apple.com/documentation/swift/encoder), but standard library types like [`JSONEncoder`](https://developer.apple.com/documentation/foundation/jsonencoder) and [`PropertyListEncoder`](https://developer.apple.com/documentation/foundation/propertylistencoder) don't conform to it. We encounter the `Encoder` protocol when we make our types conform to [`Encodable`](https://developer.apple.com/documentation/swift/encodable):
```swift
/// A type that can encode itself to an external representation.
public protocol Encodable {
    /// Encodes this value into the given encoder. 
    public func encode(to encoder: Encoder)
}
```

The protocol requires us to implement an `encode(to:)` method, which takes in an `Encoder`. 

What's going on here? 🤔🤔🤔

The answer lies in the architecture of `JSONEncoder`. 

Inspecting [the source code](https://github.com/apple/swift/blob/master/stdlib/public/Darwin/Foundation/JSONEncoder.swift) for `JSONEncoder`, we see it's a [`open` type](https://kaitlinmahar.com/2018/12/29/open-swift.html) that internally uses a `private` type `_JSONEncoder`, which *does* conform to `Encoder`. 

<div align="center">
<img src="/files/encoder-decoder/encoder-structure.png" width="400" align="center"><br>
<i>JSONEncoder uses a private _JSONEncoder.</i>
</div>
<br><br>

The actual code is rather complicated, but it boils down to something like this:
```swift
class JSONEncoder {
    func encode<T: Encodable>(_ value: T) throws -> Data {
        let privateEncoder = _JSONEncoder()
        try value.encode(to: privateEncoder)
        // do some processing to convert `privateEncoder`'s contents
        // to `Data`, and return it
    }
}
```

So that `encode(to:)` method you wrote for your `Encodable` type *is* being used -- it's just that a `_JSONEncoder` is passed in, and *not* a `JSONEncoder`.

The same is true on the decoding side - there is a [`Decoder`](https://developer.apple.com/documentation/swift/decoder) protocol, which we see in the [`Decodable`](https://developer.apple.com/documentation/swift/decodable) protocol:
```swift
/// A type that can decode itself from an external representation.
public protocol Decodable {
  /// Creates a new instance by decoding from the given decoder.
  init(from decoder: Decoder) throws
}
```

But [`JSONDecoder`](https://developer.apple.com/documentation/foundation/jsondecoder) and [`PropertyListDecoder`](https://developer.apple.com/documentation/foundation/propertylistdecoder) don't conform to `Decoder`, and instead internally use private `Decoder` types.

<div align="center">
<img src="/files/encoder-decoder/decoder-structure.png" width="400" align="center"><br>
<i>JSONDecoder uses a private _JSONDecoder.</i>
</div>
<br><br>

In code, it plays out similarly to the encoding case:
```swift
class JSONDecoder {
    func decode<T: Decodable>(_ type: T.Type, from data: Data) throws -> T {
        let privateDecoder = _JSONDecoder()
        // do some processing to store `data` in `privateDecoder`'s storage
        return try T(from: privateDecoder)
    }
}
```

## Ok... but why?

Ok, so now we see how the `Encodable` and `Decodable` protocols fit into `JSONEncoder` and `JSONDecoder`. But why were they designed that way? Why not just make `JSONEncoder` an `Encoder` too?

In short, the answer is that they provide very different APIs. The `JSONEncoder` API is designed to provide a single, simple entry point into encoding, and the `Encoder` protocol provides a completely different API for customizing how types are encoded. 

Consider `JSONEncoder`. All we really get is this one simple method: 
```swift
func encode<T: Encodable>(_ value: T) throws -> Data
```
And we know exactly what to do with it:
```swift
let s = MyType()
let sData = try JSONEncoder().encode(s)
```

There are also some configuration options available on `JSONEncoder`, such as [`outputFormatting`](https://developer.apple.com/documentation/foundation/jsonencoder/2895284-outputformatting) and [`keyEncodingStrategy`](https://developer.apple.com/documentation/foundation/jsonencoder/2949141-keyencodingstrategy). But that's about it.

On the other hand, the `Encoder` protocol requires a number of methods that provide *containers* for encoding values in:
```swift
func container<Key: CodingKey>(keyedBy type: Key.Type) -> KeyedEncodingContainer<Key>
func singleValueContainer() -> SingleValueEncodingContainer
func unkeyedContainer() -> UnkeyedEncodingContainer 
```

And those container types are themselves protocols, which provide their own various methods for storing values in them, like this:
```swift
protocol SingleValueEncodingContainer {
    mutating func encode(_ value: UInt16) throws
    mutating func encode(_ value: Int64) throws
    // ... repeat for several primitive Swift types
}
```

The methods on `Encoder` and [`SingleValueEncodingContainer`](https://developer.apple.com/documentation/swift/singlevalueencodingcontainer) (and the other container types, too) are meant to be used by `Encodable` types in their `encode(to:)` implementations to customize how exactly they're encoded. I'll talk about that more in a future blog post, but for now you can read about it [in the Swift documentation](https://developer.apple.com/documentation/foundation/archives_and_serialization/encoding_and_decoding_custom_types) (see "Encoding and Decoding Manually").

One of the authors of `Codable`, [Itai Ferber](https://itaiferber.net/), confirmed the reasoning behind the design:
<a class="embedly-card" href="https://www.reddit.com/r/swift/comments/a8jden/why_dont_jsonencoder_and_jsondecoder_conform_to/ecblk1e">Card</a>
<script async src="//embed.redditmedia.com/widgets/platform.js" charset="UTF-8"></script>

## Conclusion
In short, `JSONEncoder` and `JSONDecoder` use internal types conforming to `Encoder` and `Decoder` in order to split up portions of the encoding/decoding APIs, based on how and where you use them.

I hope you found this post helpful! Please feel free to ask questions or give feedback, either in the comments section or via [Twitter](https://www.twitter.com/k__mahar) or [email](mailto:kaitlinmahar@gmail.com).