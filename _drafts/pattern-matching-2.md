---
layout: post
title: "Pattern Matching, Part 2: tuples and type"
date: 2016-03-27
categories: swift
---

What if anything other than enum?

## Pattern matching with tuples

// Point (x,y) where …

## Checking the type

Let's have 3 structs all conforming to the same protocol

```swift
protocol Medium {
  var title: String { get }
}
struct Book: Medium {
  let title: String
  let author: String
  let year: Int
}
struct Movie: Medium {
  let title: String
  let director: String
  let year: Int
}
struct WebSite: Medium {
  let url: NSURL
  let title: String
}

// And an array of Media to switch onto
let media: [Medium] = [
  Book(title: "20,000 leagues under the sea", author: "Jules Vernes", year: 1870),
  Movie(title: "20,000 leagues under the sea", director: "Richard Fleischer", year: 1955)
]
```

Then how do we switch according to the type of `Medium`?
Notice "as" to type-check+cast, "is" to only type-check

```swift
for medium in media {
  // The title part of the protocol, so no need for a switch there
  print(medium.title)
  // But for the other properties, it depends on the type
  switch medium {
  case let b as Book:
    print("Book published in \(b.year)")
  case let m as Movie:
    print("Movie released in \(m.year)")
  case is WebSite:
    print("A WebSite")
  default:
    print("No year info for \(medium)")
  }
}
```
