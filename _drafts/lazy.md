---
layout: post
title: Being Lazy
date: 2016-02-28
categories: swift
---

Today we're gonna see how we can be more efficient by… being lazy.

## Providing a default value lazily

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

## Possible solutions

Some solutions to this problem include:

* Make `smallImage` a computed property so it's only be computed when accessed, not before. But it would then be computed **each time** we access it, and we wouldn't be able to let the user provide their own small image if they wanted to.
* We could make `smallImage` optional (`UIImage?`) so that we are not forced to initialize them during `init`. But then how could we provide them default values?

So in Objective-C for similar cases we were used to use an intermediate private variable, in a technique which could be translate like this in Swift:

```swift
class Avatar {
  static let defaultSmallSize = CGSize(width: 64, height: 64)

  private var _smallImage: UIImage?
  var smallImage: UIImage {
    get {
      if _smallImage == nil {
        _smallImage = largeImage.resizedTo(Avatar.defaultSmallSize)
      }
      return _smallImage! // 🐴😵
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

This way we can set a new `smallImage` we we want to, but if we access the `smallImage` property without assigning it a value before, it will compute one from the `largeImage` instead of returning `nil`.

That is exactly what we want. But that's also a lot of code to write. And imagine if we had more than two resolutions and wanted this behavior for all the alternate resolutions!

## Swift lazy initialization

But thanks to Swift, we can now avoid all of this glue code above, and instead just declare our `smallImage` variable to be a… `lazy` stored property!

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

If we access the `smallImage` lazy var without affecting a specific value to it beforehand, then and only then will the default value `self.largeImage.resizedTo(Avatar.defaultSmallSize)` be computed:

* If we gave `smallImage` and explicit value before accessing it, then the computational-intensive default value will never be computed, and the explicit value we gave will be returned instead
* If we never access the `smallImage` property ever, its default value won't be computed either!

So that's a great and easy way to avoid useless initialization while still providing a default value and without the use of intermediate private variables!

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

## lazy let?

You **can't** create `lazy let` instance property in Swift to provide constants that would only be computed if accessed 😢. That's due to the implementation details of `lazy` which requires the property to be modifiable because it's somehow initialized without a value and then _change_ the value when it's accessed[^lazy-let].

[^lazy-let]: Some discussion are still ongoing in the Swift mailing lists about how to fix that and allow `lazy let` to be possible, but for now in Swift 2 that's how it is.

But as we're talking about `let`, one interesting feature about it is that `let` constants declared **at global scope** (and not as properties inside a type) are are automatically lazy (and also thread-safe)[^not-in-playground]:

[^not-in-playground]: Note that if you try this in a playground or in the REPL, as the code is evaluated "as you type" and that the whole playground is somehow acting as a big `main()` function, in such contexts `let foo: Int` will not be considered a global constant and you won't observe this behavior. Don't get the special case of playgrounds or REPL fool you, in a real project those `let` global constants are really lazy.

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
This code will print `Hello`, `Initialized`, `42` then `Bye`, demonstrating that the `foo` value is only created when accessed, not before.

_⚠️ don't confuse that case with instance properties inside a class or struct. If you declare a `struct Foo { let bar = Bar() }` then the `bar` instance property will still be computed as soon as the `Foo` instance is created (as part of its initialization), not lazily._

## Lazy sequences

But let's not stop there! The Swift standard library also comes with another handy feature called "lazy sequences".

What are those? Well in short, that a way to apply operations like `map`, `flatMap` and such operations on sequences (like on an `Array`) lazily, only applying the computation on the sequence items when they are accessed.

So let's take a simple example:

```swift
func increment(x: Int) -> Int {
  print("Computing next value of \(x)")
  return x+1
}

let array = Array(0..<10)
let incArray = array.map(increment)
print("Result:")
print(incArray[0], incArray[4])
```

In this code, we create an array of values from 0 to 9, then apply `increment` to each of the values.

With this, **even before** we access the `incArray` values, **all values are computed**. So you're gonna see 10 of those `Computing next value of …` lines even before the first `print` gets executed. But then we only read values for `[0]` and `[4]` entries, and never care about the others! So imagine if that array was not containing `10` but `1,000` values and that we used a more computationally intensive function than this simple `increment` one!

## How do lazy sequences work

Ok so let's fix the above code with `lazy`. In the Swift standard library, the `SequenceType` and `CollectionType` protocols have a computed property named `lazy` which returns a special `LazySequence` or `LazyCollection`. Those types are especially dedicated to only apply high-order functions like `map`, `flatMap`, `filter` and such, in a **lazy** way.

In practice, those types just keep a reference to the original sequence and another to the closure to apply, and only do the actual computation of applying the closure on one element when that element is accessed. So let's see how this work:

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

That's way more efficient, especially for big sequences (like this one with 1,000 items) and for computationally-intensive closures.

## Chain lazy sequences

One last nice thing with lazy sequences is that you can of course combine the calls to high-order functions like you'd do with a Monad. For example you can call `map` and `flatMap` on a lazy sequence like this:

```swift
func double(x: Int) -> Int {
  print("Computing double value of \(x)…")
  return 2*x
}
let doubleArray = array.lazy.map(increment).map(double)
print(doubleArray[3])
```

And this will only compute `double(increment(array[3]))` when this entry is accessed entry, and only for this one!

Whereas using `array.map(increment).map(double)[3]` instead (without `lazy`) would have computed all the output values of the whole `array` sequence, and only once all values have been computed, extract the 4th one. What a was of computational time for all those other 999 values!

## Conclusion

Be lazy[^conclusion].

[^conclusion]: Yes I was too lazy to write a conclusion. But as this article just demonstrated, being lazy makes you a good programmer, right? 😜
