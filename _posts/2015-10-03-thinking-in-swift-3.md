---
layout: post
title: "Thinking in Swift, Part 3: Struct vs. Class"
categories: swift
translations:
  - lang: Chinese
    flag: 🇨🇳
    author: the SwiftGG team
    url: http://swift.gg/2015/10/20/thinking-in-swift-3/
---

Continuing my "Thinking in Swift" article series, today we'll do a simple change that will simplify our code again: using a `struct`.

This post is part of an article series. You can read all the parts here: [part 1](/swift/2015/09/06/thinking-in-swift-1/), [part 1 addendum](/swift/2015/09/14/thinking-in-swift-1-addendum/), [part 2](/swift/2015/09/20/thinking-in-swift-2/), [part 3](/swift/2015/10/03/thinking-in-swift-3/), [part 4](/swift/2015/10/11/thinking-in-swift-4/)
{: .note }

## Previously

In [the previous article of this series](/swift/2015/09/20/thinking-in-swift-2/), we learned about using `map` and `flatMap` on arrays, which avoid statefulness in the form of intermediate variables and make us use some functional programming instead[^fp].

[^fp]: Yes, you did some functional programming in the last article… probably even without knowing it already!

As a reminder, here is what the code looked like at the end of our previous work:

```swift
class ListItem {
    var icon: UIImage?
    var title: String = ""
    var url: NSURL!
    
    static func listItemsFromJSONData(jsonData: NSData?) -> [ListItem] {
        guard let nonNilJsonData = jsonData,
            let json = try? NSJSONSerialization.JSONObjectWithData(nonNilJsonData, options: []),
            let jsonItems = json as? Array<NSDictionary>
            else {
                return []
        }
        
        return jsonItems.flatMap { (itemDesc: NSDictionary) -> ListItem? in
            guard let title = itemDesc["title"] as? String,
                let urlString = itemDesc["url"] as? String,
                let url = NSURL(string: urlString)
                else { return nil }
            let li = ListItem()
            if let icon = itemDesc["icon"] as? String {
                li.icon = UIImage(named: icon)
            }
            li.title = title
            li.url = url
            return li
        }
    }
}
```

Today we're just gonna do a really simple change but which will make our code thinner and Swift-er again.

## Struct vs Class

One other mistake that our newcomer to Swift did in the above code, is to start with a `class`. That's understandable, because in ObjC we use classes everywhere. 

There is nothing wrong with `class`. You can still use them with Swift of course. But in Swift, `struct` are way more powerful as their C counterpart: they are no more limited to just a set of fields holding some values.

Instead, Swift's `structs` have quite the same capabilities as classes — except inheritance — but instead are **value-types** (so copied every time you pass them in another variable, much like `Int` for example) whereas classes are **reference-types**, passed by reference instead of being copied, like in Objective-C (and its ugly `*` all over the place)

I won't start a big post here about using `struct` and value types vs `class` and reference types: I rather strongly suggest that you [watch this excellent talk from Andy Matuschak on that subject](https://realm.io/news/andy-matuschak-controlling-complexity/). No need to explain it myself, I won't do better than Andy here!

## Converting our class to a struct

In our case, a `struct` seems more appropriate because it carries values, and is not intended to be mutated (and rather copied than referenced). We're gonna use them as sources for a menu for example, and they are not intended to be modified once created anyway, so this is one case where it makes sense.

Also, the advantage of migrating to a `struct` here is that they have an implicit constructor by default if you don't define any: so we can easily build a `ListItem` using its default constructor `ListItem(icon: …, title: …, url: …)`.

Last but not least, as we now can't create a bogus `ListItem` because we eliminated the problem of data corruption above, we can eliminate the default value `""` for `title`, but more importantly **we can [save that last pony](/swift/2015/09/06/thinking-in-swift-1/)** 🐴 by transforming `NSURL!` into `NSURL`[^about-time].

[^about-time]: That `NSURL!` was bogging me for some time now, and was still there only because I was too lazy creating a proper `init` method for our `ListItem` class, and didn't want to clobber the sample code before by dealing with it as I knew we'd get rid of it eventually. It's about time we saved that last pony!

So here is what it looks like:

```swift
struct ListItem {
    var icon: UIImage?
    var title: String
    var url: NSURL
    
    static func listItemsFromJSONData(jsonData: NSData?) -> [ListItem] {
        guard let nonNilJsonData = jsonData,
            let json = try? NSJSONSerialization.JSONObjectWithData(nonNilJsonData, options: []),
            let jsonItems = json as? Array<NSDictionary> else { return [] }
        
        return jsonItems.flatMap { (itemDesc: NSDictionary) -> ListItem? in
            guard let title = itemDesc["title"] as? String,
                let urlString = itemDesc["url"] as? String,
                let url = NSURL(string: urlString)
                else { return nil }
            let iconName = itemDesc["icon"] as? String
            let icon = UIImage(named: iconName ?? "")
            return ListItem(icon: icon, title: title, url: url)
        }
    }
}
```

Now we only create our `ListItem` instance as the last step, once we have everything, because `structs` provide a default `init` taking all its fields as parameters if you don't provide any `init` yourself. We could have done the same with our `class` version, but with a `class` we'd have had to declare the `init` ourselves.

## Coalescing operator

In the above example, I also used a new trick, using the `??` operator to give a default value in case `iconName` is `nil`.

This `??` operator a little similar to ObjC's `opt ?: val` expression, for those who know it: `opt ?? val` will return the value of `opt` if it's non-`nil`, and `val` if `opt` was `nil`. That means that if `opt` is of type `T?`, `val` has to be of type `T`, and so will the result of the whole expression be. 

So here `iconName ?? ""` allows us to use an empty `""` image name in case `iconName` is `nil`, which we know will lead to a `nil` `UIImage` (and `icon = nil`) in such case.

⚠️ Note ⚠️: That's **NOT** the best and clean way to handle a `nil` `iconName` and have a `nil` `UIImage` as a result. In fact, it even seems a bit ugly and cheaty to use a fake `""` name to have an empty image. But that was an occasion to show you the existance of this `??` operator… and hey, let's also keep some nice stuff for the next part of this article series 😉 _(Spoiler: it involves `flatMap` again)_.

## Conclusion

I'll stop here for today.

We didn't do much in this part 3, just changing a `class` into a `struct`. I didn't even scratch the surface about the difference between the two here (but even if I have been pretty busy lately and didn't post anything on my blog for a while, I didn't want to make you wait too much for a new article).

But we saved that last pony by getting rid of that last `NSURL!` at last 🎉. And you've plenty of stuff to read and learn already by watching [Andy's great talk about «Making Friends with Value Types»](https://realm.io/news/andy-matuschak-controlling-complexity/) until my next part anyway 😃.

I promise I'm not gonna wait that long before posting part 4, which will be about `map` and `flatMap` again, but this time on `Optionals`.
