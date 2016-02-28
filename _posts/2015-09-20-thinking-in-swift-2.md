---
layout: post
title: "Thinking in Swift, Part 2: map those arrays"
categories: swift
translations:
  - lang: Chinese
    flag: 🇨🇳
    author: the SwiftGG team
    url: http://swift.gg/2015/10/09/thinking-in-swift-2/
---

In [part 1 of this article series](/swift/2015/09/06/thinking-in-swift-1), we saw how to avoid force-unwrapping optionals, save ponies 🐴 and avoid crashes by doing so. In this part 2, we'll refine our code to make it Swift-er, introducing `map()` and `flatMap()`.

> Today's article will talk about `map` and `flatMap` on `Arrays`.

## Previously, in Thinking in Swift[^tvshow-intro]

[^tvshow-intro]: Insert opening credits of some bad*ss TV Show here.

As a reminder, here is the code as we left it [the last time](/swift/2015/09/06/thinking-in-swift-1/):

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

The aim is to adopt more and more Swift-er patterns and syntax to make that code better and thinner.

## Introducing map()

`map()` is a method on `Array` that takes a function as a parameter, which explains how to *transform* each item of the array into a new one. This allows to transform an array `[X]` into `[Y]` by just explaining how to transform `X -> Y`, without the need to create a temporary mutable array.

So in our case, instead of looping with a `for` as we did before, we could apply `map` to `jsonItems` — our JSON array of `NSDictionary` — and provide a transform to convert each `NSDictionary` into a `ListItem` instance:

```swift
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
```

It may seem like a simple change but it allows us to concentrate on the question of "how to convert one `NSDictionary` into a `ListItem`" — which is the heart of our problem after all — and more importantly avoid the need to create a intermediate mutable array like we had to in ObjC. Always avoid mutable state when possible.

## Data corruption

One problem with the code we used so far is that we still create a `ListItem` (and include it in the final array) even if we have incorrect input data. So if some of those `NSDictionary` entries are invalid, we end up corrupting our output array too by still having those empty `ListItem()` objects in it that don't really mean anything.

More importantly, we are still killing some ponies 🐴 as we are still using `NSURL!` and our code path still allows us to create `ListItem` instances that don't have an `NSURL` (`item.url` not affected if we don't have a valid `"url"` key) and that would crash our code if we try to access such an invalid `NSURL!`.

To solve this, we can instead make our transform return a `nil` `ListItem` if the input is invalid, which seems more appropriate than a corrupted/empty `ListItem`.

```swift
return jsonItems.map { (itemDesc: NSDictionary) -> ListItem? in
    guard …/* condition for valid data */… else { return nil }
    let realValidItem = ListItem()
    … /* fill the ListItem with the values */
    return realValidItem
}
```

But if we were to use that new `NSDictionary -> ListItem?` transform with `jsonItems.map`, it would generate a `[ListItem?]`, containing `nil` items in places where we have invalid `NSDictionary` entries in the input. Better than before, but still not very practical to deal with.

## Using flatMap()

That's where `flatMap()` comes to the rescue.

`flatMap()` is similar to `map()` but uses a `T->U?` transform instead of a `T->U` transform, and then doesn't add the items to the output array if that `transform` returns `nil`[^other-signatures].

Semantically, you can see this `flatMap` as if you were applying `map`, then "flatten" the result (hence the method's name) to remove the `nil` values from the output array.

[^other-signatures]: There are plenty of other signatures for `flatMap`, like one transforming an array of arrays of `T` into a flat array of `T` (`[[T]] -> [T]`) for example. But today we'll only focus on the method on `Array` converting a `[T]` into a `[U]` using a `T->U?` transform.

Applying this to our example gives us the following code:

```swift
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
```

Now we only return a real `ListItem` if all the keys are present[^optional-icon] and valid (including that `NSURL` that we ensured would be non-`nil`). Otherwise (`guard` statement), we return `nil` early, telling `flatMap` not to add that invalid element to the returned array.

[^optional-icon]: Note that we made our code still accept a `NSDictionary` without the `"icon"` key, as we decided that it's ok/valid for a `ListItem` not to have any icon. But the other keys are still mandatory.

That's much better and safer, right? And we eliminated the problem of data corruption and the risk of having dummy, invalid `ListItem` elements in our array altogether in case of bad input.


## Conclusion

We still have a lot of work to do, but that will be all for today (let's keep some material for the next parts of this article series!)

So in this episode we learned how to replace a `for` loop with a `map` or `flatMap`, and we secured our code a bit more by avoiding the generation of inconsistent output when our input data is invalid. That's quite a nice improvement already.

In the upcoming episodes, we'll see how converting our `ListItem` as a `struct` could help us, and explore other uses of `map` and `flatMap` — especially on `Optionals`.

In the meantime, take time to discover the power of `map()` and `flatMap()` on arrays. I know they can be scary or complex at first, but once you get it, you'll want to use them everywhere!

![map-everywhere](/assets/map-everywhere.jpg)
{: style="text-align: center"}
