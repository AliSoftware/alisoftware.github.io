---
layout: post
title: Using Generics to improve TableView cells
date: 2016-01-06
categories: swift generics
image: assets/ReusableLogo.png
translations:
  - lang: Chinese
    flag: 🇨🇳
    author: the SwiftGG team
    url: http://swift.gg/2016/01/27/generic-tableviewcells/
---

Happy New Year everybody 🎇🎉🎊🎆! My first post of 2016 will be a useful trick I want to share, which will demonstrate the power of Swift's generics and how they can be really handy when dealing with `UITableViewCells` and `UICollectionViewCells`.

## Introduction

I don't like string-typed stuff. [Using constants](/swift/enum/constants/2015/07/19/enums-as-constants/) is way better than using string literals to identify stuff uniquely.

But when it comes to `UITableViewCell` or `UICollectionViewCell` and their identification via `reuseIdentifiers`, I use what I think is an even better solution: using Swift's generics + [Mixins](/swift/protocol/2015/11/08/mixins-over-inheritance/) to make them magic!

## The magic

The idea is to declare the `reuseIdentifier` as a `static var` on the `UITableViewCell` (resp. `UICollectionViewCell`)  subclass, then use it transparently to instanciate cells of that class.

Let's declare that as a protocol first, so that we can [use it as a Mixin](/swift/protocol/2015/11/08/mixins-over-inheritance/) later:

```swift
protocol Reusable: class {
  static var reuseIdentifier: String { get }
}

extension Reusable {
  static var reuseIdentifier: String {
    // I like to use the class's name as an identifier
    // so this makes a decent default value.
    return String(Self)
  }
}
```

Then the magic happens when we implement the `dequeueReusableCell(…)` method, using **generics**:

```swift
func dequeueReusableCell<T: Reusable>(indexPath indexPath: NSIndexPath) -> T {
  return self.dequeueReusableCellWithIdentifier(T.reuseIdentifier, forIndexPath: indexPath) as! T
}
```

Now thanks to Swift's type inference, **this method will use the call-site context to infer the actual type of `T`**, and thus this type can be somehow seen as… **"retro-injected" into the implementation!** ✨

```swift
  let cell = tableView.dequeueReusableCell(indexPath: indexPath) as MyCustomCell
```

> Observe how the `reuseIdentifier` used internally… entirely depends on what type you tell the Swift compiler to **return**! That's why I somehow see it like if the type is "retro-injected" in the implementation… and why I 😍 it so much!

Isn't that beautiful?

![It's magic](/assets/magic.gif)
{: style="text-align: center"}

## Going further

Of course, why limit this trick to `dequeueReusableCellWithIdentifier` when you can do the same with `registerNib(_, forCellWithReuseIdentifier:)`,as well as applying the same idea to `UICollectionViewCells`, supplementary views, etc.?

As `UITableViewCells` and `UICollectionViewCells` might be registered either using a class name (`registerClass(_, forCellWithReuseIdentifier:)`) or nib (`registerNib(_, forCellWithReuseIdentifier:)`), we'll add a `static var nib: UINib?` class property to our protocol, and use registration with a nib if one is provided, using class otherwise.

## The code

Here is the actual code I use in my projects

> _[EDIT 20/01/2016]_  
> The code is now [available on GitHub](https://github.com/AliSoftware/Reusable)!

```swift
import UIKit

protocol Reusable: class {
  static var reuseIdentifier: String { get }
  static var nib: UINib? { get }
}

extension Reusable {
  static var reuseIdentifier: String { return String(Self) }
  static var nib: UINib? { return nil }
}

extension UITableView {
  func registerReusableCell<T: UITableViewCell where T: Reusable>(_: T.Type) {
    if let nib = T.nib {
      self.registerNib(nib, forCellReuseIdentifier: T.reuseIdentifier)
    } else {
      self.registerClass(T.self, forCellReuseIdentifier: T.reuseIdentifier)
    }
  }

  func dequeueReusableCell<T: UITableViewCell where T: Reusable>(indexPath indexPath: NSIndexPath) -> T {
    return self.dequeueReusableCellWithIdentifier(T.reuseIdentifier, forIndexPath: indexPath) as! T
  }

  func registerReusableHeaderFooterView<T: UITableViewHeaderFooterView where T: Reusable>(_: T.Type) {
    if let nib = T.nib {
      self.registerNib(nib, forHeaderFooterViewReuseIdentifier: T.reuseIdentifier)
    } else {
      self.registerClass(T.self, forHeaderFooterViewReuseIdentifier: T.reuseIdentifier)
    }
  }

  func dequeueReusableHeaderFooterView<T: UITableViewHeaderFooterView where T: Reusable>() -> T? {
    return self.dequeueReusableHeaderFooterViewWithIdentifier(T.reuseIdentifier) as! T?
  }
}

extension UICollectionView {
  func registerReusableCell<T: UICollectionViewCell where T: Reusable>(_: T.Type) {
    if let nib = T.nib {
      self.registerNib(nib, forCellWithReuseIdentifier: T.reuseIdentifier)
    } else {
      self.registerClass(T.self, forCellWithReuseIdentifier: T.reuseIdentifier)
    }
  }

  func dequeueReusableCell<T: UICollectionViewCell where T: Reusable>(indexPath indexPath: NSIndexPath) -> T {
    return self.dequeueReusableCellWithReuseIdentifier(T.reuseIdentifier, forIndexPath: indexPath) as! T
  }

  func registerReusableSupplementaryView<T: Reusable>(elementKind: String, _: T.Type) {
    if let nib = T.nib {
      self.registerNib(nib, forSupplementaryViewOfKind: elementKind, withReuseIdentifier: T.reuseIdentifier)
    } else {
      self.registerClass(T.self, forSupplementaryViewOfKind: elementKind, withReuseIdentifier: T.reuseIdentifier)
    }
  }

  func dequeueReusableSupplementaryView<T: UICollectionViewCell where T: Reusable>(elementKind: String, indexPath: NSIndexPath) -> T {
    return self.dequeueReusableSupplementaryViewOfKind(elementKind, withReuseIdentifier: T.reuseIdentifier, forIndexPath: indexPath) as! T
  }
}
```

## Example usage

Here's how you'd declare your `UITableViewCell` subclasses:

```swift
class CodeBasedCustomCell: UITableViewCell, Reusable {
  // By default this cell will have a reuseIdentifier or "MyCustomCell"
  // unless you provide an alternative implementation of `var reuseIdentifier`
  // ...
}

class NibBasedCustomCell: UITableViewCell, Reusable {
  // Here we provide a nib for this cell class
  // (instead of relying of the protocol's default implementation)
  static var nib: UINib? {
    return UINib(nibName: String(NibBasedCustomCell.self), bundle: nil)
  }
  // ...
}
```

Then to use them in a `UITableViewDelegate`/`UITableViewDataSource`:

```swift
class MyTableViewController: UITableViewController {
  override func viewDidLoad() {
    super.viewDidLoad()
    tableView.registerReusableCell(CodeBasedCustomCell.self) // This will register using the class without using a UINib
    tableView.registerReusableCell(NibBasedCustomCell.self) // This will register using NibBasedCustomCell.xib
  }
  
  override func tableView(tableView: UITableView, cellForRowAtIndexPath indexPath: NSIndexPath) -> UITableViewCell {
    let cell: UITableViewCell
    if indexPath.section == 0 {
      cell = tableView.dequeueReusableCell(indexPath: indexPath) as CodeBasedCustomCell
    } else {
      cell = tableView.dequeueReusableCell(indexPath: indexPath) as NibBasedCustomCell
    }
    return cell
  }
}
```

## Alternate solutions

Some might prefer to split the `Reusable` protocol in two distinct protocols like this, to distinguish between nib-based cells and class-based cells:

```swift
protocol Reusable: class {
  static var reuseIdentifier: String { get }
}
extension Reusable {
  static var reuseIdentifier: String { return String(Self) }
}

protocol NibReusable: Reusable {
  static var nib: UINib { get }
}
extension NibReusable {
  static var nib: UINib {
    return UINib(nibName: String(Self), bundle: nil)
  }
}
```

This allows you to have a default implementation for nib-based cells too — instead of reimplementing it in subclasses which are nib-based.

But that also forces you to add more implementations on `UITableView` & `UICollectionView` (one for each of the two protocols), so… that's up to you to choose where you want to put the balance ⚖😉

## Addendum: Now available on Cocoapods 🎉

_[Added on 20 Jan 2016]_

The code is now [available on GitHub](https://github.com/AliSoftware/Reusable) and published as a Swift Package and [as a CocoaPod](https://cocoapods.org/pods/Reusable) — so you can now easily add it to your Swift projects!

Feel free to make PRs to improve it 😉

----

Hope you enjoyed this trick, and see you next time! 🎉
