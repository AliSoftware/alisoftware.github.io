---
layout: post
title: "Thinking in Swift, Part 1: Saving ponies"
categories: swift
translations:
  - lang: Chinese
    flag: 🇨🇳
    author: the SwiftGG team
    url: http://swift.gg/2015/09/29/thinking-in-swift-1/
---

I often see newcomers to Swift try to translate their ObjC code into Swift.
But the hardest thing when starting to code in Swift is not the syntax, but changing your way of thinking, to use the new Swift concepts which were unknown to ObjC.  
  
In this article series, we'll take an example of ObjC code and walk thru converting it to Swift, introducing more and more concepts along the way.

> Part 1 of this article talks about: optionals, forced-unwrapped optionals, ponies, `if let`, `guard`, and 🍰.


## The ObjC code

So let's say you want to create a list of items (e.g. to display in a TableView later) — each having an icon, title and url — and initialize it from a JSON. Here's what the ObjC code could look like:

```objc
@interface ListItem : NSObject
@property(strong) UIImage* icon;
@property(strong) NSString* title;
@property(strong) NSURL* url;
@end

@implementation ListItem
+(NSArray*)listItemsFromJSONData:(NSData*)jsonData {
    NSArray* itemsDescriptors = [NSJSONSerialization JSONObjectWithData:jsonData options:0 error:nil];
    
    NSMutableArray* items = [NSMutableArray new];
    for (NSDictionary* itemDesc in itemsDescriptors) {
        ListItem* item = [ListItem new];
        item.icon = [UIImage imageNamed:itemDesc[@"icon"]];
        item.title = itemDesc[@"title"];
        item.url = [NSURL URLWithString:itemDesc[@"url"]];
        [items addObject:item];
    }
    return [items copy];
}
@end
```

Ok, pretty standard ObjC code here.

## A direct translation into Swift

Now imagine how any Swift new-comer would basically translate that code into Swift:

```swift
class ListItem {
    var icon: UIImage?
    var title: String = ""
    var url: NSURL!
    
    static func listItemsFromJSONData(jsonData: NSData?) -> NSArray {
        let jsonItems: NSArray = try! NSJSONSerialization.JSONObjectWithData(jsonData!, options: []) as! NSArray
        let items: NSMutableArray = NSMutableArray()
        for itemDesc in jsonItems {
            let item: ListItem = ListItem()
            item.icon = UIImage(named: itemDesc["icon"] as! String)
            item.title = itemDesc["title"] as! String
            item.url = NSURL(string: itemDesc["url"] as! String)!
            items.addObject(item)
        }
        return items.copy() as! NSArray
    }
}
```

Someone with a little Swift experience should already see a lot of code smell here. And experienced Swift users are probably all dead from a heart attack reading this code already.

## What can go wrong?

The first thing that looks like code smell in the above example is a very bad habit that a newcomer to Swift often fall into: using implicitly-unwrapped optionals (`value!`), force-casts (`value as! String`) and force-try (`try!`) everywhere.

**Optionals are your friends**: they are good because they force you to think about these cases when your values are `nil`, and think about what you should do in such scenarios. Like "what should I display if I don't have any icon? Should I use a placeholder in my TableViewCell? Or use a totally different cell template?".

These are use cases that we often forget to take into account when writing our ObjC code, but Swift help us not forget about them, so **it would be a shame to throw that advantage away by force-unwrapping them and make your code crash** when they were actually `nil`.

> You should **never** force-unwrap a value, except when you really know what you're doing. Keep in mind that every time you add a `!` just to please the compiler, you're killing a pony 🐴. 

_Sadly, that mistake is encouraged by Xcode, because when the error says: "value of optional type 'NSArray?' not unwrapped. Did you mean to use `!` or `?`?", the fix-it suggests that… you add a `!` at the end 🙀. Oh, Xcode, what a newbie you are._

## Let's save those ponies

So how do we avoid all those bad `!` everywhere? Here are some techniques:

* Use optional binding `if let x = optional { /* use x */ }`
* Use `as?` instead of `as!`, which returns  `nil` if the cast fails; you can use it in combination with `if let` of course.
* You can also use `try?` instead of `try!`, which will return `nil` if the expression failed [^try]

[^try]: Note that `try?` silently discards the error: when using it you can't know anymore _why_ the code failed. So generally it's better to use `do { try … } catch { }` instead of `try?` when possible. But in our case, as we want to return an empty array if the JSON serialization failed for whatever reason, `try?` is a ok here.

So, let's see what our code will become after using those rules[^force-cast-copy]:

```swift
class ListItem {
    var icon: UIImage?
    var title: String = ""
    var url: NSURL!
    
    static func listItemsFromJSONData(jsonData: NSData?) -> NSArray {
        if let nonNilJsonData = jsonData {
            if let jsonItems: NSArray = (try? NSJSONSerialization.JSONObjectWithData(nonNilJsonData, options: [])) as? NSArray {
                let items: NSMutableArray = NSMutableArray()
                for itemDesc in jsonItems {
                    let item: ListItem = ListItem()
                    if let icon = itemDesc["icon"] as? String {
                        item.icon = UIImage(named: icon)
                    }
                    if let title = itemDesc["title"] as? String {
                        item.title = title
                    }
                    if let urlString = itemDesc["url"] as? String {
                        if let url = NSURL(string: urlString) {
                            item.url = url
                        }
                    }
                    items.addObject(item)
                }
                return items.copy() as! NSArray
            }
        }
        return [] // In case something failed above
    }
}
```

[^force-cast-copy]: As you can see, I kept one `as!` in that code in the end (`items.copy() as! NSArray`). Sometimes it's ok to ~~kill a pony~~ force-cast, if you _really, really_ know that the returned type _can't_ be anything else, like here with `mutableArray.copy()`. But those exceptions are rare and only acceptable if you really thought about the use case first (beware, if that 🐴 dies, it will be forever on your conscience).

## The pyramid of doom

Sadly, adding those `if let` everywhere shifted our code a lot to the right, leading to the infamous [pyramid of doom](https://en.wikipedia.org/wiki/Pyramid_of_doom_(programming)) (_insert dramatic music here_).

There are some mechanisms that can help us reduce that in Swift:

* combining multiple `if let` statements into one: `if let x = opt1, y = opt2`
* using the `guard` statement, which allow us to bail early from a function if a condition is not met, avoiding to shift the rest of the function body.

Let's also use that code cleaning iteration to remove the variables types when they can be infered — like simply use `let items = NSMutableArray()` — and take advantage of that `guard` statement to also ensure that our json is really an array of `NSDictionary` objects. Finally, let's use a Swift-er return type `[ListItem]` instead of that ObjC `NSArray`:

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
                // If we failed to unserialize the JSON
                // or that JSON wasn't an Array of NSDictionaries,
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

The `guard` statement is really nice because it concentrate the code checking if the input is valid at the top of the function, and in the rest of the code you don't have to bother with those checks anymore. If the input was not as expected, we bail early, which help us concentrate on the good path, where things are as expected.

## Wasn't Swift supposed to be more compact than ObjC?

<center>![the cake is a lie](/assets/the-cake-is-a-lie.png)</center>

Well ok, yes the code seems more complex than the ObjC counterpart. But don't worry, we'll make it way more compact in the upcoming part 2 of this article.

But most importantly, this code is **much safer than its ObjC counterpart**. In fact, the ObjC code was shorter only because we forgot to perform a lot of safety tests. Even if it seemed pretty common, **our ObjC code would have crashed right away in a number of cases**, like if we gave it an invalid JSON, or one that wasn't structured as an array of dictionary of strings (like if the one creating the JSON thought the "icon" key was just a Boolean telling if the item had an icon or not, instead of a `String`…). **We simply forgot to handle all those use cases in ObjC**, because ObjC didn't help use think about those cases whereas Swift forces us to consider them.

So of course ObjC was shorter: that's because we simply forgot to handle all those stuff! It's easy to be shorter if you don't protect yourself from crashes. Sure it's easier to drive without worring about what obstacles can block the road. But that's also how you kill ponies.


## Conclusion

Swift is designed to be safer. Don't disregard optionals by force-unwrapping all the things: when you see a `!` in your Swift code, you should always think it's probably a code smell and something could go wrong.

In the upcoding part 2 of this article, we'll see how to make this Swift code thinner and continue thinking in Swift by migrate away from `for` loops and `if-let` using `map` and `flatMap` instead.

In the meantime, drive safe and, yes, save ponies! 🐴
