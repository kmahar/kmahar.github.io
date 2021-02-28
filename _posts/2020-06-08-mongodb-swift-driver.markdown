---
layout: post
title:  Announcing the 1.0 Release of the Official MongoDB Swift Driver
date:   2020-06-08 08:00:00 -0400
categories:
---

_This is a cross-post from the MongoDB blog. You can find the original article [here](https://www.mongodb.com/blog/post/announcing-release-official-swift-driver)._

---

I'm very excited to announce the 1.0 release of the official [MongoDB driver for Swift](https://github.com/mongodb/mongo-swift-driver)! The driver is now GA, ready for use in production, and fully supported by MongoDB.

In this blog post, I'll cover the history of the driver, features and use cases we support, how you can get started using and contributing to the driver, and our future plans for the project.

# Evolution of the Driver
My team and I initially began developing this library in 2018 to support the use of MongoDB Mobile on iOS. MongoDB's mobile efforts were refocused with the acquisition of [Realm](https://www.mongodb.com/realm) in 2019, but the year we spent working with the Swift community gave us valuable insight nevertheless. Once we better understood Swift's potential beyond iOS as a great general-purpose programming language, we started to think about how we could best support Swift developers writing server-side applications with MongoDB.

With new use cases for the driver in mind, we turned to the community and the [Swift Server Work Group](https://swift.org/server/) (SSWG), a steering team for server-side Swift efforts. After a number of conversations, both on the [Swift Forums](https://forums.swift.org/) and at conferences like [ServerSide.swift](https://www.serversideswift.info/), we submitted a [proposal](https://github.com/swift-server/sswg/blob/main/proposals/0010-mongo-swift-driver.md) for the driver to take part in the SSWG [incubation process](https://github.com/swift-server/sswg/blob/main/process/incubation.md) for libraries. All of this generated invaluable feedback from both members of the workgroup and the Swift community at large and helped us refine our API to integrate better with the rest of the server-side ecosystem.

Now that our proposal has been accepted, we look forward to taking part in the incubation process and iterating through feedback and direction from the SSWG, and we hope to ultimately reach the process's graduation stage.

# Supported Features and Use Cases
The server-side Swift ecosystem is decidedly asynchronous first, so we are releasing a fully asynchronous 1.0 driver based on Apple's [SwiftNIO](https://github.com/apple/swift-nio). This means you can expect our APIs to work seamlessly with popular asynchronous libraries and frameworks like [Vapor](https://vapor.codes/).

We also strongly believe in the potential of Swift as a language for synchronous use cases, such as scripts and command line utilities, so we provide a synchronous API as well.

The driver has Swifty APIs for everything from [BSON](https://docs.mongodb.com/manual/reference/bson-types/) to [change streams](https://docs.mongodb.com/manual/changeStreams/) to [aggregation](https://docs.mongodb.com/manual/aggregation/) and supports connection pooling, application performance monitoring, and automatically retrying failed commands.

In addition to providing a flexible BSON document type, the driver leverages Swift’s [`Codable`](https://developer.apple.com/documentation/swift/codable) types, enabling use of custom types directly with MongoDB collections without needing a separate ODM (object-document mapper) library.

The minimum Swift version the driver supports is 5.1. The driver can be used with all MongoDB versions 3.6 and higher, including 4.4, though there are some newer server features we haven't exposed APIs for yet. You can expect to see those added in upcoming minor versions.

You can view our full API, along with guides on how to use various aspects of the API, on our [documentation website](https://mongodb.github.io/mongo-swift-driver/docs/current/MongoSwift/index.html).

# Getting Started
You can include the driver as a dependency in your application via [Swift Package Manager](https://swift.org/package-manager/):

```swift
// swift-tools-version:5.1
import PackageDescription

let package = Package(
    name: "MyPackage",
    dependencies: [
        .package(url: "https://github.com/mongodb/mongo-swift-driver", .upToNextMajor(from: "1.0.0")),
        .package(url: "https://github.com/apple/swift-nio", .upToNextMajor(from: "2.0.0"))
    ],
    targets: [
        .target(name: "MyTarget", dependencies: ["MongoSwift", "NIO"])
    ]
)
```

In order to use the asynchronous API, you'll need a SwiftNIO [`EventLoopGroup`](https://apple.github.io/swift-nio/docs/current/NIO/Protocols/EventLoopGroup.html), so you'll need to include SwiftNIO as a dependency, too. (For synchronous usage, you can omit that dependency and depend on the `MongoSwiftSync` module.)

You can then run `swift build` to download your dependencies and compile your application.

To create a client and perform some basic operations:
```swift
import MongoSwift
import NIO

// Create an EventLoopGroup and client
let elg = MultiThreadedEventLoopGroup(numberOfThreads: 4)
let client = try MongoClient("mongodb://localhost:27017", using: elg)

// Application cleanup code
defer {
    try? client.syncClose()
    cleanupMongoSwift()
    try? elg.syncShutdownGracefully()
}

// Define a struct that matches the data in our collection
struct Cat: Codable {
    let name: String
    let color: String
    let _id: BSONObjectID
}

let db = client.db("test")
let cats = db.collection("cats", withType: Cat.self)

let data = [
    Cat(name: "Chester", color: "tan", _id: BSONObjectID()),
    Cat(name: "Roscoe", color: "orange", _id: BSONObjectID())
]

// insert values into the collection
let insert = cats.insertMany(data)

insert.whenSuccess { result in
    print(result?.insertedCount) // prints 2
}

// create a cursor over the collection to read back the documents
let result = insert.flatMap { _ in
    cats.find()
}.flatMap { cursor in
    cursor.forEach { cat in
        print(cat) // prints out a `Cat` struct
    }
}
```

Please see our [README](https://github.com/mongodb/mongo-swift-driver/blob/main/README.md) for more details. Our GitHub repository also contains [example applications](https://github.com/mongodb/mongo-swift-driver/tree/main/Examples) demonstrating how to integrate the driver with server-side Swift frameworks like [Vapor](https://vapor.codes/).

# Next Steps for the Team
Currently, the driver uses the [MongoDB C Driver](https://github.com/mongodb/mongo-c-driver) (also known as "libmongoc") underneath the hood. `libmongoc` has provided a solid, reliable core, and depending on it allowed us to focus on building a great API. Now that we've achieved API stability, we will shift our focus to writing driver internals in Swift. We expect this to enhance driver performance and to speed up our own development process. Meanwhile, we'll continue adding support to the driver for the latest MongoDB versions.

We are grateful for the opportunity we have to give back to the server-side Swift community. We plan to make as much of our work as possible useful to the ecosystem and want to collaborate with the SSWG and the community at large whenever we can on our upcoming projects. As an example, we'll need to write a SwiftNIO-based connection pool for the driver to use, based on our connection pooling [specification](https://github.com/mongodb/specifications/blob/master/source/connection-monitoring-and-pooling/connection-monitoring-and-pooling.rst). We'd like to implement that as a standalone library which can serve as a reference implementation for other Swift developers.

# Getting Involved
If you have any questions, feedback, or ideas you'd like to discuss with us, you can contact us either through our [JIRA project](https://jira.mongodb.org/projects/SWIFT/issues) or through a [GitHub](https://github.com/mongodb/mongo-swift-driver) issue.

We're happy to accept GitHub pull requests, too -- check out our [Development Guide](https://github.com/mongodb/mongo-swift-driver/blob/main/Guides/Development.md) for information on how to build and test the driver.

We'd love for you to try out the driver and let us know what you think. Thanks so much for reading!
