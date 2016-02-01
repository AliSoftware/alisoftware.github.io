---
layout: post
title: Asynchronous error handling
date: 2016-02-02
categories: swift error
---

In previous articles, I talked about [error handling in Swift using `throw`]() and [`rethrows`](). But what happens when you deal with asynchronous workflows, when `throw` doesn't fit?

## What's wrong with `throw` and async?

As a reminder, a function which can fail can use `throw` in the following way:

```swift
// Define an error type and a throwing function
enum ComputationError: ErrorType { case DivisionByZero }
func inverse(x: Float) throws -> Float {
  guard x != 0 else { throw ComputationError.DivisionByZero }
  return 1.0/x
}
// Call it
do {
  let y = try inverse(5.0)
} catch {
  print("Woops: \(error)")
}
```

But what happens when the function is asynchronous and will only return a result **later**, e.g. with a completion block?

```swift
func fetchUser(completion: User? /* throws */ -> Void) /* throws */ {
  let url = …
  NSURLSession.sharedSession().dataTaskWithURL(url) { (data, response, error) -> Void in
//    if let error = error { throw error } // No can do, fetchUser can't "throw asynchronously"
    let user = data.map { User(fromData: $0) }
    completion(user)
  }.resume()
}
// Call it:
fetchUser() { (user: User?) in
  /* do something */
}
```

How can you `throw` in this situation in case the request failed?

* It doesn't make sense to make the `fetchUser` function itself to `throw`, because this function returns immediately, but the network error would only happen later. So when the error occurs, it would be too late to `throw` an error as the result of the `fetchUser` function call itself.
* Maybe you can mark the `completion` as `throws`? But the code doing the call to that `completion(user)` method is inside the `fetchUser`, it's not at call site. So the one receiving and facing to handle the error would be the code inside `fetchUser` itself, not the call site. So that doesn't solve it either. 😢

## Hacking it

One way of working around this limitation is to make the `completion` return not directly a `User?`, but a throwing function `Void throws -> User` instead, itself returning a `User` (let's call this a `UserBuilder`). This way, we can use `throw`s again. Then when the completion returns the `userBuilder` func, we'll then need to call `try userBuilder()` to access the `User`… or have it `throw` on error.

```swift
enum UserError: ErrorType { case NoData, ParsingError }
struct User {
  init(fromData: NSData) throws { /* … */ }
  /* … */
}

typealias UserBuilder = Void throws -> User
func fetchUser(completion: UserBuilder -> Void) {
  let url = …
  NSURLSession.sharedSession().dataTaskWithURL(url) { (data, response, error) -> Void in
    let userBuilder = { UserBuilder in
      if let error = error {
        throw error
      } else if let data = data {
        return try User(fromData: data)
      } else {
        throw UserError.NoData
      }
    }
    completion(userBuilder)
  }.resume()
}

fetchUser { (userBuilder: UserBuilder) in
  do {
    let user = try userBuilder()
  } catch {
    print("Async error while fetching User: \(error)")
  }
}
```

This way the completion does not directly return a `User` but a function returning a `User`… or throwing. And then you have your error handling again.

But let's be honest, that's not the prettiest and most readable solution, having to return a `Void throws -> User` instead of returning a `User?` directly. So what are our other alternatives?

## Introducing Result

Back in Swift 1.0, when `throw` didn't exist yet, people started to handle errors in a functional way. As Swift borrowed a lot of features from the functional programming world, it was logical that people used the `Result` pattern in Swift to better handle errors[^1]. Here is what `Result` looks like:

```swift
enum Result<T> {
  case Success(T)
  case Failure(ErrorType)
}
```

[^1]: This is only one possible implementation for `Result`. Other implementations might allow to also specify a more specific error type for example.

The `Result` type is actually quite simple: it simply represents either a success — which has an associated value representing the successful result — or a failure — with an associated error. Perfect to represent operations which can potentially fail.

Then how do we use it? That's actually quite simple: build either a `Result.Success` or a `Result.Failure` and call `completion` with the resulting `Result`:

```swift
func fetchUser(completion: Result<User> -> Void) {
  let url = NSURL(string: "")!
  NSURLSession.sharedSession().dataTaskWithURL(url) { (data, response, error) -> Void in
    if let error = error {
      completion( Result.Failure(error) )
    } else if let data = data {
      do {
        let user = try User(fromData: data)
        completion( Result.Success(user) )
      } catch {
        completion( Result.Failure(error) )
      }
    } else {
      completion( Result.Failure(UserError.NoData) )
    }
  }.resume()
}
```

## From Result to `throw`

What can get annoying is that when you call `fetchUser` later, you then have to use `switch` to check if the returned value is a `case Result.Success(let user)` or a `case Result.Failure(let error)`, which might not combine nicely with other functions that use the `throw` mecanism of Swift.
But we can actually improve that by providing ways to convert from one world to another.

Let's start by adding a way to go from `Result` to `throw`:

```swift
extension Result {
  func resolve() throws -> T {
    switch self {
    case Result.Success(let value): return value
    case Result.Failure(let error): throw error
    }
  }
}
```

Then the other way around, building a `Result` from a throwing call:

```swift
extension Result {
  init(_ throwingExpr: Void throws -> T) {
    do {
      let value = try throwingExpr()
      self = Result.Success(value)
    } catch {
      self = Result.Failure(error)
    }
  }
}
```

We can now convert a throwing method to a result and back:

```swift
let user = User(admin: false)
let result = Result { try user.performAdminTask() }
// result is equal to Result.Failure(.AccessDenied)
do {
  try result.resolve() // This will fail as it is a Result.Failure
} catch {
  print("Error: \(error)") // so we'll print "AccessDenied"
}
```



## Remember monads?

The nice thing about `Result` is it can be turned into a Monad. Remember [Monad](http://alisoftware.github.io/swift/2015/10/17/lets-talk-about-monads/)? That simply means that we can add the high-order `map` and `flatMap` methods on `Result`, which will take a closure `T->U` or `T->Result<U>` and return a `Result<U>`. If the original `Result` was a `.Success(let t)` then we apply the closure to it and return the result. If it was a `.Failure` then we simply pass that failure along:

```swift
extension Result {
  func map<U>(f: T->U) -> Result<U> {
    switch self {
    case .Success(let t): return .Success(f(t))
    case .Failure(let err): return .Failure(err)
    }
  }
  func flatMap<U>(f: T->Result<U>) -> Result<U> {
    switch self {
    case .Success(let t): return f(t)
    case .Failure(let err): return .Failure(err)
    }
  }
}
```

For more information I invite you to re-read my article about Monads, but long story short, this allows us to do stuff like this:

```swift
func readFile(file: String) -> Result<NSData> { … }
func toJSON(data: NSData) -> Result<NSDictionary> { … }
func extractUserDict(dict: NSDictionary) -> Result<NSDictionary> { … }
func buildUser(userDict: NSDictionary) -> Result<User> { … }

let userResult = readFile("me.json")
  .flatMap(toJSON)
  .flatMap(extractUserDict)
  .flatMap(buildUser)
```

What is cool in that expression is that if one of the method (say `toJSON` for example) fails and return a `.Failure`, then the failure will be propagated to the end without even trying to apply the `extractUserDict` and `buildUser` methods. This somehow allows the error to "take a shortcut".

No need for you to handle intermediate failures at each stage like you did before, you will get it at the end eventually, meaning that you can manage all your error cases in one point instead of repeating the error-handling code at each intermediate stage. Isn't that cool?


