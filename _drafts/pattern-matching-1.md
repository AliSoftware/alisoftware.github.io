---
layout: post
title: "Pattern Matching, Part 1: switch that enum"
date: 2016-03-27
categories: swift
---

From a simple `switch` to complex expressions, pattern matching in Swift can be quite powerful. Today we're gonna start exploring pattern matching by seing some advanced usages of `switch`, before going further in a later article.

## Switch Basics

The simplest and more common usage of pattern matching in Swift is via a `switch` statement. You hopfully already know their simplest form, e.g.:

```swift
enum Direction {
  case North, South, East, West
}

// We can easily switch between simple enum values
extension Direction: CustomStringConvertible {
  var description: String {
    switch self {
    case North: return "⬆️"
    case South: return "⬇️"
    case East: return "➡️"
    case West: return "⬅️"
    }
  }
}
```

But `switch` can go even further, allowing you to match against **patterns** containing variables, and binding to those variables when it matches. This applies typically to `enum` with associated values:

```swift
enum Media {
  case Book(title: String, author: String, year: Int)
  case Movie(title: String, director: String, year: Int)
  case WebSite(url: NSURL, title: String)
}

extension Media {
  var mediaTitle: String {
    switch self {
    case .Book(title: let aTitle, author: _, year: _):
      return aTitle
    case .Movie(title: let aTitle, director: _, year: _):
      return aTitle
    case .WebSite(url: _, title: let aTitle):
      return aTitle
    }
  }
}

let book = Media.Book(title: "20,000 leagues under the sea", author: "Jules Verne", year: 1870)
book.mediaTitle
```

The generic syntax in this case is `case MyEnum.EnumValue(let variable)` to tell "if the value is a `MyEnum.EnumValue` — which has an associated value — then bind the variable `variable` to that associated value".

When the `media` instance match one of the `case` — like with `book` which is a `Media.Book` and matches the first `case` of the `switch` — then **a new variable `let aTitle` is created** and the associated value `title` is _bound_ to this given variable.
That's why the `let` is needed there, because that's gonna create a new variable (well, constant) if it matches.

Note that you can write the `let` in front of the whole expression, instead of using it in front of each variable, e.g. these two lines are equivalent:

```swift
case .Book(title: let aTitle, author: let anAuthor, year: let aYear): …
case let .Book(title: aTitle, author: anAuthor, year: aYear): …
```

Notice also the use of the wildcard pattern `_` in the above code, which basically means "I expect to be something there, but I don't care about it, so don't bother binding it to a variable as I won't use it anyway". So that's somehow like a placeholder for a value we won't use.


## Using fixed values

Remember that `case`  is still about **pattern matching**, so it tries to **match** what you try to compare it with. This means that you can also use a constant values to check if it matches. For example:

```swift
extension Media {
  var isFromJulesVerne: Bool {
    switch self {
    case .Book(title: _, author: "Jules Verne", year: _): return true
    case .Movie(title: _, director: "Jules Verne", year: _): return true
    default: return false
    }
  }
}
book.isFromJulesVerne
```

Granted, this example is not really useful _per se_, but it's just to show you that you can bind to constant values too. I mentionned it because I've already seen code that binds a value to a variable then check if that variable is equal to a constant… instead of pattern-match directly with the constant!

A more useful and generic example could be something like:

```swift
extension Media {
  func checkAuthor(author: String) -> Bool {
    switch self {
    case .Book(title: _, author: author, year: _): return true
    case .Movie(title: _, director: author, year: _): return true
    default: return false
    }
  }
}
book.checkAuthor("Jules Verne")
```

Notice here that althrough we use `author` in the `case` patterns, we don't need to use `let` here (unlike in the preview paragraph). That's because in this case, we won't create a variable to bind a value to it . Instead, we use a constant which already has a value (provided by the function parameter) — not creating a new one that is gonna be bound to the value matched is `self`.

## Binding multiple patterns at once

As per Swift 2.2, we can't bind multple patterns at once. So for example this isn't possible yet, because here we try to bind variables both if `self` matches `.Book` or `.Movie` , and bind a variable in both cases:

```swift
extension Media {
  var mediaTitle2: String {
    switch self {
      // Error: 'case' labels with multiple patterns cannot declare variables
    case let .Book(title: aTitle, author: _, year: _), let .Movie(title: aTitle, director: _, year: _):
      return aTitle
    case let .WebSite(url: _, title: aTitle):
      return aTitle
    }
  }
}
```

This is understandable in most cases, like what would you expect the code to do if you tried to write `case let .Book(title: aTitle, author: _, year: _), let .Movie(title: _, director: _, year: aYear)`? How would you be able to use the bound variables `aTitle` or `aYear` in your `case` code then? If it's a `.Book` then only `aTitle` would have been bound, so what about `aYear`? What if you tried to use that `aYear` variable in that `case`? Wouldn't probably make sense.

But one might think that in the specific case when you try to bind variables of the same type and with the same name, that would still make sense to work, like in the example above where we try to bind with `aTitle` in both cases (`.Book` and `.Movie`). So why is this not possible in that specific case? Well fear not, [this Swift-Evolution Proposal SE-0043](https://github.com/apple/swift-evolution/blob/master/proposals/0043-declare-variables-in-case-labels-with-multiple-patterns.md) has been accepted and will fix that gap in Swift 3.

## Using tuples without argument labels

Note that when dealing with `enum` with associated values, we can consider the associated values to be one single associated value represented as a tuple containing all the real associated values. This has two consequences:

* First you can omit the argument labels, and the `case` will still work:

```swift
extension Media {
  var mediaTitle2: String {
    switch self {
    case let .Book(title, _, _): return title
    case let .Movie(title, _, _): return title
    case let .WebSite(_, title): return title
    }
  }
}
```

* Second, you can also treat the associated value like a unique, big tuple, then access its individual elements:

```swift
extension Media {
  var mediaTitle3: String {
    switch self {
    case let .Book(tuple): return tuple.title
    case let .Movie(tuple): return tuple.title
    case let .WebSite(tuple): return tuple.title
    }
  }
}
```

## Using Where

Pattern matching allows way more powerful stuff than just comparing two enums. You can add conditions to the comparisons, using a `where` clause, like this:

```swift
extension Media {
  var publishedAfter1930: Bool {
    switch self {
    case let .Book(_, _, year) where year > 1930: return true
    case let .Movie(_, _, year) where year > 1930: return true
    case .WebSite: return true // same as "case .WebSite(_)" but we ignore the associated tuple value
    default: return false
    }
  }
}
```

## What's next?

This article was pretty simple to remind you of the basics of pattern matching in `switch`. The next parts are gonna talk about more advanced usages, including:

* using `switch` with anything other than `enum` (especially pattern matching with `tuples`, `structs`, `is` and `as`).
* using pattern matching with other statements, including `if case`, `guard case`, `for case`, `=~`, …
* Nested patterns, including ones containing `Optional` values
* Combining them all to create some magic.
