---
layout: post
title:  What does open mean in Swift?
date:   2018-12-28 23:00:00 -0400
categories:
---

You may have encountered an `open class` in a library you use, or maybe you've considered using one within your own code. `open` provides an additional access control level that's even less restrictive than `public`. But what exactly does `open` mean, and how is it different from `public`?

Here's what you need to know about `open`:

#### 1. `open` classes can be subclassed within modules that import them.
In contrast, `public` classes *cannot* be subclassed in modules that import them.

So if I have `ModuleA` with the following classes defined:
```swift
open class ClassA {}
public class ClassB {}
```

And I try to subclass them in my own module `ModuleB`:
```swift
import ModuleA

// this compiles just fine.
class SubclassA: ClassA {}

// this errors.
class SubclassB: ClassB {}
```

My subclass of the `open class` works just fine - but when I try to subclass the `public class`, I get `error: cannot inherit from non-open class 'ClassB' outside of its defining module`.

#### 2. `open` class members can be overridden within modules that import their classes.
(By "class members", I mean `class` properties and methods.)

In contrast, `public` class members *cannot* be subclassed in modules that import their classes.

Let's return to our previous `open class` and add a method to it:
```swift
open class ClassA {
    open func foo() {
        print("foo in base class")
    }
}
```

Now, when we import `ModuleA` into our own module, we can override `foo` in our subclass:
```swift
import ModuleA

class SubclassA: ClassA {
    override func foo() {
        print("foo in subclass")
    }
}
```

#### 3. Class members of an `open` class don't have to be `open` too, and they're `internal` by default. 
Just like with members of `public` classes, if you don't specify the access control level for a member of an `open` class, it will default to `internal`. 

If `ClassA` looked like this:
```swift
open class ClassA {
    open func foo() {
        print("foo in base class")
    }

    func bar() {
        print("bar in base class")
    }
}
```
I could override `foo` within `ModuleB` just fine like before, but trying to override `bar` would give me `error: method does not override any method from its superclass`, since there is no public or open method `bar`that we know about from within `ModuleB`. 

We also could've marked `bar` with an explicit level that wasn't `open`, like `private` or `public`, and the typical rules for those access levels would apply.

#### 4. `open` only applies to classes and class members.
Since `open` describes behavior related to inheritance, it only makes sense to use it with types that support inheritance - in Swift, that's just classes!

#### 5. Think carefully before marking anything in your library open.
There's a reason that `open` is not the default - it should only be used when you're *sure* that you want your users to be able to do what it allows.

The Swift language guide states:
> Marking a class as open explicitly indicates that you’ve considered the impact of code from other modules using that class as a superclass, and that you’ve designed your class’s code accordingly.

Basically, unless if there is a good, concrete reason for users to subclass your class, don't allow it! Once you make a library class `open`, changing it back to `public` becomes a breaking API change, as users may have already implemented subclasses that would no longer be allowed. 

It's worth noting that it's used very sparingly in the Swift standard library -- I only count [6 occurrences](https://github.com/search?l=Swift&q=%22open+class%22+repo%3Aapple%2Fswift+path%3Astdlib%2F&type=Code) of `open class`.

#### In conclusion
Making a class and its members `open` can be a useful tool if you want the ability to subclass/override outside of the module where the class is defined, but use it carefully!

For more information, I recommend reading the Swift language guide [chapter on access control](https://docs.swift.org/swift-book/LanguageGuide/AccessControl.html).

Please let me know any thoughts or questions you have in the comments or via Twitter or email!
