---
layout: post
title: Being Lazy
date: 2016-02-28
categories: swift
---

Today we're gonna see how we can be more efficient ⚡️ by… being la💤y 😴.  
In particular, we'll talk about `lazy var` and `LazySequence`. And cats 😸.

![Lazy cat](/assets/lazy-cat.jpg){: height="200px" }
{: style="text-align: center"}

## The problem

Let's say you are making a chat app and want to represent your users using an avatar. You might have different resolutions for each avatar, so let's represent them this way:

```swift
extension UIImage {
  func resizedTo(size: CGSize) -> UIImage {
    /* Some computational-intensive image resizing algorithm here */
  }
}

class Avatar {
  static let defaultSmallSize = CGSize(width: 64, height: 64)

  var smallImage: UIImage
  var largeImage: UIImage

  init(largeImage: UIImage) {
    self.largeImage = largeImage
    self.smallImage = largeImage.resizedTo(Avatar.defaultSmallSize)
  }
}
```

The problem with this code is that we compute the `smallImage` during `init`, because the compiler enforces us to initialize every property of `Avatar` in `init`.

But maybe we won't even use this default values, because we'll provide the small version of the user's Avatar ourselves. So we'd have computed this default value, using a computational-intensive image-scaling algorithm, all for nothing!

## Possible solution

In Objective-C for similar cases we were used to use an intermediate private variable, in a technique which could be translate like this in Swift:

```swift
class Avatar {
  static let defaultSmallSize = CGSize(width: 64, height: 64)

  private var _smallImage: UIImage?
  var smallImage: UIImage {
    get {
      if _smallImage == nil {
        _smallImage = largeImage.resizedTo(Avatar.defaultSmallSize)
      }
      return _smallImage! // 🐴
    }
    set {
      _smallImage = newValue
    }
  }
  var largeImage: UIImage

  init(largeImage: UIImage) {
    self.largeImage = largeImage
  }
}
```

This way we can set a new `smallImage` if we want to, but if we access the `smallImage` property without assigning it a value before, it will compute one from the `largeImage` instead of returning `nil`.

That is exactly what we want. But that's also a lot of code to write. And imagine if we had more than two resolutions and wanted this behavior for all the alternate resolutions!

## Swift lazy initialization

![Lazy cat on keyboard](/assets/lazy-cat-keyboard.jpg){: height="200px" }
{: style="text-align: center"}

But thanks to Swift, we can now avoid all of this glue code above and do some lazy coding… by just declare our `smallImage` variable to be a `lazy` stored property!

```swift
class Avatar {
  static let defaultSmallSize = CGSize(width: 64, height: 64)

  lazy var smallImage: UIImage = self.largeImage.resizedTo(Avatar.defaultSmallSize)
  var largeImage: UIImage

  init(largeImage: UIImage) {
    self.largeImage = largeImage
  }
}
```

And just like that, using this `lazy` keyword, we achieve the exact same behavior with way less code to write!

* If we access the `smallImage` lazy var without affecting a specific value to it beforehand, then _and only then_ will the default value be computed then returned. Then if we access the property later again, the value will already have been computed once so it will just return that stored value.
* If we gave `smallImage` and explicit value before accessing it, then the computational-intensive default value will never be computed, and the explicit value we gave will be returned instead
* If we never access the `smallImage` property ever, its default value won't be computed either!

So that's a great and easy way to avoid useless initialization while still providing a default value and without the use of intermediate private variables! 🎉

## Initialization with a closure

As with any other properties, you can provide the default value for `lazy` vars using an in-place-evaluated closure too — using `= { /* some code */ }()` instead of just `= some code`. This is useful if you need multiple lines of code to compute that default value.

```swift
class Avatar {
  static let defaultSmallSize = CGSize(width: 64, height: 64)

  lazy var smallImage: UIImage = {
    let size = CGSize(
      width: min(Avatar.defaultSmallSize.width, self.largeImage.size.width),
      height: min(Avatar.defaultSmallSize.height, self.largeImage.size.height)
    )
    return self.largeImage.resizedTo(size)
  }()
  var largeImage: UIImage

  init(largeImage: UIImage) {
    self.largeImage = largeImage
  }
}
```

But because this is a `lazy` property, **you can reference `self` in there!** (note that this is true even if you don't use a closure, like in the previous example).

The fact that the property is `lazy` means that the default value will only be computed later, at a time when `self` will already be fully initialized, that's why it's ok to access `self` there — contrary to when you give default values to non-`lazy` properties which gets evaluated during the init phase.

ℹ️ _According to my experimentations, closures used for default values of `lazy` variables don't seem to strongly capture `self`, so there's no need to use `[unowned self]` in that closure to avoid a reference cycle._

## lazy let?

You **can't** create `lazy let` instance property in Swift to provide constants that would only be computed if accessed 😢. That's due to the implementation details of `lazy` which requires the property to be modifiable because it's somehow initialized without a value and then _change_ the value when it's accessed[^lazy-let].

[^lazy-let]: Some discussion are still ongoing in the Swift mailing lists about how to fix that and allow `lazy let` to be possible, but for now in Swift 2 that's how it is.

But as we're talking about `let`, one interesting feature about it is that `let` constants declared **at global scope** (and not as properties inside a type) are are automatically lazy (and also thread-safe)[^not-in-playground]:

[^not-in-playground]: Note that in a playground or in the REPL, as the code is evaluated like big `main()` function, declaring `let foo: Int` at top-level will not be considered a global constant and thus you won't observe this behavior. Don't get the special case of playgrounds or REPL fool you, in a real project those `let` global constants are really lazy.

```swift
// Global variable. Will be created lazily (and in a thread-safe way)
let foo: Int = {
  print("Initialized")
  return 42
}()

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {
  func application(application: UIApplication, didFinishLaunchingWithOptions launchOptions: [NSObject: AnyObject]?) -> Bool {
    print("Hello")
    print("Instance is: \(foo)")
    print("Bye")
    return true
  }
}
```
This code will print `Hello` first, then `Initialized` and `42`, then `Bye`; demonstrating that the `foo` value is only created when accessed, not before.

![Lazy cat on leash](/assets/lazy-cat-on-leash.gif){: height="200px" }
{: style="text-align: center" }

_⚠️ don't confuse that case with instance properties inside a class or struct. If you declare a `struct Foo { let bar = Bar() }` then the `bar` instance property will still be computed as soon as the `Foo` instance is created (as part of its initialization), not lazily._

## Another example: Sequences

Let's take another example, this time using a sequence / `Array` and some high-order function[^hof] like `map`:

[^hof]: "high-order functions" are funtions that either take another function as a parameter, or return a function (or both). Example of high-order functions are `map`, `flatMap`, `filter`, etc.

```swift
func increment(x: Int) -> Int {
  print("Computing next value of \(x)")
  return x+1
}

let array = Array(0..<1000)
let incArray = array.map(increment)
print("Result:")
print(incArray[0], incArray[4])
```

With this code, **even before** we access the `incArray` values, **all output values are computed**. So you're gonna see 1000 of those `Computing next value of …` lines even before the first `print` gets executed! Even if we only read values for `[0]` and `[4]` entries, and never care about the others… imagine if we used a more computationally intensive function than this simple `increment` one!

## Lazy sequences

![Lazy Cat in sequence](/assets/lazy-cats-sequence.jpg){: height="200px" }
{: style="text-align: center;" }

Ok so let's fix the above code with another kind of `lazy`.

In the Swift standard library, the `SequenceType` and `CollectionType` protocols have a computed property named `lazy` which returns a special `LazySequence` or `LazyCollection`. Those types are dedicated to only apply high-order functions like `map`, `flatMap`, `filter` and such, in a **lazy** way.[^lazy-internals]

[^lazy-internals]: In practice, those types just keep a reference to the original sequence and a reference to the closure to apply, and only do the actual computation of applying the closure on one element when that element is accessed.

Let's see how this work in practice:

```swift
let array = Array(0..<1000)
let incArray = array.lazy.map(increment)
print("Result:")
print(incArray[0], incArray[4])
```

Now this code only print this, demonstrating that it only apply the `increment` function when the values are accessed, and not when the call to `map` appears, and only apply them to the values being accessed, not all the one thousand values of the entire array!

```
Result:
Computing next value of 0…
Computing next value of 4…
1 5
```

That's way more efficient! This can change everything especially with big sequences (like this one with 1,000 items) and for computationally-intensive closures.[^each-time]

[^each-time]: But beware then — that according to my experimentations at least — the computed value isn't cached (no memoization); so if you request `incArray[0]` again it will compute the result again. We can't have it all… (yet?)

## Chaining lazy sequences

One last nice thing with lazy sequences is that you can of course combine the calls to high-order functions like you'd do with a Monad. For example you can call `map` or `flatMap` on a lazy sequences, like this:

```swift
func double(x: Int) -> Int {
  print("Computing double value of \(x)…")
  return 2*x
}
let doubleArray = array.lazy.map(increment).map(double)
print(doubleArray[3])
```

And this will only compute `double(increment(array[3]))` when this entry is accessed, not before, and only for this one!

_Whereas using `array.map(increment).map(double)[3]` instead (without `lazy`) would have computed all the output values of the whole `array` sequence, and only once all values have been computed, extract the 4th one. What a was of computational time for all those other 999 values!_

## Conclusion

Be lazy[^conclusion].

![Lazy cat on sofa](/assets/lazy-cat-on-sofa.jpg)
{: style="text-align: center" }

[^conclusion]: Yes I was too lazy to write a conclusion. But as this article just demonstrated, being lazy makes you a good programmer, right? 😜
