---
layout: post
title: "Pattern Matching, Part 3: ranges & custom pattern matching"
date: 2016-03-29
categories: swift
---

## Ranges

Using ranges

```
extension Book {
  var century: String {
    switch year {
    case 1800..<1900: return "XIX"
    case 1900..<2000: return "XX"
    case 2000..<2100: return "XXI"
    default: return "\(year/100)"
    }
  }
}
```

## Declaring our own pattern matching operator

Actually works with everything implementing the ~= operator

1900..<2000 ~= 1950

1900..<2000 ~= book

## Syntactic sugar

Using x? as a syntactic sugar for .Some(x)
