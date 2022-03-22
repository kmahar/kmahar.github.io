---
layout: post
title:  "Full-Stack Swift: Building an iOS App with a Vapor Backend"
date:   2022-03-23 12:00:00 -0400
categories:
---
I recently [revealed on Twitter](https://twitter.com/k__mahar/status/1494118397491269636) something that may have come as a surprise to many of my followers from the Swift/iOS community: I had never written an iOS app before! I've been writing Swift for a few years now but have focused entirely on [library development](https://www.youtube.com/watch?v=9-fdbG9jNt4) and [server-side Swift](https://www.swift.org/server/).

A highly compelling feature of Swift is that it allows you to write an iOS app and a corresponding backend – a complete, end-to-end application – all in the same language. This is similar to how using [Node.js](https://nodejs.org/en/) for a web app backend allows you to write Javascript everywhere.

To test this out and learn about iOS development, I decided to build a full-stack application entirely in Swift. I settled on a familiar CRUD app I've created a web version of before, an application that allows the user to manage a list of kittens and information about them.

I chose to build the app using the following components:
* A backend server, written using the popular Swift web framework [Vapor](https://vapor.codes/) and using the [MongoDB Swift driver](https://github.com/mongodb/mongo-swift-driver) via [MongoDBVapor](https://github.com/mongodb/mongodb-vapor) to store data in MongoDB
* An iOS application built with [SwiftUI](https://developer.apple.com/xcode/swiftui/) and using [SwiftBSON](https://github.com/mongodb/swift-bson) to support serializing/deserializing data to/from [extended JSON](https://docs.mongodb.com/manual/reference/mongodb-extended-json/), a version of JSON with MongoDB-specific extensions to simplify type preservation
* A [SwiftPM](https://www.swift.org/package-manager/) package containing the code I wanted to share between the two above components

I was able to combine all of this into a single code base with a folder structure as follows:
```
FullStackSwiftExample/
├── Models/
│   ├── Package.swift
│   └── Sources/
│       └── Models/
│           └── Models.swift
├── Backend/
│   ├── Package.swift
│   └── Sources/
│       ├── App/
│       │   ├── configure.swift
│       │   └── routes.swift
│       └── Run/
│           └── main.swift
└── iOSApp/
    └── Kittens/
        ├── KittensApp.swift
        ├── Utilities.swift
        ├── ViewModels/
        │   ├── AddKittenViewModel.swift
        │   ├── KittenListViewModel.swift
        │   └── ViewUpdateDeleteKittenViewModel.swift
        └── Views/
            ├── AddKitten.swift
            ├── KittenList.swift
            └── ViewUpdateDeleteKitten.swift
```

Overall, it was a great learning experience for me, and although the app is pretty basic, I'm proud of what I was able to put together! [Here](https://github.com/mongodb/mongo-swift-driver/tree/main/Examples/FullStackSwiftExample) is the finished application, instructions to run it, and documentation on each component.

In the rest of this post, I'll discuss some of my takeaways from this experience.

## 1. Sharing data model types made it straightforward to consistently represent my data throughout the stack.

As I mentioned above, I created a shared SwiftPM package for any code I wanted to use both in the frontend and backend of my application. In that package, I defined `Codable` types modeling the data in my application, for example:

```swift
/**
* Represents a kitten.
* This type conforms to `Codable` to allow us to serialize it to and deserialize it from extended JSON and BSON.
* This type conforms to `Identifiable` so that SwiftUI is able to uniquely identify instances of this type when they
* are used in the iOS interface.
*/
public struct Kitten: Identifiable, Codable {
   /// Unique identifier.
   public let id: BSONObjectID

   /// Name.
   public let name: String

   /// Fur color.
   public let color: String

   /// Favorite food.
   public let favoriteFood: CatFood

   /// Last updated time.
   public let lastUpdateTime: Date

   private enum CodingKeys: String, CodingKey {
       // We store the identifier under the name `id` on the struct to satisfy the requirements of the `Identifiable`
       // protocol, which this type conforms to in order to allow usage with certain SwiftUI features. However,
       // MongoDB uses the name `_id` for unique identifiers, so we need to use `_id` in the extended JSON
       // representation of this type.
       case id = "_id", name, color, favoriteFood, lastUpdateTime
   }
}
```

When you use separate code/programming languages to represent data on the frontend versus backend of an application, it's easy for implementations to get out of sync.  But in this application, since the same exact model type gets used for the frontend **and** backend representations of kittens, there can't be any inconsistency.

Since this type conforms to the `Codable` protocol, we also get a single, consistent definition for a kitten's representation in external data formats. The formats used in this application are:
* [Extended JSON](https://docs.mongodb.com/manual/reference/mongodb-extended-json/), which the frontend and backend use to communicate via HTTP, and
* [BSON](https://docs.mongodb.com/manual/reference/bson-types/), which the backend and MongoDB use to communicate

For a concrete example of using a model type throughout the stack, when a user adds a new kitten via the UI, the data flows through the application as follows:
1. The iOS app creates a new `Kitten` instance containing the user-provided data
1. The `Kitten` instance is serialized to extended JSON via `ExtendedJSONEncoder` and sent in a POST request to the backend
1. The Vapor backend deserializes a new instance of `Kitten` from the extended JSON data using `ExtendedJSONDecoder`
1. The `Kitten` is passed to the MongoDB driver method `MongoCollection<Kitten>.insertOne()`
1. The MongoDB driver uses its built-in `BSONEncoder` to serialize the `Kitten` to BSON and send it via the MongoDB [wire protocol](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/) to the database

With all these transformations, it can be tricky to ensure that both the frontend and backend remain in sync in terms of how they model, serialize, and deserialize data.  Using Swift everywhere and sharing these `Codable` data types allowed me to avoid those problems altogether in this app.

## 2. Working in a single, familiar language made the development experience seamless.

Despite having never built an iOS app before, I found my existing Swift experience made it surprisingly easy to pick up on the concepts I needed to implement the iOS portion of my application. I suspect it's more common that someone would go in the opposite direction, but I think iOS experience would translate well to writing a Swift backend too!

I used several Swift language features such as [protocols](https://docs.swift.org/swift-book/LanguageGuide/Protocols.html), [trailing closures](https://docs.swift.org/swift-book/LanguageGuide/Closures.html), and [computed properties](https://docs.swift.org/swift-book/LanguageGuide/Properties.html) in both the iOS and backend code. I was also able to take advantage of Swift's new built-in features for [concurrency](https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html) throughout the stack. I used the `async` APIs on [`URLSession`](https://developer.apple.com/documentation/foundation/urlsession) to send HTTP requests from the frontend, and I used Vapor and the MongoDB driver's `async` APIs to handle requests on the backend.  It was much easier to use a consistent model and syntax for concurrent, asynchronous programming throughout the application than to try to keep straight in my head the concurrency models for two different languages at once.

In general, using the same language really made it feel like I was building a single application rather than two distinct ones, and greatly reduced the amount of context-switching I had to do as I alternated between work on the frontend and backend. 

## 3. SwiftUI and iOS development are really cool!

Many of my past experiences trying to cobble together a frontend for school or personal projects using HTML and Javascript were frustrating.  This time around, the combination of using my favorite programming language and an elegant, declarative framework made writing the frontend very enjoyable. More generally, it was great to finally learn a bit about iOS development and what most people writing Swift and that I know from the Swift community do!

---

In conclusion, my first foray into iOS development building this full-stack Swift app was a lot of fun and a great learning experience. It strongly demonstrated to me the benefits of using a single language to build an entire application, and using a language you're already familiar with as you venture into programming in a new domain.

I've included a list of references below, including a link to the example application. Please feel free to get in touch with any questions or suggestions regarding the application or the MongoDB libraries listed below – the best way to get in touch with me and my team is by filing a [GitHub issue](https://github.com/mongodb/mongo-swift-driver/issues/new/choose) or [Jira ticket](http://jira.mongodb.org/browse/SWIFT)!

## References
* [Example app source code](https://github.com/mongodb/mongo-swift-driver/tree/main/Examples/FullStackSwiftExample)
* [MongoDB Swift driver](https://github.com/mongodb/mongo-swift-driver) and [documentation](https://mongodb.github.io/mongo-swift-driver/docs/current/MongoSwift/index.html)
* [MongoDBVapor](https://github.com/mongodb/mongodb-vapor) and [documentation](https://mongodb.github.io/mongodb-vapor/current/index.html)
* [SwiftBSON](https://github.com/mongodb/swift-bson) and [documentation](https://mongodb.github.io/swift-bson/docs/current/SwiftBSON/index.html)
* [Vapor](https://vapor.codes/)
* [SwiftUI](https://developer.apple.com/xcode/swiftui/)
