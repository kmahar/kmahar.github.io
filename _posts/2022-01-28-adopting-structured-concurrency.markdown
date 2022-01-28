---
layout: post
title:  Adopting Structured Concurrency in the MongoDB Swift Driver
date:   2022-01-28 10:00:00 -0400
categories:
---

Swift 5.5 introduced built-in support for writing structured, asynchronous, concurrent code, which you can find a great overview of in the [Swift Language Guide](https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html). While in prior versions of Swift it was possible to write this type of code without the built-in support, such code used paradigms such as futures/promises and callbacks and was harder to read, understand, and reason about. 

These new features are very exciting, and we've just begun to adopt them in [the MongoDB Swift driver](https://github.com/mongodb/mongo-swift-driver). As we've just put out a [version 1.3.0 pre-release](https://github.com/mongodb/mongo-swift-driver/releases/tag/v1.3.0-alpha.1), I wanted to write a short blog post highlighting the new features structured concurrency has enabled us to add so far.

# Async/Await APIs

Our users could previously only write asynchronous code by using our API methods that return [SwiftNIO](https://github.com/apple/swift-nio)'s [`EventLoopFuture`](https://apple.github.io/swift-nio/docs/current/NIOCore/Classes/EventLoopFuture.html)s, and then using API methods on that type to register callbacks. For example, code to create a collection, insert a document to it, and then print the resulting document's `_id` field would look something like:

```swift
let result = db.createCollection(
    "kittens",
    withType: Kitten.self
).flatMap { collection in
   collection.insertOne(Kitten(name: "Chester", color: "tan"))
}.flatMapThrowing { result in
   print(result?.insertedID)
}
```

We have now added new, `async` versions of every API method which can be used with Swift's new async/await syntax to allow writing this code in a more natural and readable manner. The example above would become:

```swift
let collection = try await db.createCollection("kittens", withType: Kitten.self)
let result = try await collection.insertOne(Kitten(name: "Chester", color: "tan"))
print(result?.insertedID)
```

# Improved Cursor and Change Stream Ergonomics

Structured concurrency has also enabled us to improve the experience of working with two driver types that produce sequences of values, `MongoCursor` and `ChangeStream`. 

Previously, user code to loop over the values in a cursor or change stream and print each one, and then close the cursor might look like this: 

```swift
let result = coll.find().flatMap { cursor in
   cursor.forEach { doc in
       print(doc)
   }.always { _ in _ = cursor.kill() }
}
```

Through a `forEach` method we added to these types, users were able to register a callback that we would execute for each value in the sequence. While this allowed users to get the job done, it was not ideal that we needed to provide a custom version of a method that would be generally useful for any asynchronous sequence. It would also be non-trivial to write more complicated looping logic, for example to break out of the loop after encountering a particular value.

To provide an easier way to work with the sequences of values cursors and change streams produce, we added conformance to the new [`AsyncSequence`](https://developer.apple.com/documentation/swift/asyncsequence) protocol for these types. This protocol provides a number of conveniences for working with asynchronously produced sequences of values, including a familiar, simpler way to loop over the values using a for-in loop:

```swift
for try await doc in try await coll.find() {
   print(doc)
}
```

In the future-based API, users were required to add logic to "kill" the cursor when they were done using it before it went out of scope. This was necessary as proper cleanup of this type can require communicating with the connected MongoDB deployment in order to instruct it to destroy its corresponding resources. Ideally, resource cleanup would happen in the type's `deinit`, but in the past there was no easy way for us to kick off such work in a non-blocking manner from `deinit`. 

With structured concurrency, we now have a more straightforward way to do this by creating a new `Task`:

```swift
deinit {
   Task {
       // ... kick off asynchronous cleanup work here
   }
}
```

Now, on any Swift versions and platforms where structured concurrency is available, we can automatically do this cleanup for users. One caveat to note is that background cleanup prevents users from synchronizing cleanup with other operations since there is no way to tell it is complete, but we don't expect that to be a common need, and the `kill()` method is still available if this is necessary (if this method has been called, we won't re-attempt cleanup in `deinit`).

With these new features, the example above becomes:

```swift
for try await doc in try await coll.find() {
   print(doc)
}
// cursor cleaned up in the background here
```

For long-running cursors and change streams where this loop may continue indefinitely, we've also made it straightforward to gracefully interrupt such a loop by canceling the `Task` the cursor or change stream is running in. For example:

```swift
let changeStreamTask = Task {
   for try await changeEvent in try await coll.watch() {
       // process change event
   }
}

// later
changeStreamTask.cancel()
```

Under the hood, the loop above repeatedly calls the `AsyncSequence` protocol method `next()`, which returns either the next value in the sequence or `nil` if the end of the sequence has been reached. In our implementation of that protocol method, we check whether the `Task` is cancelled via `Task.isCancelled`, and if so, return `nil`. This will terminate any loops over the sequence and allow their `Task`s to complete.  For this reason, we now recommend using long-running cursors and change streams in their own dedicated `Task`s.

# Future Directions

Our primary goal with this release was to provide an async version of our existing future-based API. That said, going forward we think there is room to improve other aspects of our API using these new features as well. For example, the driver provides an API for subscribing to command events, which currently allows registering either a callback or a custom listener type to process events. In the future, we'd like to allow users to subscribe to these via an `AsyncSequence`, like:

```swift
for try await event in client.commandEvents {
   // process event
}
```

In general, as Swift concurrency matures we will be following along and ensuring the driver continues to integrate well with the rest of the ecosystem. If you have any suggestions for improvement, please reach out!


# Trying it Out

You can try this out today by adding the pre-release tag `1.3.0-alpha.1` as a dependency in your `Package.swift` file:

```swift
// swift-tools-version:5.5
import PackageDescription

let package = Package(
   name: "MyPackage",
   platforms: [
       .macOS(.v12)
   ],
   dependencies: [
       .package(url: "https://github.com/mongodb/mongo-swift-driver", .exact("1.3.0-alpha.1"))
   ],
   targets: [
       .target(
           name: "MyTarget",
           dependencies: [
               .product(name: "MongoSwift", package: "mongo-swift-driver")
           ]
       )
   ]
)
```
Or, if you're using [Vapor](https://vapor.codes/), we recommend consuming the driver via [`MongoDBVapor`](https://github.com/mongodb/mongodb-vapor), a small library integrating the driver with Vapor. In that case, you can depend on the latest alpha release of that library:

```swift
// swift-tools-version:5.5
import PackageDescription

let package = Package(
   name: "MyPackage",
   platforms: [
       .macOS(.v12)
   ],
   dependencies: [
       .package(url: "https://github.com/mongodb/mongodb-vapor", .exact("1.1.0-alpha.1")),
       .package(url: "https://github.com/vapor/vapor", .upToNextMajor(from: "4.50.0"))
   ],
   targets: [
       .target(
           name: "MyTarget",
           dependencies: [
               .product(name: "MongoDBVapor", package: "mongodb-vapor")
           ]
       )
   ]
)
```

We have a complete example project using the driver and Vapor's async/await APIs as well, available [here](https://github.com/mongodb/mongo-swift-driver/tree/main/Examples/VaporExample).

We've also updated our [README](https://github.com/mongodb/mongo-swift-driver/blob/main/README.md) and [documentation guides](https://mongodb.github.io/mongo-swift-driver/docs/current/MongoSwift/General%20Guides.html) to contain examples using async/await syntax. 

If you have any questions or feedback for us, please feel free to get in touch, either by filing an issue on our [GitHub repo](https://github.com/mongodb/mongo-swift-driver/issues) or our [Jira project](https://jira.mongodb.org/browse/SWIFT).

# References
* [MongoDB Swift driver](https://github.com/mongodb/mongo-swift-driver)
    * [API documentation](https://mongodb.github.io/mongo-swift-driver/docs/current/MongoSwift/index.html)
* [MongoDBVapor](https://github.com/mongodb/mongodb-vapor)
    * [API documentation](https://mongodb.github.io/mongodb-vapor/current/index.html)
* [Swift Language Guide - Concurrency](https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html)
* [AsyncSequence](https://developer.apple.com/documentation/swift/asyncsequence)
* [SwiftNIO](https://github.com/apple/swift-nio)
* [Vapor](https://vapor.codes/)
