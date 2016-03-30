---
layout: post
title: "Creating new types"
date: 2016-05-10
categories: swift
---

In this article we'll see how creating new types even for very simple values like a `Duration` or a `Currency` can be simple, useful, and provide additional safety to your code as well as a cleaner API.


```swift
struct Duration {
  let seconds: Double
  init(_ totalSeconds: Double) {
    self.seconds = totalSeconds
  }
  init(hours: Int = 0, minutes: Int = 0, seconds: Int = 0) {
    self.seconds = Double((hours * 60 + minutes) * 60+seconds)
  }
  var minutes: Double {
    return seconds/60
  }
  var hours: Double {
    return minutes/60
  }
}

extension Duration: CustomStringConvertible {
  var description: String {
    return "\(Int(self.hours))h\(Int(self.minutes)%60)'\(Int(self.seconds)%60)\""
  }
}

func +(lhs: Duration, rhs: Duration) -> Duration {
  return Duration(lhs.seconds + rhs.seconds)
}

func -(lhs: Duration, rhs: Duration) -> Duration {
  return Duration(lhs.seconds - rhs.seconds)
}

func *(lhs: Duration, rhs: Double) -> Duration {
  return Duration(lhs.seconds * rhs)
}

func *(lhs: Double, rhs: Duration) -> Duration {
  return rhs * lhs
}

func /(lhs: Duration, rhs: Double) -> Duration {
  return Duration(lhs.seconds / rhs)
}

extension Int {
  var seconds: Duration {
    return Duration(seconds: self)
  }
  var minutes: Duration {
    return Duration(minutes: self)
  }
  var hours: Duration {
    return Duration(hours: self)
  }
}

(5.hours + 30.minutes + 17.seconds) * 3
```
