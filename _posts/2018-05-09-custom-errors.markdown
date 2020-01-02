---
layout: post
title:  Custom error messages in Swift 4
date:   2018-05-09 10:00:00 -0400
categories:
---

I always took error types with custom messages for granted in other languages, so I was a bit surprised at the amount of work I had to do to implement them in Swift. I had to spend a while digging through outdated Stack Overflow posts before coming upon this solution, so hopefully this will be useful to you!

The [Swift documentation](https://developer.apple.com/documentation/swift/error) on the [`Error` protocol](https://developer.apple.com/documentation/swift/error) spends a while discussing how to add associated values to error `enum`s, and even lists 

```swift
var localizedDescription: String
``` 

as part of the protocol. Yet, neither the associated values nor the `localizedDescription` are actually printed out when we throw an error! See the following example:

```swift
import XCTest

enum MyError: Error {
    case first(message: String)
    case second(message: String)

    var localizedDescription: String { return "Some description here!" }
}

final class ErrorTest: XCTestCase {
    static var allTests: [(String, (ErrorTest) -> () throws -> Void)] {
        return [
            ("testMyError", testMyError)
        ]
    }

    func testMyError() throws {
        throw MyError.first(message: "first error")
    }
}
```

This looks like it should print the associated messages, or maybe the `localizedDescription`, when we throw the error. And yet, my output, running the test from the command line:

`failed: caught error: The operation couldn’t be completed.(MyTests.MyError error 0.) `

While it does tell me that the first error case was thrown (`error 0`), I don't get any of the useful information I worked so hard to store with my error! 

The solution I found for this was that, rather than implementing `Error`, you can implement the [`LocalizedError`](https://developer.apple.com/documentation/foundation/localizederror) protocol instead (which inherits from `Error`.) This protocol specifies a property:

```swift
var errorDescription: String? { get }
```

which is printed out when the error is thrown. 

Modifying the previous example to implement `LocalizedError`, we get:
```swift
enum MyError: LocalizedError {
    case first(message: String)
    case second(message: String)

    var errorDescription: String? { return "Some description here!" }
}
```

When we run the same test as before, the output is now:
```
failed: caught error: Some description here!
```

Finally, what if we want to take advantage of our error's associated values to give better error messages? We can get fancier with the description, and do something like:
```swift
enum MyError: LocalizedError {
    case first(message: String)
    case second(message: String)

    var errorDescription: String? {
        switch self {
        case let .first(message), let .second(message):
            return message
        }
    }
}
```

Now, when we run the test, we get:
```
failed: caught error: first error
```
Exactly the message we specified when we originally threw the error!

Thanks for reading, and let me know any questions or feedback you have via Twitter or email 🙂
