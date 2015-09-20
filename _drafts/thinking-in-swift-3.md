---
layout: post
title: "Thinking in Swift, Part 3: Struct vs. Class"
categories: swift
---

## Struct vs Class

One other mistake that our newcomer to Swift did in the above code, is to start with a `class`. That's understandable, because in ObjC we use classes everywhere. 

I won't start a big post here about using `struct` and value types vs `class` and reference types: I rather strongly suggest that you [watch this excellent talk from Andy Matuschak on that subject](https://realm.io/news/andy-matuschak-controlling-complexity/). In short, in our use case, a `struct` seems more appropriate because it carries value, and is not intended to be mutated (and rather copied than referenced).

Also, the advantage of migrating to a `struct` here is that they have an implicit constructor by default if you don't define any: so we can easily build a `ListItem` using its default constructor `ListItem(icon: …, title: …, url: …)`.

Lastly, as we now can't create a bogus `ListItem` because we eliminated the problem of data corruption above, we can eliminate the default value `""` for `title`, and to save that last pony by transforming `NSURL!` into `NSURL`[^about-time].

[^about-time]: That `NSURL!` was bogging me for some time now, and was still there only because I was too lazy creating a proper `init` method for our `ListItem` class, and didn't want to clobber the sample code before by dealing with it as I knew we'd get rid of it eventually. It's about time we saved that last pony!

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

In the above example, I also used a new trick, using the `??` operator to give a default value in case `iconName` is `nil`.

This `??` operator a little similar to ObjC's `opt ?: val` expression, for those who know it: `opt ?? val` will return the value of `opt` if it's non-`nil`, and `val` if `opt` was `nil`. That means that if `opt` is of type `T?`, `val` has to be of type `T`, and so will the result be. 

So here `iconName ?? ""` allows us to use an empty `""` image name in case `iconName` is `nil`, which we know will lead to a `nil` `UIImage` (and `icon = nil`) in such case.

⚠️ That's **NOT** the best way to achieve this, but let's keep some stuff for the next part of this article series 😉 _(Spoiler: it involves `flatMap` too)_
