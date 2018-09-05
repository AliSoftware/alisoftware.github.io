---
layout: post
title: "Private properties in protocols"
categories: swift protocols
---

In Swift, protocols can't specify access control to the properties they declare. If a property is listed in a protocol, you have to make conforming types declare those properties explicitly.  
But sometimes, even if you need those properties in order to provide your implementations, you don't want those properties to be used outside the type. Let's see how to workaround that problem.

### A simplified example

Let's see that you want to create a dedicated object to manage your ViewControllers navigation, like a Coordinator.

Every coordinator is gonna have a root `UINavigationController`, and share some common capabilities, like pushing and poping other ViewControllers on it. So at first it might look like this[^1]:

```swift
// Coordinator.swift

protocol Coordinator {
  var navigationController: UINavigationController { get }
  var childCoordinator: Coordinator? { get set }

  func push(viewController: UIViewController, animated: Bool)
  func present(childViewController: UIViewController, animated: Bool)
  func pop(animated: Bool)
}

extension Coordinator {
  func push(viewController: UIViewController, animated: Bool = true) {
    self.navigationController.pushViewController(viewController, animated: animated)
  }
  func present(childCoordinator: Coordinator, animated: Bool) {
    self.navigationController.present(childCoordinator.navigationController, animated: animated) { [weak self] in
      self?.childCoordinator = childCoordinator
    }
  }
  func pop(animated: Bool = true) {
    if let childCoordinator = self.childCoordinator {
      self.dismissViewController(animated: animated) { [weak self] in
        self?.childCoordinator = nil
      }
    } else {
      self.navigationController.popViewController(animated: animated)
    }
  }
}
```

And then when we want to declare a new `Coordinator` object, we'd do something like this:

```swift
// MainCoordinator.swift

class MainCoordinator: Coordinator {
  let navigationController: UINavigationController = UINavigationController()
  var childCoordinator: Coordinator?

  func showTutorialPage1() {
    let vc = makeTutorialPage(1, coordinator: self)
    self.push(viewController: vc)
  }
  func showTutorialPage2() {
    let vc = makeTutorialPage(2, coordinator: self)
    self.push(viewController: vc)
  }

  private func makeTutorialPage(_ num: Int, coordinator: Coordinator) -> UIViewController { … }
}
```

[^1]: That's some simplified example ; don't focus on the implementation of the Coordinator pattern here — that's not really the point of that example, which focuses more about the need to have publicly accessible properties declared in the protocol.

### The problem: leaking implementation details

There are two problems with this solution regarding `protocol` visibility:

* When we want to declare a new `Coordinator` object, we have to explicitly declare a `let navigationController: UINavigationController` property AND a `var childCoordinator: Coordinator?` every time. **Even if we don't use them explicitly** in implementations of our conforming types — they are just there because we need them for the default implementations provided by the `protocol Coordinator` to work.
* Those two properties we have to declare have to be of the same visibility (the implicit `internal` access control level in our case) as our `MainCoordinator`, because that's a requirement of our `protocol Coordinator`. That makes them also visible to the outside, i.e. to code using `MainCoordinator`

So the problem is that we have to declare some properties every time while it's only some implementation details, but also that these implementation details are leaked to the outside interface, allowing consumers of that class to do things they shouldn't be allowed to do, like:

```swift
let mainCoord = MainCoordinator()
// Consumers shouldn't be allowed to access the navigationController directly but they can
mainCoord.navigationController.dismissViewController(animated: true)
// and neither should they be allowed to do stuff like this
mainCoord.childCoordinator = mainCoord
```

You might think that we could just not declare those two properties in the `protocol` in the first place, as we don't want them to be visible. But if we do that, we wouldn't be able to provide default implementations via our `extension Coordinator`, as those default implementations need those properties to exist in order for their code to compile.

One could also hope that Swift would allow declaring those properties `fileprivate` in the protocol, but as of Swift 4, you can't specify any access control attributes inside `protocols`.

So how could we solve that, to both provide those default implementations which require those properties, and not letting them leak to the outside interface?

### A solution

A trick to achive that is to hide those properties inside an intermediate object, and make the properties of that object `fileprivate`.

That way, even if we'll still have conforming types to declare that property in their public interface, consumers of that interface won't be able to access internal properties of that object. While our default implementation of the protocol will be able to access them — as long as it's declared in the same file (as they'll be `fileprivate`).

This would look like this:

```swift
// Coordinator.swift

class CoordinatorComponents {
  fileprivate let navigationController: UINavigationController = UINavigationController()
  fileprivate var childCoordinator: Coordinator? = nil
}

protocol Coordinator: AnyObject {
  var coordinatorComponents: CoordinatorComponents { get }

  func push(viewController: UIViewController, animated: Bool)
  func present(childCoordinator: Coordinator, animated: Bool)
  func pop(animated: Bool)
}

extension Coordinator {
  func push(viewController: UIViewController, animated: Bool = true) {
    self.coordinatorComponents.navigationController.pushViewController(viewController, animated: animated)
  }
  func present(childCoordinator: Coordinator, animated: Bool = true) {
    let childVC = childCoordinator.coordinatorComponents.navigationController
    self.coordinatorComponents.navigationController.present(childVC, animated: animated) { [weak self] in
      self?.coordinatorComponents.childCoordinator = childCoordinator // retain the child strongly
    }
  }
  func pop(animated: Bool = true) {
    let privateAPI = self.coordinatorComponents
    if privateAPI.childCoordinator != nil {
      privateAPI.navigationController.dismiss(animated: animated) { [weak privateAPI] in
        privateAPI?.childCoordinator = nil
      }
    } else {
      privateAPI.navigationController.popViewController(animated: animated)
    }
  }
}
```

And the conforming type `MainCoordinator` would now:

* Only have to declare a single `let coordinatorComponents = CoordinatorComponents()` property, not having to know what's inside that `CoordinatorComponents` type (hiding the implementation details)
* Wouldn't be able to access any of the `coordinatorComponents` properties from its `MainCoordinator.swift` file, as they are declared `fileprivate`

```swift
public class MainCoordinator: Coordinator {
  let coordinatorComponents = CoordinatorComponents()

  func showTutorialPage1() {
    let vc = makeTutorialPage(1, coordinator: self)
    self.push(viewController: vc)
  }
  func showTutorialPage2() {
    let vc = makeTutorialPage(2, coordinator: self)
    self.push(viewController: vc)
  }

  private func makeTutorialPage(_ num: Int, coordinator: Coordinator) -> UIViewController { … }
}
```

Sure, you still need to declare `let coordinatorComponents` in the conforming type to provide the storage, and this declaration has to be visible (can't be made `private`) as it's part of the requirement for conforming to `protocol Coordinator`. But:

* that's only one property to declare instead of 2 (or more in more complex cases)
* and more importantly, even if it's accessible from the conforming type's implementation, and also from the outside interface, you can't do anything with it.

Sure you can still access `myMainCoordinator.coordinatorComponents` but you won't be able to do anything with it as all its properties are `fileprivate`!

### Conclusion

Swift might not provide all the features you want right outside of the box. One might hope that some day `protocols` would be able to allow declaring access control attributes to the properties and function requirements they declare, or some way to make those hidden to the public API.

But in the meantime, having those kind of tricks and workaround up your sleeve can make your public API nicer and more safe, avoiding to leak implementation details or to give access to properties that shouldn't be modified outside of the implementation while still using the [Mixin pattern](/swift/protocol/2015/11/08/mixins-over-inheritance/) and providing default implementations.
