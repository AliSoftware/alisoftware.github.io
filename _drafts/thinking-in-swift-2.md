---
layout: post
title: "Thinking in Swift, Part 2: Using map & flatMap"
categories: swift
---

In [part 1 of this article](/swift/2015/09/06/thinking-in-swift-1), we saw how to avoid force-unwrapping optionals, save ponies 🐴 and avoid crashes by doing so. In this part 2, we'll refine our Swift code to make it smaller and Swift-er, using `map` and `flatMap`.

## Previously, in Thinking in Swift[^tvshow-intro]

[^tvshow-intro]: insert opening credits of some bad*ss TV Show here.

If you haven't read [part 1 of this article](/swift/2015/09/06/thinking-in-swift-1), you should start with that first. As a reminder, here is the code as we left it the last time:

```
class ListItem {
    var icon: UIImage?
    var title: String = ""
    var url: NSURL!
    
    static func listItemsFromJSONData(jsonData: NSData?) -> [ListItem] {
        guard let nonNilJsonData = jsonData,
            let json = try? NSJSONSerialization.JSONObjectWithData(nonNilJsonData, options: []),
            let jsonItems = json as? Array<NSDictionary>
            else {
                // If we failed to unserialize the JSON or that JSON wasn't an NSArray,
                // then bail early with an empty array
                return []
        }
        
        var items = [ListItem]()
        for itemDesc in jsonItems {
            let item = ListItem()
            if let icon = itemDesc["icon"] as? String {
                item.icon = UIImage(named: icon)
            }
            if let title = itemDesc["title"] as? String {
                item.title = title
            }
            if let urlString = itemDesc["url"] as? String, let url = NSURL(string: urlString) {
                item.url = url
            }
            items.append(item)
        }
        return items
    }
}
```

## Introducing Map

Another thing that's hard whe migrating from ObjC to Swift is to think Swift, and know those Swift ways that ObjC doesn't offer. For example, instead of creating a temporary `NSMutableArray` and using a `for` loop, Swift has a way better feature to deal with this typical example: `map()`.

`map()` is a method on `Array` that takes a function as a parameter, applies that function on every item of the array, and return a new array with the transformed items.

So here we could start with `jsonItems` — our JSON array of `NSDictionary` — and call `map()` on it, providing a transform to convert each `NSDictionary` into a `ListItem`. So `jsonItems.map { … }` will transform a `[NSDictionary]` into a `[ListItem]`, which is exactly what we want to return as the output of our function:

```swift
class ListItem {
        var icon: UIImage?
        var title: String = ""
        var url: NSURL!
        
        static func listItemsFromJSONData(jsonData: NSData?) -> [ListItem] {
            guard let nonNilJsonData = jsonData,
                let json = try? NSJSONSerialization.JSONObjectWithData(nonNilJsonData, options: []),
                let jsonItems = json as? Array<NSDictionary> else {
                    // If we failed to unserialize the JSON or that JSON wasn't an NSArray, then bail early with an empty array
                    return []
            }
            
            return jsonItems.map { (itemDesc: NSDictionary) -> ListItem in
                let item = ListItem()
                if let icon = itemDesc["icon"] as? String {
                    item.icon = UIImage(named: icon)
                }
                if let title = itemDesc["title"] as? String {
                    item.title = title
                }
                if let urlString = itemDesc["url"] as? String, let url = NSURL(string: urlString) {
                    item.url = url
                }
                return item
            }
        }
    }
```

This change avoid us to create an intermediary, mutable (`var`) array of `ListItems`, and instead concentrate on the question of "how to convert one `NSDictionary` into a `ListItem`, which is the heart of our problem after all.

## Map, optionals & flatMap

`map()` is also a method on optionals. `opt.map(f)` will return `nil` if `opt` is nil, and `f(opt!)` if `opt` is non-nil. So we could convert our optional `String?` that represent the icon name into an optional `UIImage` and avoid our `if let` here: 

```swift
...
    item.icon = (itemDesc["icon"] as? String).map { imageName in
      UIImage(named: imageName)
    }
...
```

The problem there is that `UIImage(named: …)` itself returns an optional. `map`'s signature on an `Optional<T>` is `func map<U>(T -> U) -> U?`, returning an optional if `T` itself is `nil`. But here the closure we provided has a signature of `String -> UIImage?`, so `U` is itself an optional `UIImage?`! That means that the result of this whole line we wrote above is actually of type `UIImage??`, with a double-optional wrapping… oh boy!

For workaround this, we can use `flatMap` instead. `flatMap` is like `map`, but takes a `T -> U?` function that itself returns an optional, and it "flatten" the result to only have one level of optional. This is exactly what we want!

```swift
item.icon = (itemDesc["icon"] as? String).flatMap { imageName in
    UIImage(named: imageName)
}
```

This actually could be written in a more compact way, as Xcode 7 now exposes the constructors of a type through the type's `.init` property, so `UIImage.init` is actually already a function that takes a `String` and return an `UIImage?`… So let's use that directly with our `flatMap` call!

```swift
item.icon = (itemDesc["icon"] as? String).flatMap(UIImage.init)
```

Wow! so much magic!

<center>![magic](/assets/magic.gif)</center>

## Struct vs Class

One last mistake that our newcomer to Swift did in the above code, is to start with a `class`, because in ObjC we use classes everywhere. In our case, because a `ListItem` represent a value (an aggregate of 3 fields, `icon`, `title` & `url`), using a `struct` is more appropriate. I won't start with the rationale on using `struct` and value types over classes and reference types, and rather strongly suggest that you [watch this excellent talk from Andy Matuschak on that subject](https://realm.io/news/andy-matuschak-controlling-complexity/).

Also, `structs` have implicit constructors by default if you don't define any, so we can easily build a `ListItem` using its default constructor `ListItem(icon: …, title: …, url: …)`.

Lastly, because we don't want to generate a default, empty `ListItem` if one of the entries in our JSON is invalid, we'll return a `[ListItem?]` instead of a `[ListItem]`, and output `nil` instead of an empty `ListItem` if that entry in the JSON was bogus.

So let's convert our `class` into a more appropriate `struct` here:

```swift
struct ListItem {
    var icon: UIImage?
    var title: String
    var url: NSURL
    
    static func listItemsFromJSONData(jsonData: NSData?) -> [ListItem?] {
        guard let nonNilJsonData = jsonData,
            let json = try? NSJSONSerialization.JSONObjectWithData(nonNilJsonData, options: []),
            let jsonItems = json as? Array<NSDictionary> else { return [] }
        
        return jsonItems.map { (itemDesc: NSDictionary) -> ListItem? in
            if let title = itemDesc["title"] as? String,
                let urlString = itemDesc["url"] as? String,
                let url = NSURL(string: urlString) {
                    let icon = (itemDesc["icon"] as? String).flatMap(UIImage.init)
                    return ListItem(icon: icon, title: title, url: url)
            }
            return nil
        }
    }
}
```

## Conclusion

That was a lot for today's article! Congrats for those who followed until the end
