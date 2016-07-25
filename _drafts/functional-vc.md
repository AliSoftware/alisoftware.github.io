---
layout: post
title: "Functional ViewControllers"
categories: swift architecture
published: false
---

### Intro to Rx

https://realm.io/news/altconf-scott-gardner-reactive-programming-with-rxswift/

### Using Promises

Future & Promises, BrightFuture, PromiseKit

```swift
let nc = UINavigationController()
self.presentViewController(nc, animated: true) {}

firstly {
  let vc1 = ViewController1()
  nc.pushViewController(vc1, animated: true)
  return vc1.nextPromise()
}
.then {
  let vc2 = ViewController2()
  nc.pushViewController(vc2, animated: true)
  return vc2.nextPromise()
}
.then {
  let vc3 = ViewController3()
  nc.pushViewController(vc3, animated: true)
  return vc3.nextPromise()
}
…
.finally {
  self.dismissViewController(true) {}
}
…
```

### Using RxSwift

```swift
let nc = UINavigationController()
self.presentViewController(nc, animated: true) {}
var disposeBag = DisposeBag()


let vc1 = ViewController1()
nc.pushViewController(vc1, animated: true)
vc1.nextObservable()
.flatMap {
  let vc2 = ViewController2()
  nc.pushViewController(vc2, animated: true)
  return vc2.nextObservable()
}
.flatMap {
  let vc3 = ViewController3()
  nc.pushViewController(vc3, animated: true)
  return vc3.nextObservable()
}
…
.subscribe {
  self.dismissViewController(true) {}
  disposeBag = DisposeBag()
}
.addDisposableTo(disposeBag)
```
