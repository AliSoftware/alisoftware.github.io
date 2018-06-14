---
layout: post
title: Exploring @dynamicMemberLookup
date: 2018-06-14
categories: swift exploration
---

With Xcode 10 and Swift 4.2, the new `@dynamicMemberLookup` proposal is now available in Swift. Let's have some fun with it.

## Welcome Xcode 10

With the WWDC'18 just finished, the first beta of Xcode 10 is already available and really pleasant with a lots of very welcome improvements, including:

* way better autocompletion: more contextual completion, more reactive, more accurate, also works in auxiliary files in Playgrounds, …
* better Playgrounds: errors and warnings properly shown inline in red/yellow ribbons, new REPL mode is really cool, way more stable, … they are actually usable again!

But Xcode 10 also comes with Swift 4.2 included. Which brings some interesting new features recently implemented through the Swift-Evolution process and included in this release of Swift

## Welcome Swift 4.2… and @dynamicMemberLookup

One of the new features included in Swift 4.2 is [dynamicMemberLookup](https://github.com/apple/swift-evolution/blob/master/proposals/0195-dynamic-member-lookup.md). This allows to call object properties that will be dynamically resolved at runtime.

This proposal had some controversy, and one thing I didn't personally like on this new feature is that it meant that typed annotated with `@dynamicMemberLookup` would not, by design, show any potential compilation error. Which is understandable, as the whole need for that proposal was to be able to call properties which we didn't know at compile-time.

But this also meant that once the type was annotated with `@dynamicMemberLookup`, you wouldn't be able to choose at call-site if you wanted an expression to allow dynamic member lookup or wanted the expression on that type to be type-checked

### Example of a problematic dynamicMemberLookup

Let's take the example given in the SE-0195 proposal:

```swift
@dynamicMemberLookup
enum JSON {
  case intValue(Int)
  case stringValue(String)
  case arrayValue(Array<JSON>)
  case dictionaryValue(Dictionary<String, JSON>)
  subscript(dynamicMember member: String) -> JSON? {
    if case .dictionaryValue(let dict) = self {
      return dict[member]
    }
    return nil
  }
  var count: Int {
    switch self {
    case .intValue, .stringValue: return 1
    case .arrayValue(let a): return a.count
    case .dictionaryValue(let d): return d.count
    }
  }
}
```

The problem I have with this example is that we can call whatever property we want on a `JSON` instance, because it's `@dynamicMemberLookup` this property lookup will always compile, but sometimes returns the real property and sometimes doing a dynamic lookup, without any distinction at call site:

```swift
let j = JSON.dictionaryValue([
  "comment": .stringValue("Not being able to tell the difference at call site is confusing"),
  "count": .intValue(42),
  "count2": .intValue(1337)
  ])

// Show the issue: hard to know when this is solved dynamically vs at compile time
j.count  // This one will return 3 — from `var count: Int` in `enum JSON`. And not 42 from the dictionary
j.count2 // While this one will return the 1337 value from the dictionary
j.cuont  // And this will compile despite the typo (and return nil because no such key exists in the JSON)
```

## Improving the situation: make it explicit at call site

What I want to be able to do is to distinguish explicitly between calls that are checked at compile time and those that are dynamically looked up (and are not compile-time checked)

An example of the call site I'd find more clear would be like this:

```swift
j.count // compile-time checked. Compiles, calls the property on Dictionary, returns 3
j.cuont // compile-time error
j^.count2 // explicit that this is dynamically looked-up and not compile-time checked. Dynamically lookup inside the dictionary at runtime, and returns the value of key "count2" (which is 1337)
```
## Let's build it!

The solution is actually quite short to implement. We'll create a `^` postfix operator that will wrap the instance it's applied on into some proxy object that is the one being `@dynamicMemberLookup`. That proxy object will just wrap the dictionary we applied `^` on, and on dynamic lookup, will search the key in the dictionary to return the corresponding value (if it exists).

```swift
@dynamicMemberLookup
public struct DynamicLookupProxy {
  let dict: [String: Any]
  public subscript(dynamicMember member: String) -> Any? {
    return dict[member]
  }
}

postfix operator ^
public postfix func ^ (lhs: [String: Any]) -> DynamicLookupProxy {
  return DynamicLookupProxy(lhs)
}
```

And now if we test this on our example:

```swift
let j: [String: Any] = [
  "comment": "Being able to tell the difference at call site is explicitly is nicer",
  "count": 42,
  "count2": 1337
]

j.count // checked at compile time. Returns 3
j^.count // dynamic lookup. Returns 42


j.cuont // compile-time error
j^.cuont // dynamic lookup, but key not found. Returns nil.
```

Success!

## Chain all the things!

The only problem with this is that our dynamic lookup returns Any. This means we have to cast it to the desired object, and we can't chain those calls:

```swift
let j2: [String: Any] = [
  "name": "Olivier",
  "address": [
    "street": "Swift Street",
    "number": 1337,
    "city": "PlaygroundVille"
  ]
]
j2^.name
j2^.address^.street // 🛑 Cannot convert value of type 'Any?' to expected argument type '[String : Any]'. Fix: Insert ' as! ([String : Any])'
```

How do we solve that? Surely [not by the force-cast](/swift/2015/09/06/thinking-in-swift-1/) suggested by that Fix-It!

Well, if instead of returning `Any?` we make the `subscript` generic (yes, that's also a recent addition, in Swift 4!) to make it be able to return some typed value, we could make it infer it being `[String: Any]` in some contexts?

So let's add that generic subscript to our `DynamicLookupProxy` struct:

```swift
@dynamicMemberLookup
public struct DynamicLookupProxy {
  private let dict: [String: Any]
  public init(_ dict: [String: Any]) {
    self.dict = dict
  }

  public subscript<T>(dynamicMember member: String) -> T? {
    return dict[member] as? T
  }
  public subscript(dynamicMember member: String) -> Any? {
    return dict[member]
  }
}
```

_Note: I still kept the `-> Any?` implementation so that if the return type is not specified/inferable, it'll falls back to `Any?` instead of forcing you to specify the type explicitly_

And now look at that!

```swift
j2^.name // Returns "Olivier"
j2^.address?^.street // Returns "Swift Street"
j2^.address?^.zipcode // Returns nil, as there's no zipcode
j2.count // Calls the compile-time property count on Dictionary, returns 2
```

The line `j2^.address` is actually equivalent to `j2["address"]`… so why bother having all that? It's not that different after all, and maybe it's better to keep strings explicit? Maybe… but now we can also chain those properties to access subkeys without having to do explicit casts, as opposed to using string subscripts!

```swift
// Nice call site with our new solution, but still explicit that it's dynamic lookup:
j2^.address?^.street
// What we'd have to write without `@dynamicMemberLookup` to have the same:
(j2["address"] as? [String: Any])?["street"]
```

And even better, since we're now using generics, we can even let the compiler infer the return type of the dynamic member lookup by hinting the type of the variable we store the result to:

```swift
let addr: [String: Any]? = j2^.address
addr?^.street // Returns "Swift Street"
addr?["street"] // same thing, really, but arguably less nice

addr?.count // Still no ambiguity: no '^' here, this one is compile-time checked (returns 3)
```


## Avoid the question mark?

The last thing we could consider doing is to get rid of the `?` and consider that dynamic member lookup on a `nil` value should directly return `nil` — without having to rely on optional chaining to do that.

Indeed, having to use `?` in front of every `^` in the chain (besides the first one) can feel repetitive.

The solution actually consists of a simple change: just allow our `^` operator to be applied on optional dictionaries. If our dictionary isn't optional, the compiler will lift it to optional for us, and if it's optional, we can imagine just return our `DynamicLookupProxy` but with an empty dictionary — so that any following dynamic lookup will always return a `nil` value anyway.

So just change our `func ^` signature and implementation to this:

```swift
public postfix func ^ (lhs: [String: Any]?) -> DynamicLookupProxy {
  return DynamicLookupProxy(lhs ?? [:])
}
```

And we're good to go! Now we can update our call sites to look like this:

```swift
// No need for the extra `?` before the second `^` anymore.
let street: String? = j2^.address^.street // Still returns "Swift Street"
```

Was that a good idea? Wasn't it better to keep explicit the fact that the return type of any dynamic lookup was optional by requiring the optional chaining syntax and that `?` before continuing chaining with `^.street`? That's another debate, and I'll let you decide. But at least you know that's possible 😉


## More exploration: creating a context/scope

If you've made it so far, maybe you're even ready to take it a step further? (if not, you can skip this last part, I won't blame you 😄)

In the playground attached below, I've explored that idea a bit more, by allowing another syntax. This one looks more like creating a context or scope in which everything we call is dynamically looked up (instead of looking like chaining `^` calls):

```swift
let street: String? = j2.dynamicLookup { $0.address?.street }
```

To implement this, I've created a separate proxy type called `DynamicLookupContext` similar to our first `DynamicLookupProxy`, but this time, our subscript on `DynamicLookupContext` does not returns the value found in the dictionary, but instead wraps that value in a new `DynamicLookupContext` and return that. Which means we can chain them (without having to re-lift them at every step like we had to do by repeating `^` at each step in our previous solution).

Then the `func dynamicLookup<T>` I've added on `[String: Any]` takes care of wrapping that initial dictionary we want to query into a `DynamicLookupContext`, let you execute your chain of dynamic lookups on this (via a closure you provide), and extract the output from the result returned by the end of that chain:

```swift
@dynamicMemberLookup
public struct DynamicLookupContext {
  let value: Any

  public subscript(dynamicMember member: String) -> DynamicLookupContext? {
    let dict = value as? [String: Any]
    guard let value = dict?[member] else { return nil }
    return DynamicLookupContext(value: value)
  }
  public subscript(index: Int) -> DynamicLookupContext? {
    guard let array = value as? [Any] else { return nil }
    return DynamicLookupContext(value: array[index])
  }
}

public extension Dictionary where Key == String, Value: Any {
  func dynamicLookup<T>(execute: (DynamicLookupContext) -> DynamicLookupContext?) -> T? {
    let wrapped = DynamicLookupContext(value: self)
    let result = execute(wrapped)
    return result?.value as? T
  }
}
```

As a result, you can now choose between either syntax, whichever you like best:

```swift
// With our DynamicLookupProxy and ^ operator we saw before:
let street1: String? = j2^.address^.street
// With our new DynamicLookupContext and dynamicLookup method above:
let street2: String? = j2.dynamicLookup { $0.address?.street }
// Without any usage of those @dynamicMemberLookup tricks, we'd have to do this instead:
let addr = j2["address"] as? [String: Any]
let street3 = addr["street"] as? String
```

> I've [attached an Xcode 10 playground](/assets/DynamicMemberLookup.playground.zip) with my experimentation around this and the code presented above.

## Conclusion

What do you think of this solution?

To be honest, I am torn as to consider this really useful in production code or just a toy idea.

* I like the fact that it's now clear at call site when we intend to do a dynamic lookup and when we intend to do a real property access that is checked at compile time. We can choose when we want one or the other.

* On the other hand, the fact that the code still looks like real Swift instructions — without explicit String-typed keys clearly appearing — can make this confusing; people have to know what this `^` operator we created does and that it allows part of the expression to not be checked by the compiler.

As always when you create a custom operator — and even more so here where it kind of "disable the compile-time checks on some parts of the expression" — it can be hard for newcomers on your code to understand how that works, so even if this is fun and powerful, we also have to keep that in mind.


What do you all think? Do you like the idea? Would you use that in production to allow you to quickly parse parts of an arbitrary JSON or Plist or dictionary? Or should it just be kept as a fun experiment in a playground?
