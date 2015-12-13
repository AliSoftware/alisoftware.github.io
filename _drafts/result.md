## As a Result…

When Swift and its new approach arrived back in Swift 1.0, people started to handle errors in a better way. As Swift borrowed a lot of features from the functional programming world, it was logical that people used the `Result` pattern in Swift to better handle errors(^1)

```swift
enum Result<T> {
  case Success(T)
  case Failure(NSError)
}
```

(^1): this is one possible implementation for `Result`. Other implementations allow to also specify an error type other than `NSError`, we'll see that later when we introduce `ErrorType`.




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


