---
layout: post
title: StringInterpolation in Swift 5 — Introduction
date: 2018-12-15
categories: swift
swift_version: 5.0
---


In Swift 4, the `StringInterpolation` protocol got deprecated, because its original design was inefficient and inflexible, with the goal of redisigning it entirely after Swift 4. Since then, [SE-0228](https://github.com/apple/swift-evolution/blob/master/proposals/0228-fix-expressiblebystringinterpolation.md) introduced a new design for `StringInterpolation`, which is going to be part of Swift 5, and opens a whole lot of powerful possibilities.

This feature is part of Swift 5, so you will need Xcode 10.2 and Swift 5 to use it.

Note: This article is an introduction to the new `StringInterpolation`. A followup article with a more advanced example is available in [part 2 of this article](/swift/2018/12/16/swift5-stringinterpolation-part2/).

## The new StringInterpolation design

I really encourage you to read the [SE-0228](https://github.com/apple/swift-evolution/blob/master/proposals/0228-fix-expressiblebystringinterpolation.md) proposal to get an idea of the design and motivations behind the new API.

Basically, to make a type conform to `ExpressibleByStringInterpolation`, you have to:

* Make this type have a subtype `StringInterpolation`, that must itself conform to `StringInterpolationProtocol` and will be responsible to handle the interpretation of the interpolation
* That subtype just have to implement `appendLiteral(_ literal: String)` and one or more `appendInterpolation(…)` method, with the signature of your choosing depending on what you want to support
* This `StringInterpolation` subtype will serve as a "Builder Pattern" for your main type, and the compiler will call those `append…` methods to build the object step by step
* Then your main type needs to implement `init(stringInterpolation: StringInterpolation)` to instantiate itself with the result of those incremental steps.

The fact that you can implement whatever `appendInterpolation(…)` method you like means that you can choose what interpolation to support. This is a super powerful feature that opens a large range of possibilities!

For example, if you implement `func appendInterpolation(_ string: String, pad: Int)` that means that you'll be able to build your type using an interpolation like: `"Hello \(name, pad: 10), how are you?"`. The interpolation just has to match one of the `appendInterpolation` signatures that your `StringInterpolation` subtype support.

## Simple example

Let's start with a simple type to demonstrate how it goes. Let's build a `GitHubComment` type that would allow you to reference issue numbers and users.

The goal of that example is to be then able to write something like this:

```swift
let comment: GitHubComment = """
  See \(issue: 123) where \(user: "alisoftware") explains the steps to reproduce.
  """
```

So, how do we implement this?

First let's declare the basic `struct GitHubComment` and make it `ExpressibleByStringLiteral` (because `ExpressibleByStringInterpolation` inherits that protocol so let's get that implementation out of the way) and `CustomStringConvertible` (for nice debugging when printing in the console)

```swift
struct GitHubComment {
  let markdown: String
}

extension GitHubComment: ExpressibleByStringLiteral {
  init(stringLiteral value: String) {
    self.markdown = value
  }
}

extension GitHubComment: CustomStringConvertible {
  var description: String {
    return self.markdown
  }
}
```

Then, we'll make `GitHubComment` conform to `ExpressibleByStringInterpolation`, which means having a `StringInterpolation` subtype that will handle what to do when:

* initializing itself: `init(literalCapacity: Int, interpolationCount: Int)` lets you the possibility to reserve some capacity to the buffer you'll use while building the type step by step. In our case, we could simply have used a `String` and append the segments to it while building the instance, but I instead chose to use a `parts: [String]`, that we'll assemble together later
* implement `appendLiteral(_ string: String)` to append the literal text to the `parts`
* implement `appendInterpolation(user: String)` to append the markdown representation of a link to that user's profile when encountering `\(user: xxx)`
* implement `appendInterpolation(issue: Int)` to append the markdown representation of a link to that issue
* then implement `init(stringInterpolation: StringInterpolation)` on `GitHubComment` to build a comment from those `parts`

```swift
extension GitHubComment: ExpressibleByStringInterpolation {
  struct StringInterpolation: StringInterpolationProtocol {
    var parts: [String]

    init(literalCapacity: Int, interpolationCount: Int) {
      self.parts = []
      // - literalCapacity is the number of characters in literal segments (L)
      // - interpolationCount is the number of interpolation segments (I)
      // We estimate that we generally have a structure like "LILILIL"
      // — e.g. "Hello \(world, .color(.blue))!" — hence the 2n+1
      self.parts.reserveCapacity(2*interpolationCount+1)
    }

    mutating func appendLiteral(_ literal: String) {
      self.parts.append(literal)
    }

    mutating func appendInterpolation(user name: String) {
      self.parts.append("[\(name)](https://github.com/\(name))")
    }

    mutating func appendInterpolation(issue number: Int) {
      self.parts.append("[#\(number)](issues/\(number))")
    }
  }

  init(stringInterpolation: StringInterpolation) {
    self.markdown = stringInterpolation.parts.joined()
  }
}
```

And that's it! we're done!

Notice how, because of the signatures of the `appendInterpolation` methods we've implemented, we're allowed to use `Hello \(user: "alisoftware")` but not `Hello \(user: 123)`, as `appendInterpolation(user:)` expects a `String` as parameter. Similarly, `\(issue: 123)` in your string will only allow an `Int` because `appendInterpolation(issue:)` takes an `Int` as parameter.

In fact, if you try to use interpolations that are not supported by your `StringInterpolation` subtype, the compiler will give you nice errors:

```swift
let comment: GitHubComment = """
  See \(issue: "bob") where \(username: "alisoftware") explains the steps to reproduce.
  """
//             ^~~~~         ^~~~~~~~~
// error: cannot convert value of type 'String' to expected argument type 'Int'
// error: incorrect argument label in call (have 'username:', expected 'user:')
```

## It's only the beginning

This new design opens a large range of possibilities for making your own types `ExpressibleByStringInterpolation`. Some ideas include:

* Making a `HTML` type conform so that you can write HTML using interpolations
* Making a `SQLStatement` type conform so you can write SQL statements easily
* Using string interpolation to support more custom formatting, like formatting `Double` or `Date` values inside your interpolated string
* Making a `RegEx` type conform so that you can write regular expressions with a fancy syntax
* Making a `AttributedString` type conform to build an `NSAttributedString` using string interpolation

[Brent Royal-Gordon](https://github.com/brentdax), who  came up with that awesome design for the new String Interpolation alongside [Michael Ilseman](https://github.com/milseman), provided some more example in [this gist](https://gist.github.com/brentdax/0b46ce25b7da1049e61b4669352094b6)

Personally I gave a shot at building NSAttributedStrings using StringInterpolation, and found the result really beautiful 🤩 So I am really excited to share it with you in [part 2 of this post](/swift/2018/12/16/swift5-stringinterpolation-part2/) 🙂 

See you there! 👋
