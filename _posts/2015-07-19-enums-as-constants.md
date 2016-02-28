---
layout: post
title: Enums as constants
excerpt: Use enums as constants for your images, colors, storyboards, and more… and make your code beautiful (and short, and with autocompletion, and error-free)
categories: swift enum constants
---

Swift enums are way more powerful than C/ObjC enums. For example, they are not limited to be mapped to Ints like in C but can also be e.g. Strings. And they also can contain methods inside them, quite like classes and struct.

That new feature opens a broad new usage for them, especially using them as specialized constants that produce types which were not possible to use as constants before. Let's see some interesting use for that.


> TL;DR: [here is a nice tool to generate enums for all your constants](https://github.com/AliSoftware/SwiftGen), provided with a Playground to see the below examples in action.

<!--more-->

## Enums for your Image Assets

When you use the images of your Assets Catalog, you're generally writing stuff like `UIImage(named: "FooBar")` to get the image of that name. The problem with that is that you need to remember exactly the name you gave to the image asset (did you name it `"foobar"`, `"FooBar"`, `"foo_bar"` or `"foo-bar"` in your assets catalog?), you are prone to typos, and you don't have any kind of autocompletion.

Or course, you could declare a `let FooBarImageName = "FooBar"` constant somewhere in your code to avoid those problems, but having constants in the global space is not really cool as they are mixed with every other stuff, and are not clearly grouped with other constants representing image names. You could also group them in a static `struct` in the global namespace, but that's still not very elegant.

So here comes the Swift `enum`. As Swift can have `enums` with a raw type `String`, you can write stuff like this:

```swift
enum ImageAsset : String {
  GreenApple = "green-apple"
  Banana = "banana"
  Pear = "pear"
  Strawberry = "1-strawberry"
  Strawberries = "strawberry-basket"
}
```

### Adding constructors

That's nice, but we can go further by adding a method (or better, a computed variable) that convert this enum into an `UIImage`.

```swift
extension ImageAsset {
  var image : UIImage {
    return UIImage(named: self.rawValue)!
  }
}
```

And here you go, now `ImageAsset.GreenApple.image` will return your `UIImage`. No risk of any typo at the call site here, free autocompletion, everything looks nice!

But hey, why not also make an `UIImage` constructor as well?

```swift
extension UIImage {
  convenience init(asset: ImageAsset) {
    self.init(named: asset.rawValue)!
  }
}
```

And now you can go use `UIImage(asset: .GreenApple)` too!

_Given that this enum is actually so linked with `UIImage`, I like to declare it inside an `UIImage` extension instead of declaring it at the global scope._

Pretty cool, right?

## Enums for your UIColors

But wait, why stop at `UIImage`? There is a lot of other stuff where you could apply that pattern too. For example, you typically have custom `UIColors` which you use everywhere in your code and that are tailored to your client's corporate identity.

### An `enum` of type `UIColor`?!

But how could we associate a raw type of `UIColor` to an enum? You can create `enum` of type `Int` or `String`, but using `UIColor` as the `enum`'s `RawType` is not possible.

That's not really a problem, because we know other ways to represent colors: using their RGBA hex value (like `ffcc00ff`)! And your graphic designer actually probably gave you those color references in that `#ffcc00` format anyway (as it's also the one used in CSS), so that's also nice to have a direct correspondance here.

So let's use that instead, representing colors using some `UInt32` `enum`, and take advantage of the `0x` notation to affect each value using their hexadecimal representation.  
We'll also add it to the `UIColor` class as an `extension`, to scope it in that namespace instead of putting that `enum` the global scope:

```swift
extension UIColor {
  enum ColorName : UInt32 {
    case Translucent = 0xffffffcc
    case ArticleBody = 0x339666ff
    case Cyan = 0xff66ccff
    case ArticleTitle = 0x33fe66ff
  }
}
```

Ok, right, but now how do we build an `UIColor` instance from that?

We will simply need an initializer that use that `UInt32`, do some bitmask arithmetics on it to split it in 4 `UInt8` components, and build an `UIColor` with that. Then, we will be able to write a `convenience init` to build a color from that `enum`:

```swift
extension UIColor {
  convenience init(named name: Name) {
    let rgbaValue = name.rawValue
    let red   = CGFloat((rgbaValue >> 24) & 0xff) / 255.0
    let green = CGFloat((rgbaValue >> 16) & 0xff) / 255.0
    let blue  = CGFloat((rgbaValue >>  8) & 0xff) / 255.0
    let alpha = CGFloat((rgbaValue      ) & 0xff) / 255.0
    
    self.init(red: red, green: green, blue: blue, alpha: alpha)
  }
}

UIColor(named: .ArticleBody)
```

That looks pretty handy!


## Use `enum` constants everywhere!

There are plenty of other types for which `enum` as constants can be applied. Let's look at two more example of what your code could look once you've implemented them for Strings and Storyboards _(implementation for those `enum`s are left as an exercice to the reader… or you can go check my Xcode Playground linked below)_.

### Localizable.strings

You can use the same pattern for your `Localizable.strings` strings, for example, like creating the appropriate `enum` and adding a `convenience init` in a `String` extension, so that you could write `String(key: .Greetings)` instead of `NSLocalizedString("Greetings", comment: "")`. That's pretty similar to how we implemented the `UIImage(named: .SomeAsset)` above.

_You could go way further with that, by using the concept of "Associated Values" to pass arguments to your strings that use parameters (but that will be the subject of a dedicated article)_

### UIStoryboard

You can also use it for creating your `UIStoryboard` instances and their scenes / `UIViewControllers` (avoiding to use global-scope constants for the scene's `storyboardIDs`/`identifiers` like a lot of us used to do in ObjC).

Wouldn't it be cool to be able to write stuff like that?

```swift
let wizzardVC = UIStoryboard.Wizzard.initialViewController()
let msgCompVC = UIStoryboard.Message.Composer.viewController()
let tutorialVC = UIStoryboard.Tutorial.viewController(.QuickStart)
```

Well you got it, using `enum` of course it is possible!  
_One simple difference here is that we have an additional level of `enum`s (There are actually multiple enums like `Message` and `Composer`, each having a `case` entry for each of their scene)_

## Automate all the things!

All that is pretty cool, but would need you to write a lot of `enums` and keep them all in sync when you create a new `UIStoryboard`, a new asset in your ImageCatalog, or add an entry in your `Localizable.strings`.

That's where you think automation, of course.

I created a suite of tools called ["SwiftGen", available here in my GitHub](https://github.com/AliSoftware/SwiftGen#uicolor), that are dedicated to this task.

These tools can automatically generate the code to create `enums` and the `convenience init` constructors for:

* All your assets in your `.xcassets` Asset Catalog
* All your `UIStoryboard` and their Scenes
* All your strings in `Localizable.strings`
* Even a tool to create the `UIColor` enum from a text file declaring them

### Usage

You can use those tool to generate the code for your `enum`s and create the appropriate `.swift` files, add those generated files in your Xcode project, and re-run the tools every time it's needed when you added an Asset or Storyboard Scene.

These tools are written in Swift 2, and once compiled (using the provided `Rakefile`) into binary executables, are really fast to execute — even when they are parsing a big `Localizable.strings` file or a whole directory to search for `.imageset` asset files — so it shouldn't even be a problem to make them execute as part of an _"Script Build Phase"_ early in your build, to ensure that the code is always up-to-date each time you build your code.

## Playground

My [SwiftGen](https://github.com/AliSoftware/SwiftGen) repository comes with an Xcode Playground so you can play with those "`enum` as constants" concepts and see them in action.

Enjoy!
