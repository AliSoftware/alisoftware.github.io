---
layout: post
title: "Thinking in Swift, Part 4: map() & flatMap() on Optionals"
categories: swift
---

## map() on Optionals

The `Optional` type also have a method named `map()`. You can think of `Optional.map` as a method that applies a transform to the inner element inside the optional (if any), similar to it's `Array.map` counterpart. If `opt` is an optional, `opt.map(f)` will return `nil` if `opt` is `nil`, and will apply `f` to the inner value and return `f(opt!)` if `opt` is non-`nil`.

In our example, we could use that to convert our `itemDesc["icon"]` optional `String?` into an optional `UIImage?` using a function that knows how to transform a non-optional `String` into a `UIImage`. This way, we concentrate our work on the function that transforms the non-optional types (`String -> UIImage`), without worrying about the optional itself. We somehow work "inside taht box that is the optional".

Let's try it:

```swift
item.icon = (itemDesc["icon"] as? String).map { imageName in
  UIImage(named: imageName)
}
```

Here, `itemDesc["icon"] as? String` is a `String?` so applying `map` on it allows us to only apply `UIImage(named: imageName)` if this string is non-`nil`, and return `nil` directly if the string is `nil`.

But the above code doesn't compile. Why?

## flatMap() on Optionals

The problem with the above code is that `UIImage(named: …)` returns an optional itself. 

`map`'s signature on an `Optional<T>` is `func map<U>(T -> U) -> U?`. So if we have a `T?` we can transform it into an `U?` by giving a `T->U` transform that works on the non-optional types.

But here the closure we provided above has a signature of `String -> UIImage?`, not `String -> UIImage`, because `UIImage(named: imageName)` can fail and return `nil` (if there is no image with that name in your bundle typically). So in our case, `U` is itself an optional or type `UIImage?`. That means that the result of this whole expression we wrote above is actually of type `UIImage??`, wrapped in a double-optional… oh boy!

To workaround this, we can use `flatMap` instead. `flatMap` is like `map`, but takes a `T -> U?` transform, that itself returns an optional, and it "flattens" (hence the name) the result to only have one level of optional. This is exactly what we want!

```swift
item.icon = (itemDesc["icon"] as? String).flatMap { imageName in
    UIImage(named: imageName)
}
```

So here `item.icon` will only have a non-`nil` value if `itemDesc["icon"] as? String)` is non-`nil` and `UIImage(named: imageName)` also returns a non-`nil` image.

## Init as a method name

The code above could also be written in a more compact way, as Xcode 7 now exposes the constructors of a type through the type's `.init` property.

This means that `UIImage.init` is actually already a function that takes a `String` and return an `UIImage?`. So let's use that directly with our `flatMap` call!

```swift
item.icon = (itemDesc["icon"] as? String).flatMap(UIImage.init)
```

Wow! so much magic!

<center>![magic](/assets/magic.gif)</center>

Ok, that may be a bit too much for one lesson, and may start harder to read (I personally prefer using an explicit closure than `UIImage.init` directly, to make the code more clear, but that's a matter of preference).



