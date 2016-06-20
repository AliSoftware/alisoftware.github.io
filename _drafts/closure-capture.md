---
layout: post
title: "Closures Capturing Semantics"
categories: swift closures
published: false
---

Talk about the behavior of closures capturing values in Swift.

* Behaves more like ObjC's `__block`
* Swift always captures the reference to the variable so it is possible to modify the value from within the closure
* You can capture the value as a constant at closure-creation time using a capture group `[constant = value]`

```
import UIKit
import XCPlayground
XCPlaygroundPage.currentPage.needsIndefiniteExecution = true
​
func perform(initial: Int) {
    var value: Int = initial
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 1), dispatch_get_main_queue()) { [myConstant = value] in
        print("constant in first block \(myConstant)")
        print("captured \(value)")
        value = initial * 2
    }
​
    value = initial + 1
​
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 2), dispatch_get_main_queue()) { [myConstant = value] in
        print("constant in second block \(myConstant)")
        print("captured \(value)")
    }
}
​
perform(10)
​
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 3), dispatch_get_main_queue()) {
    print("--------")
    perform(30)
}
​
// Console:
constant in first block 10
captured 11
constant in second block 11
captured 20
--------
constant in first block 30
captured 31
constant in second block 31
captured 60
```

Credits to @merowing for the code.
