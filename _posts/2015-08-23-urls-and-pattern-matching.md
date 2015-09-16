---
layout: post
title: URLs and Pattern Matching
date: 2015-08-23
categories: swift pattern-matching
translations:
  - lang: Chinese
    flag: 🇨🇳
    author: the SwiftGG team
    url: http://swift.gg/2015/09/15/urls-and-pattern-matching/
---

Today's goal is to parse URLs like `http://mywebsite.org/customers/:cid/orders/:oid` so that we can determine it's a customer's order request and extract the order #`oid` and customer #`cid` from it.  
We're gonna try and do that in an elegant way, using pattern matching and variable binding.

## The idea

The basic idea to solve this problem is to split the URL into path components, then use a `switch` statement and pattern matching to match **each** component wrapped in a tuple. Using this, we'll be able to use variable binding to extract the variable parts of our URL.

We will thus end up with something similar to this:

```swift
// path is a [String] containing the components of the URL's path
// e.g. ["customer","5","order","12"]
switch (path[0], path[1], path[2], path[3]) {
  case ("customer", let cid, "order", let oid):
    print("Customer #\(cid), Order #\(oid)")
  default:
    print("Invalid request")
}
```

## The problem

At the beginning, we'll have an `NSURL` from which we'll extract our path as an **`Array`** (we'll come to that below). But using both pattern matching and variable binding to extract the customer and order IDs won't work with arrays (we can't do `switch path { case ["customer", let cid]: … }`). So we'll need to use tuples (`case ("customer", let cid)`) instead.

But we can't transform an arbitrary array of any size into a tuple — as a tuple is a type on its own defined by the number and type of its inner elements. 
Sure we could create different `switch` statements, matching tuples of different length… but this makes the code quite messy:

```swift
func parse(path: [String]) -> String? {
    switch path.count {
    case 1:
        switch path[0] {
        case "products":  return "List of products"
        case "customers": return "List of customers"
        default: return nil
        }
    case 2:
        switch (path[0], path[1]) {
        case ("products", let pid):  return "Product #\(pid)"
        case ("customers", let cid): return "Customer #\(cid)"
        default: return nil
        }
    case 3:
        switch (path[0], path[1], path[2]) {
        case ("customers", let cid, "orders"):
            return "List of orders for customer #\(cid)"
        default: return nil
        }
        // ...
    default: return nil
    }
}
```

This is ugly and quite verbose. Exactly what we don't want here on this blog. So how do we solve this?


## A tuple of fixed length

What about using a tuple of fixed length, and use `nil` values at the end if it's shorter? Sure that's wouldn't be a great way to represent persistant data across our app, but using that kind of format only inside our `switch` is ok, and will become very handy to treat every case with the same tuple of fixed length.

But how to build such a tuple? Well of course one could do a `switch` again:

```swift
switch path.count {
    case 0: return (nil, nil, nil)
    case 1: return (path[0], nil, nil)
    case 2: return (path[0], path[1], nil)
    default: return (path[0], path[1], path[2])
}
```

Or you could do all of this inline, which would even be uglier:

```swift
return (path.count <= 0 ? nil : path[0], path.count <= 1 ? nil : path[1], path.count <= 2 ? nil : path[2], …)
```

Now you imagine that you have a maximum path of 7 or 8 components, and this starts to get way too verbose… and that's just to build the tuple, before even starting doing any pattern matching!

That's not really elegant, and still not satisfying.

## Using a Generator

That's where I pull another trick from my hat: using a `Generator`

If you don't know what a `Generator` is in the Swift standard library, it's quite simple really. It's like an iterator in C++, basically. It's an object which has a `next()` method, which returns the next item in the sequence it iterates over, or `nil` once it reached the end of the sequence.

So how we're gonna use that fact to build our tuple? Simple! Every `SequenceType` (and thus any `Array` in particular) have a generator, and we just have to build our tuple by calling `next()` for each item. It will start filling the last items of the tuple with `nil` if the array is shorter:

```swift
let path : [String] = …
// ask for a generator that will iterate on the array
var g = path.generate() // Note: we need it to be a var because g.next() is mutating
let tuple = (g.next(), g.next(), g.next(), g.next())
```

And that's it! if `path` only has two values, e.g. `["a", "b"]`, then `tuple` will be `("a", "b", nil, nil)`. No need to `switch` depending on the `path.count` anymore!

## Finally the elegant solution!

We now have all the tools to use a much more simple — and unique — `switch` to parse our URL and its variable parts, whatever the number of path components it contains.

Let's use an enum to represent the various possible requests we are able to process, with associated values to hold the variable parameters:

```swift
enum Request {
    case ProductsList                         // "/products"
    case Product(productID: Int)              // "/products/:pid"
    case CustomersList                        // "/customers"
    case Customer(customerID: Int)            // "/customers/:cid"
    case OrdersList(customerID: Int)          // "/customers/:cid/orders"
    case Order(customerID: Int, orderID: Int) // "/customers/:cid/orders/:oid"
}
```

With the tricks we saw eariler, we're now able to build a `Request` instance using a `[String]` representing the path components, with a single `switch`. Of course an initializer is the perfect candidate for that, and it will be failable because the components could match none of the expected path, or have a path whose IDs components are not convertible into `Int` values for example (let's use a `guard` statement to catch those potential conversion failures, because they're awesome).

This gives us the following code for our initializer [^1] [^2] [^3]:

```swift
extension Request {
    init?(path: [String]) {
        var g = path.generate() // use a generator to build our tuple
        switch (g.next(), g.next(), g.next(), g.next(), g.next()) {
        case ("products"?, nil, _, _, _):
            self = .ProductsList
        case ("products"?, let spid?, nil, _, _):
            guard let pid = Int(spid) else { return nil }
            self = .Product(productID: pid)
        case ("customers"?, nil, _, _, _):
            self = .CustomersList
        case ("customers"?, let scid?, nil, _, _):
            guard let cid = Int(scid) else { return nil }
            self = .Customer(customerID: cid)
        case ("customers"?, let scid?, "orders"?, nil, _):
            guard let cid = Int(scid) else { return nil }
            self = .OrdersList(customerID: cid)
        case ("customers"?, let scid?, "orders"?, let soid?, nil):
            guard let cid = Int(scid), oid = Int(soid) else { return nil }
            self = .Order(customerID: cid, orderID: oid)
        default: return nil
        }
    }
}
```

[^1]: You can see in that code that I'm using question marks, like `"products"?`, in the pattern matching `cases`. That's because our tuple contains optional `String?` elements, and pattern matching will expect to match with tuples of the exact same type, so the strings we use should in our `case` patterns must also be optionals. I also use `let spid?` to ensure that this `spid` variable doesn't bind to `nil`. As a reminder, in that context `x?` is some Swift 2.0's syntaxic sugar which is equivalent to `.Some(x)`.

[^2]: The length of my tuple (5 items) is 1 more than the max length of the paths I want to process. This is to ensure that `/customers/:cid/orders/:oid/foo/bar` won't `return .Order(customerID: …, orderID: …)` — by making sure the `let soid?` non-nil component is followed by a `nil`, and thus is the end of the path.

[^3]: I'm using `_` for the last components after the first `nil`, because I don't really care about their value: given how I built my tuple, I know they can't be anything other than `nil`, so why bother mathing them? Of course you could use `nil` for those values instead of `_` here, but I feel it makes the code more clean and simple.

## The final touch

The only thing left if we want to finish the exercice in full is to extract the array of path components from the URL and give it to our `Request(path:…)` initializer.

We'll obviously use `NSURLComponents` to split the `URL` into its `host`, `path`, etc, then `NSString.pathComponents` to split the path into an array of dirs. In addition:

* We'll want to get rid of the leading `/` as it'll always be present in an absolute URL and we want to avoid the need to match it in every `case` of our `switch` as it's known to always be there
* If we have a trailing `/`, we want to get rid of it too, because in our specific case we want both URLs ending with `/customers/5` and `/customers/5/` to be parsed as `.Customer(customerID: 5)`


```swift
import Foundation

func parse(url: NSURL) -> Request? {
    if let comps = NSURLComponents(URL: url, resolvingAgainstBaseURL: false),
        let path = comps.path where comps.host == "mywebsite.org"
    {
        let pathComps = (path as NSString).pathComponents
        if pathComps.first == "/" {
            var canonicalComps = pathComps.dropFirst()
            if canonicalComps.last == "/" {
                // In case we have a trailing "/", ignore (drop) it
                canonicalComps = canonicalComps.dropLast()
            }
            return Request(path: Array(canonicalComps))
        }
    }
    return nil
}

if let url = NSURL(string: "http://mywebsite.org/customers/12/orders"),
    let req = parse(url) {
    print(req) // Prints: OrdersList(12)
}
```

Et voila!
