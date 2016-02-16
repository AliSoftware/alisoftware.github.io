---
layout: post
title: Asynchronous error handling
date: 2016-02-06
categories: swift async error
---

In a previous article, I talked about [error handling in Swift using `throw`](http://alisoftware.github.io/2015/12/17/let-it-throw/). But what happens when you deal with asynchronous workflows, where `throw` can't really fit?

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

One way of working around this limitation is to make the `completion` return not directly a `User?`, but a throwing function `Void throws -> User` instead, itself returning a `User` (let's call this a `UserBuilder`). This way, we can use `throw` again.

Then when the completion returns the `userBuilder` func, we'll then need to call `try userBuilder()` to access the `User`… or have it `throw` on error.

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
    completion({ UserBuilder in
      if let error = error { throw error }
      guard let data = data else { throw UserError.NoData }
      return try User(fromData: data)
    })
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

Back in Swift 1.0, when `throw` didn't exist yet, people started to handle errors in a functional way. As Swift borrowed a lot of features from the functional programming world, it was logical that people used the `Result` pattern in Swift to better handle errors. Here is what `Result` looks like[^1]:

```swift
enum Result<T> {
  case Success(T)
  case Failure(ErrorType)
}
```

[^1]: This is only one possible implementation for `Result`. E.g. other implementations might allow to also specify a more specific error type.

The `Result` type is actually quite simple: it represents either a success — which has an associated value representing the successful result — or a failure — with an associated error. Perfect to represent operations which can potentially fail.

Then how do we use it? Just build either a `Result.Success` or a `Result.Failure` and call `completion` with the resulting `Result`[^return]:

```swift
func fetchUser(completion: Result<User> -> Void) {
  let url = …
  NSURLSession.sharedSession().dataTaskWithURL(url) { (data, response, error) -> Void in
    if let error = error {
      return completion( Result.Failure(error) )
    }
    guard let data = data else {
      return completion( Result.Failure(UserError.NoData) )
    }
    do {
      let user = try User(fromData: data)
      completion( Result.Success(user) )
    } catch {
      completion( Result.Failure(error) )
    }
  }.resume()
}
```

[^return]: Here I use a little trick by calling `return completion(…)` in this code instead of calling `completion(…)` then `return` to exit the function scope. This works because `completion` returns a `Void` and `fetchUser` returns a `Void` too (returns nothing), and because `return Void` is equivalent to just `return`. It's a matter of taste, but I think it's nice to be able to write this as a one-liner.

## Remember monads?

The nice thing about `Result` is it can be turned into a Monad. Remember [Monads](http://alisoftware.github.io/swift/2015/10/17/lets-talk-about-monads/)? That simply means that we can add the high-order `map` and `flatMap` methods on `Result`, which will take a closure `f: T->U` or `f: T->Result<U>` and return a `Result<U>`.

If the original `Result` was a `.Success(let t)` then we apply the closure to that `t` and return the resulting `f(t)`. If it was a `.Failure` then we simply pass the error along:

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

What is cool in that expression is that if one of the method (say `toJSON` for example) fails and return a `.Failure`, then the failure will be propagated to the end without even trying to apply the `extractUserDict` and `buildUser` methods.

This somehow allows the error to "take a shortcut": exactly like with `do…catch`, you can process all your error cases together in one point at the end of the chain instead of handling errors at each intermediate stage. Isn't that cool?


## From `Result` to `throw` and back

Problem is, `Result` is not built in the Swift standard library, and a lot of functions use `throw` to report synchronous errors anyway. Like in practice, to build a `User` from a `NSDictionary` we might have a `init(dict: NSDictionary) throws` constructor instead of a `NSDictionary -> Result<User>` function.

So how to mix both those worlds? Easy: let's extend `Result` just for that[^noescape]!

```swift
extension Result {
  // Return the value if it's a .Success or throw the error if it's a .Failure
  func resolve() throws -> T {
    switch self {
    case Result.Success(let value): return value
    case Result.Failure(let error): throw error
    }
  }

  // Construct a .Success if the expression returns a value or a .Failure if it throws
  init(@noescape _ throwingExpr: Void throws -> T) {
    do {
      let value = try throwingExpr()
      self = Result.Success(value)
    } catch {
      self = Result.Failure(error)
    }
  }
}
```

[^noescape]: In this code, the `@noescape` keyword means that the `throwingExpr` closure is guaranteed to be used directly in the scope of the `init` function before the `init` function returns — in contrast of being stored in a property and used later. This allows the compiler not to force you in using `self.` or `[weak self]` at call site when passing the closure, and be ensured retain cycles are avoided.

And now we can easily convert our throwing initializer to a closure returning a `Result`:

```swift
func buildUser(userDict: NSDictionary) -> Result<User> {
  // Here we build a `Result` by calling the `init` using a throwing trailing closure
  return Result { try User(dictionary: userDict) }
}
```

And if we wrap `NSURLSession` into a function asynchronously returning a `Result`, we can balance between the two world however we like, for example:

```swift
func fetch(url: NSURL, completion: Result<NSData> -> Void) {
  NSURLSession.sharedSession().dataTaskWithURL(url) { (data, response, error) -> Void in
    completion(Result {
      if let error = error { throw error }
      guard let data = data else { throw UserError.NoData }
      return data
    })
  }.resume()
}
```

Which also calls the `completion` block by passing a `Result` object built from a throwing closure[^like-builder].

[^like-builder]: Take a pause for a second there. See how this code looks really like the one we wrote using `UserBuilder` at the beginning of that article? Feels like we were on the right path 😉

Then we can chain it all using `flatMap`, and/or go back in the `do…catch` world, depending on our needs:

```swift
fetch(someURL) { (resultData: Result<NSData>) in
  let resultUser = resultData
    .flatMap(toJSON)
    .flatMap(extractUserDict)
    .flatMap(buildUser)

  // Then if we want to go back in the do/try/catch world for the rest of the code:
  do {
    let user = try resultUser.resolve()
    updateUI(user: user)
  } catch {
    print("Error: \(error)")
  }
}
```

## I Promise, this is the Future

`Result` is cool, but given they are mainly useful in asynchronous functions (since we already have `throw` for synchronous function), why not make it so it also manage the asynchronicity?

Well actually, there is a type for that™. It's called `Promise` (and sometimes `Future`, the two terms are very alike).

`Promise` is a type which kinda brings both the `Result` type (which can succeed or fail) and the asynchronicity together. A `Promise<T>` can either _fulfill_ with a successful value of type `T` when that value becomes available later (hence the async aspect of it), or be _rejected_ if an error occurred.

A `Promise` is also a monad. But usually instead of calling the monadic functions by the usual name `map` and `flatMap`, both are called `then` by convention:

```swift
class Promise<T> {
  // the monadic equivalent of "map", which is usually called "then" in the Promise type
  func then(f: T->U) -> Promise<U>
  // the monadic equivalent of "flatMap", which is usually called "then" too
  func then(f: T->Promise<U>) -> Promise<U>
}
```

And errors are unwrapped using `.error` or `.recover`.
At call site, it can mainly be used the same way you'd use a `Result`. They are both monads after all:

```swift
fetch(someURL) // returns a `Promise<NSData>`
  .then(toJSON) // assuming toJSON is now a `NSData -> Promise<NSDictionary>`
  .then(extractUserDict) // assuming extractUserDict is now a `NSDictionary -> Promise<NSDictionary>`
  .then(buildUser) // assuming buildUser is now a `NSDictionary -> Promise<User>`
  .then {
    updateUI(user: user)
  }
  .error { err in
    print("Error: \(err)")
  }
```

See how smooth and readable this looks? It's all about a stream of small processing steps chained together nicely. And it does all the heavy lifting of handling both the asynchronicity and the errors for you. If an error happens during the process, e.g. in `extractUserDict`, it jumps directly into the `error` block. Like with a `do…catch`. Or like with `Result`.

The implementation of `fetch` to use a `Promise` — instead of a completion block and a `Result` — would probably look something like this:

```swift
func fetch(url: NSURL) -> Promise<NSData> {
  // PromiseKit has a convenience `init` which turns a (T?, NSError?) closure into a `Promise`
  return Promise { resolve in
    NSURLSession.sharedSession().dataTaskWithURL(url) { (data, _, error) -> Void in
      resolve(data, error)
    })
  }.resume()
}
```

This `fetch` method will return immediately, so there is no `completionBlock` necessary. But it will return a `Promise` object, which will only execute the closure — given via the `then` — when the promise is _fulfilled_ and the data (asynchronously) arrives later.

## Observe and be Reactive

`Promises` are cool. But there is another concept which allows both that stream of small processing steps, handling asynchronicity, and handling and propagating errors whenever and wherever they occur during that stream.

This other concept is called Reactive Programming.
Some of you might already know `ReactiveCocoa` (RAC in short), or `RxSwift`.
Even if it's sharing some concepts with `Promises` (asynchronicity, error propagation, …), that's the next level beyond `Futures` & `Promises`: `Rx` allows multiple values to be emitted during time (instead of just one return value), and is a lot richer with tons of other features.

But that's a whole new subject, which is gonna be an exploration for another day!
