---
layout: post
title: StringInterpolation in Swift 5 — AttributedStrings
date: 2018-12-16
categories: swift
swift_version: 5.0
---

In [the previous post](/swift/2018/12/15/swift5-stringinterpolation-part1/), we introduced the new StringInterpolation design coming to Swift 5. In this second part, I'll focus on one application of that new `ExpressibleByStringInterpolation`, to make `NSAttributedString` prettier.

## The goal

One of the first application I thought about when seeing that [new StringInterpolation design in Swift 5](https://github.com/apple/swift-evolution/blob/master/proposals/0228-fix-expressiblebystringinterpolation.md) was to make it easy to build an `NSAttributedString`.

My goal was to be able to create an attributed string using a syntax like this[^1]:

```swift
let username = "AliGator"
let str: AttrString = """
  Hello \(username, .color(.red)), isn't this \("cool", .color(.blue), .oblique, .underline(.purple, .single))?

  \(wrap: """
    \(" Merry Xmas! ", .font(.systemFont(ofSize: 36)), .color(.red), .bgColor(.yellow))
    \(image: #imageLiteral(resourceName: "santa.jpg"), scale: 0.2)
    """, .alignment(.center))

  Go there to \("learn more about String Interpolation", .link("https://github.com/apple/swift-evolution/blob/master/proposals/0228-fix-expressiblebystringinterpolation.md"), .underline(.blue, .single))!
  """
```

[^1]: For the code in that post & playground, you'll need to use Swift 5. At the time of this writing, the latest Xcode is 10.1 with Swift 4.2, so if you want to try that code you'll need to [download the Swift 5 development snapshot by following the official instructions here](https://swift.org/download/#snapshot). It's an easy way to install the Swift 5 toolchain which you can then activate in your Xcode preferences (see official instructions).

That big String uses the multi-line string literals syntax ([new in Swift 4.0, in case you missed it](https://github.com/apple/swift-evolution/blob/master/proposals/0168-multi-line-string-literals.md)) — and even goes as far as wrapping another multi-line String literal inside another (see the `\(wrap: …)` segment)! — and contains interpolations to add some styling to parts of that big String… so a lot of new features of Swift coming together!

The result `NSAttributedString`, once rendered in a `UILabel` or `NSTextView`, should then look like this:

![rendering of the AttributedString created by the code above](/assets/StringInterpolation-AttrString.png)  
{: style="text-align: center"}

☝️ Yes, that above with the text and image… is really **just** an `NSAttributedString` (and not a complex view with layout or anything)! 🤯

## First implementation

So how are we going to implement this? Well, similar to how we implemented `GitHubComment` [in part 1](/swift/2018/12/15/swift5-stringinterpolation-part1/) of course!

So, let's start with the declaration of the dedicated type first, before even tackling string interpolation:

```swift
struct AttrString {
  let attributedString: NSAttributedString
}

extension AttrString: ExpressibleByStringLiteral {
  init(stringLiteral: String) {
    self.attributedString = NSAttributedString(string: stringLiteral)
  }
}

extension AttrString: CustomStringConvertible {
  var description: String {
    return String(describing: self.attributedString)
  }
}
```

Simple enough, right? That's just a wrapper around `NSAttributedString`.
Now, let's add support for `ExpressibleByStringInterpolation` that would allow both literals but also strings annotated with `NSAttributedString` attributes:

```swift
extension AttrString: ExpressibleByStringInterpolation {
  init(stringInterpolation: StringInterpolation) {
    self.attributedString = NSAttributedString(attributedString: stringInterpolation.attributedString)
  }

  struct StringInterpolation: StringInterpolationProtocol {
    var attributedString: NSMutableAttributedString

    init(literalCapacity: Int, interpolationCount: Int) {
      self.attributedString = NSMutableAttributedString()
    }

    func appendLiteral(_ literal: String) {
      let astr = NSAttributedString(string: literal)
      self.attributedString.append(astr)
    }

    func appendInterpolation(_ string: String, attributes: [NSAttributedString.Key: Any]) {
      let astr = NSAttributedString(string: string, attributes: attributes)
      self.attributedString.append(astr)
    }
  }
}
```

At that stage, we're already able to use it that way to easily build an `NSAttributedString`:

```swift
let user = "AliSoftware"
let str: AttrString = """
  Hello \(user, attributes: [.foregroundColor: NSColor.blue])!
  """
```

That's already nice as it is, right?

## Adding styling convenience

But dealing with attributes as a dictionary `[NAttributedString.Key: Any]` isn't really nice. Especially since that `Any` isn't typed, and forces us to know the expected type of the value for each key…

So let's make that nicer by creating a dedicated `Style` type[^2] to help us building attributes dictionaries:

```swift
extension AttrString {
  struct Style {
    let attributes: [NSAttributedString.Key: Any]
    static func font(_ font: NSFont) -> Style {
      return Style(attributes: [.font: font])
    }
    static func color(_ color: NSColor) -> Style {
      return Style(attributes: [.foregroundColor: color])
    }
    static func bgColor(_ color: NSColor) -> Style {
      return Style(attributes: [.backgroundColor: color])
    }
    static func link(_ link: String) -> Style {
      return .link(URL(string: link)!)
    }
    static func link(_ link: URL) -> Style {
      return Style(attributes: [.link: link])
    }
    static let oblique = Style(attributes: [.obliqueness: 0.1])
    static func underline(_ color: NSColor, _ style: NSUnderlineStyle) -> Style {
      return Style(attributes: [
        .underlineColor: color,
        .underlineStyle: style.rawValue
      ])
    }
    static func alignment(_ alignment: NSTextAlignment) -> Style {
      let ps = NSMutableParagraphStyle()
      ps.alignment = alignment
      return Style(attributes: [.paragraphStyle: ps])
    }
  }
}
```

This allows us to use `Style.color(.blue)` to create a `Style` wrapping `[.foregroundColor: NSColor.blue]` easily.

[^2]: Of course I've only implemented a limited list of styles there, for demo purposes. The idea would be to extend that `Style` type to support way more styles in the future, and ideally cover all possible `NSAttributedString.Key` that exists.

But let's not stop there then, and make our `StringInterpolation` handle such `Style` attributes now!

So the idea is to be able to write this:

```swift
let str: AttrString = """
  Hello \(user, .color(.blue)), how do you like this?
  """
```

Wouldn't it be nice? Well, let's just implement the right `appendInterpolation` for that!

```swift
extension AttrString.StringInterpolation {
  func appendInterpolation(_ string: String, _ style: AttrString.Style) {
    let astr = NSAttributedString(string: string, attributes: style.attributes)
    self.attributedString.append(astr)
  }
```

And here you have it! But… this only supports one `Style` at a time then. Why not allow passing multiple `Style` as parameters there? And to do so, instead of allowing a `[Style]` parameter, forcing us to wrap the list of styles in brackets at call site… why not use variadic parameters here?

So instead of the previous implementation, let's instead implement it that way:

```swift
extension AttrString.StringInterpolation {
  func appendInterpolation(_ string: String, _ style: AttrString.Style...) {
    var attrs: [NSAttributedString.Key: Any] = [:]
    style.forEach { attrs.merge($0.attributes, uniquingKeysWith: {$1}) }
    let astr = NSAttributedString(string: string, attributes: attrs)
    self.attributedString.append(astr)
  }
}
```

And now we can mix multiple styles!

```swift
let str: AttrString = """
  Hello \(user, .color(.blue), .underline(.red, .single)), how do you like this?
  """
```

## Supporting Images

Another capability of `NSAttributedString` is to add images as part of the string, by using `NSAttributedString(attachment: NSTextAttachment)`. To do that, it's just a matter of implementing `appendInterpolation(image: NSImage)` to use it.

For that feature, I wanted to add the ability to rescale the image as well. And because I tried all that in a macOS playground, which has its graphic context flipped, I also had to draw the image flipped. (Note that this detail might be different for the iOS implementation supporting `UIImage`). So here's what I came up with:

```swift
extension AttrString.StringInterpolation {
  func appendInterpolation(image: NSImage, scale: CGFloat = 1.0) {
    let attachment = NSTextAttachment()
    let size = NSSize(
      width: image.size.width * scale,
      height: image.size.height * scale
    )
    attachment.image = NSImage(size: size, flipped: false, drawingHandler: { (rect: NSRect) -> Bool in
      NSGraphicsContext.current?.cgContext.translateBy(x: 0, y: size.height)
      NSGraphicsContext.current?.cgContext.scaleBy(x: 1, y: -1)
      image.draw(in: rect)
      return true
    })
    self.attributedString.append(NSAttributedString(attachment: attachment))
  }
}
```

## Wrapping styles in one another

Finally, sometimes, you want to apply a style to a large section of text, which itself might contain styles inside subsections of that text. Like `"<b>Hello <i>world</i></b>"` in HTML where the whole section is bold but contains sub-parts in oblique.

Our API doesn't support that yet, so let's add it. The idea there is to be able to apply a list of `Style...` to something that's not just a `String` but already an `AttrString` with already existing attributes.

The implementation will be similar to `appendInterpolation(_ string: String, _ style: Style...)`, but will mutate the `AttrString.attributedString` to _add_ attributes to it, intead of creating a brand new `NSAttributedString` from a plain `String`.

```swift
extension AttrString.StringInterpolation {
 func appendInterpolation(wrap string: AttrString, _ style: AttrString.Style...) {
    var attrs: [NSAttributedString.Key: Any] = [:]
    style.forEach { attrs.merge($0.attributes, uniquingKeysWith: {$1}) }
    let mas = NSMutableAttributedString(attributedString: string.attributedString)
    let fullRange = NSRange(mas.string.startIndex..<mas.string.endIndex, in: mas.string)
    mas.addAttributes(attrs, range: fullRange)
    self.attributedString.append(mas)
  }
}
```

And with all that, we have achieved our goal and are finally able to create an AttributedString using this single string with interpolations:

```swift
let username = "AliGator"
let str: AttrString = """
  Hello \(username, .color(.red)), isn't this \("cool", .color(.blue), .oblique, .underline(.purple, .single))?

  \(wrap: """
    \(" Merry Xmas! ", .font(.systemFont(ofSize: 36)), .color(.red), .bgColor(.yellow))
    \(image: #imageLiteral(resourceName: "santa.jpg"), scale: 0.2)
    """, .alignment(.center))

  Go there to \("learn more about String Interpolation", .link("https://github.com/apple/swift-evolution/blob/master/proposals/0228-fix-expressiblebystringinterpolation.md"), .underline(.blue, .single))!
  """
```

![rendering of the AttributedString created by the code above](/assets/StringInterpolation-AttrString.png)
{: style="text-align: center"}

## Conclusion

I hope that you enjoyed this series on `StringInterpolation` and that it gave you a glimpse at the power brought by that new design.

You can [download my Playground here](/assets/StringInterpolation.playground.zip)[^1] with the full implementation for `GitHubComment` (see [part 1](/swift/2018/12/15/swift5-stringinterpolation-part1/)), `AttrString`, and even some fun I tried with an simplistic implementation for `RegEx`.

There are plenty more nice ideas around here to make use of that new `ExpressibleByStringInterpolation` API coming in Swift 5 — including some from [Erica Sadun's blog here](https://ericasadun.com/2018/12/12/the-beauty-of-swift-5-string-interpolation/), [here](https://ericasadun.com/2018/12/14/more-fun-with-swift-5-string-interpolation-radix-formatting/) and [here](https://ericasadun.com/2018/12/16/swift-5-interpolation-part-3-dates-and-number-formatters/) — so don't hesitate to read more about it… and even have fun with it yourself!

