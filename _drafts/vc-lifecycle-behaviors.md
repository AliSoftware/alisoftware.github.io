---
layout: post
title: ViewController LifeCycle Behaviors
categories: swift analytics
published: false
---

## ViewController LifeCycle Behaviors

For Analytics & all


```swift
protocol VCLifeCycleBehavior {
  func viewDidLoad()
  func viewWillAppear()
  func viewDidAppear()
  func viewWillDisappear()
  func viewDidDisappear()
  func traitCollectionDidChange()
  func didReceiveMemoryWarning()
  …
}

extension VCLifeCycleBehavior() {
  func viewDidLoad() {}
  func viewWillAppear() {}
  func viewDidAppear() {}
  func viewWillDisappear() {}
  func viewDidDisappear() {}
  func traitCollectionDidChange() {}
  func didReceiveMemoryWarning() {}
}
```

```swift
BlockVCLifeCycleBehavior: VCLifeCycleBehavior {
  let onViewDidLoad: () -> () = {}
  let onViewWillAppear: () -> () = {}
  let onViewDidAppear: () -> () = {}
  let onViewWillDisappear: () -> () = {}
  let onViewDidDisappear: () -> () = {}
  let onTraitCollectionDidChange: () -> () = {}
  let onDidReceiveMemoryWarning: () -> () = {}

  func viewDidLoad() { onViewDidLoad() }
  func viewWillAppear() { onViewWillAppear() }
  func viewDidAppear() { onVewDidAppear() }
  func viewWillDisappear() { onViewWillDisappear() }
  func viewDidDisappear() { onViewDidDisappear() }
  func traitCollectionDidChange() { onTraitCollectionDidChange() }
  func didReceiveMemoryWarning() { onDidReceiveMemoryWarning() }
}

let logAppearance = BlockVCLifeCycleBehavior(
  onViewDidAppear = { print("Appeard!") },
  onViewDidDisappear = { print("Disappeared!") }
)
```

```swift
struct AnalyticsBehavior: VCLifeCycleBehavior {
  let screenName: String
  func viewDidAppear() {
    AnalyticsTracker.sharedInstance.trackScreen(screenName)
  }
}

let vc = SomeViewController()
vc.addBehavior(AnalyticsBehavior(screenName: "SomeScreen"))
vc.addBehavior(logAppearance)
```

```swift
class BehaviorViewController {
  let behaviors: [VCLifeCycleBehavior]
  init(behaviors: [VCLifeCycleBehavior]) { self.behaviors = behaviors }
  required init initWithCoder(decoder: NSCoder) { fatalError(…) }
  
  override func viewDidLoad() {
    behaviors.forEach { $0.viewDidLoad() }
  }
  
  …
}
extension UIViewController {
  func addBehaviors(behaviors: [VCLifeCycleBehavior]) {
    let bvc = BehaviorViewController(behaviors: behaviors)
    self.addChildViewController(bvc)
    self.view.addSubview(bvc.view)
    bvc.didMoveToParent(self)
  }
  func addBehavior(behavior: VCLifeCycleBehavior) {
    addBehaviors([behavior])
  }
}
```


### Alternatives

* Using inheritance, and declare `class VCLifeCycleBehavior` (instead of using Protocol-Oriented Programming)
* Parent class will do nothing, subclasses will `override` 
* We can imagine extending the parent class using:

```swift
extension VCLifeCycleBehavior {
  static func analytics(_ screenName: String) -> VCLifeCycleBehavior {
    return AnalyticsBehavior(screenName: screenName)
  }
}

vc.addBehavior(.analytics("SomeScreen"))
```
