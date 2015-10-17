---
layout: post
title: "Thinking in Swift, Part 4: map all the things!"
categories: swift
---

In [previous articles of this series](/swift/2015/09/20/thinking-in-swift-2/), we learned about using `map` and `flatMap` on arrays. Today we'll discover `map` and `flatMap` also exists on Optionals. And on plenty of other types.

## Array vs. Optional

So, as a reminder, we learned in a previous article that the signature of the `map()` and `flatMap` functions on `Array<T>` are:

```swift
// Method on Array<T>
    map( transform: T ->          U  ) -> Array<U>
flatMap( transform: T ->    Array<U> ) -> Array<U>
```

This means that, given a `transform: T->U` you can transform an array of `T` into an array of `U`. Simply call `map( transform: T->U )` on your `Array<T>` and it will return an `Array<U>`

Well, not surprisingly, the signature of `map()` and `flatMap` on `Optional<T>` are really similar:

```swift
// Method on Optional<T>
    map( transform: T ->          U  ) -> Optional<U>
flatMap( transform: T -> Optional<U> ) -> Optional<U>
```

See the resemblance?

## map() on Optionals

So, what does the `map` method do on the `Optional<T>` type (a.k.a. `T?`)?

Well that's quite simple: similarly to the same method on `Array<T>`, it takes the content of that `Optional<T>`, transform it using the provided `transform: T->U`, and wrap the result in a new `Optional<U>`

When you think about it, this is very similar to what `Array<T>.map` does: it applies a `transform` to each inner value inside the `Array<T>` (resp. `Optional<T>`) and return a new `Array<U>` (resp. `Optional<U>`) wrapping the transformed values.

## Back to our example

So, how can all this be useful in the example code we've been working on?

Well [in the last version of our code](/swift/2015/10/03/thinking-in-swift-3/#converting-our-class-to-a-struct), we had an `itemDesc["icon"]` which was a `String?`, and we wanted to transform it into an `UIImage`; but
`UIImage(named:)` takes a `String` as a parameter, not a `String?`, so we needed to call that method with the _inner_ `String`, and only if the optional indeed had such an inner value (= was not `nil`).

One solution would be to use Optional Binding to do this:

```swift
let icon: UIImage?
if let iconName = itemDesc["icon"] as? String {
  icon = UIImage(named: iconName)
} else {
  icon = nil
}
```

But that's a lot of lines for such a simple operation.

In the previous example, we used a (quite dirty) alternative solution, using the `nil`-coalescing operator `??`:

```swift
let iconName = itemDesc["icon"] as? String
let icon = UIImage(named: iconName ?? "")
```

That worked but only because if the `iconName` is `nil`, we will trick the initialized of `UIImage` and use `UIImage(named: "")` which will indeed return a `nil` image. But that doesn't feel clean at all, we're kinda bending the initializer to make it work here.

## Let's use map

So why not use `map`? Indeed, we want to unwrap our `Optional<String>` if it's not `nil`, transform that inner value into an `UIImage` and return that `UIImage`, so isn't it exactly an appropriate use case?

Let's try it:

```swift
let iconName = itemDesc["icon"] as? String
item.icon = iconName.map { imageName in UIImage(named: imageName) }
```

Wait… that above code doesn't compile. Could you guess why?

## What's wrong?

The problem with the above code is that `UIImage(named: …)` returns an optional too: if there is no image with the `name` provided, it can't create an `UIImage`, so that makes total sense for this initializer to be _failable_ and return `nil` in such cases.

So the problem here is that the closure we gave to `map` takes a `String` and returns… an `UIImage?` — as the initializer is _failable_ and can return `nil`. And if you check the signature of `map` again, it expects a closure of type `T->U`, and return an `U?`. So in our case, `U` stands for `UIImage?` and so the whole `map` expression will return `U?` which is… an `UIImage??`… Yes, that's a double-optional, oh boy!

## flatMap() to the rescue

`flatMap` is like `map`, but takes a `T->U?` transform (instead of a `T->U` one), and it "flattens" (hence the name) the result to only have one level of optional. This is exactly what we want!

```swift
let iconName = itemDesc["icon"] as? String
item.icon = iconName.flatMap { imageName in UIImage(named: imageName) }
```

So here's what `flatMap` does in practice:

* If `iconName` is `nil`, it returns `nil` directly (but typed as `UIImage?`)
* If `iconName` isn't `nil`, it applies the `transform` on the inner value, thus trying to build an `UIImage` using that `String`, and return the result — which is already a `UIImage?` and thus which can be `nil` if the `UIImage` initializer failed.

In short, `item.icon` will only have a non-`nil` value if `itemDesc["icon"] as? String` is non-`nil` AND `UIImage(named: imageName)` succeeded.

This feels much better and more idiomatic than tricking the initializer with a `??`.

## Use init as a closure

To go a bit further, the code above could also be written in a more compact way, as Xcode 7 now exposes the constructors of a type through the type's `.init` property.

This means that `UIImage.init` is actually already a function that takes a `String` and return an `UIImage?`. So we can use it directly as the parameter to our `flatMap` call, no need to wrap it in a closure!

```swift
let iconName = itemDesc["icon"] as? String
item.icon = iconName.flatMap(UIImage.init)
```

Wow! so much magic!

<center>![magic](/assets/magic.gif)</center>

Ok, that being said:

* I find this harder to read and personally prefer using an explicit closure here, to make the code more clear and less ambiguous. But that's just a matter of personal preference, and that's always good to know that's possible!
* As pointed out in the comments, this actually compile but doesn't seem to work as expected — I'm guessing the Swift compiler maps `UIImage.init` to `{ UIImage(contentsOfFile:$0) }` instead of using the expected `{ UIImage(named:$0) }`. One more reason to prefer explicit closures there.

## Our final Swift code

So here is this last lesson applied to our previous code.

```swift
struct ListItem {
    let icon: UIImage?
    let title: String
    let url: NSURL
    
    static func listItemsFromJSONData(jsonData: NSData?) -> [ListItem] {
        guard let jsonData = jsonData,
            let json = try? NSJSONSerialization.JSONObjectWithData(jsonData, options: []),
            let jsonItems = json as? Array<NSDictionary> else { return [] }
        
        return jsonItems.flatMap { (itemDesc: NSDictionary) -> ListItem? in
            guard let title = itemDesc["title"] as? String,
                let urlString = itemDesc["url"] as? String,
                let url = NSURL(string: urlString)
                else { return nil }
            let iconName = itemDesc["icon"] as? String
            let icon = iconName.flatMap { UIImage(named: $0) }
            return ListItem(icon: icon, title: title, url: url)
        }
    }
}
```

## A look back at our ObjC code

Take a little time to compare the final above Swift code with [the original ObjC code](http://alisoftware.github.io/swift/2015/09/06/thinking-in-swift-1/#the-objc-code) we started with. We've adapted quite a lot of stuff from there!

If you look closely at those two ObjC vs. Swift codes, you'll realize that the Swift code is not significantly smaller (5+15 LoC[^loc] for Objc vs. 19 LoC[^loc] for Swift), **but it's way safer**.

Especially, in that Swift code we learned to use `guard`, `try?` and `as?` that forces us to check that everything is of the expected type, where the ObjC code didn't bother and would have crashed 💣💥. So maybe it's quite the same size, but the ObjC code is way more dangerous!

[^loc]: Lines of Codes

## Conclusion

With this article series, I hope you realize that you shouldn't try to translate your ObjC code to Swift directly. Instead, try re-thinking your code, and reimagining it. It's often better to start from a clean state and rewrite your code with the Swift idioms in mind than to try to translate ObjC code directly.

I'm not saying that it's easy. Changing your way of thinking when you're already used to code in ObjC and used to its patterns and ways to write code may take some time. But it's definitely for the better.

---

That concludes the last part of the Thinking in Swift series[^epilogue]. It's time for you to go nuts on new Swift projects, and to fully commit to thinking in Swift!

[^epilogue]: I'll just post an upcomming epilogue very soon to drop a word about _Monads_ and really conclude all this series. But don't worry, there are still a lot articles on Swift to come after that!

Happy Swifting, and…  
<center>![map-everywhere](/assets/map-all-the-things.jpg)</center>
