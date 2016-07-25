---
layout: post
title: "Closures Capturing Semantics, Part 2: Reference Cycles"
categories: swift closures
published: false
---

## Reminder about weak & strong

_Note: If you know about weak & strong references & retain cycles already, you can skip that part and jump directly to the nextparagraph._

### Strong references & ownership

When an object `A` has a property of type `B`, we sometimes say that `A` "owns" the object `B` it refers to. That means that as long as the instance of `A` lives in memory, it keeps a hold on that property of `B` it contains, so that this `B` stays around while `A` is alive.

**That's what we call a `strong` reference**, when `A` strongly reference `B` and owns it and keeps it alive as long as it is itself alive. You could see it like a leash between a dog owner and the dog, as long as the owner keeps the dog in leash the dog won't run away, and as soon as the owner is released from memory nobody holds that leash anymore so to dog is released too.

Obviously, that concept only makes sense with reference types, a.k.a classes instances, because by definition, if `B` were a value type (like `Int` or any other `struct`) then each `A` instance will have its own copy of that `B`. But if `B` is a reference type (a `class`) then `A` only holds a _reference_ to that object `B` and someone must keep that object `B` alive.

_Multiple objects could have a strong reference to the same object `B`. In that case, `B` will be kept in memory as long as at least one object has a strong reference to it, and will be released only when nobody retains it anymore._

### Reference cycles

The problem arises when `A` has a strong reference on `B`… but `B` also has a strong reference on `A`. In that scenario, everybody retains everybody and nobody will be released. That's like if the dog _also_ held a leash in its muzzle getting hold of it's owner, the owner retaining the dog but the dog also retaining the owner. The dog won't be released until the owner is released, and the owner won't be released until the dog is released…

They are both retaining each other, that's what we call a reference cycle or retain cycle. And that means that both those objects will never be released from memory so you'll have a memory leak.

### Weak references

To solve that problem, we can mark properties and other references as `weak`. Typically, one object `A` (usually the one that can be seen as "the parent" or "the owner") has a _strong_ reference to the other object `B`, but if `B` also needs a reference to `A` we will make it a `weak` reference, to prevent the reference cycle. A `weak` reference means that `B` will only "point at" `A`, but won't retain it Like if the dog looked at its owner and could tell the owner's name, but won't retain the owner with any kind of leash.

To create a weak reference, symply mark the reference with the `weak` keyword.

Because `B` doesn't retain its reference to `A` in a `weak` reference, that means that `A` can disappear / be released while `B` would still be alive. That means that `weak` references can only be `Optional`, because if object `A` being referenced is later released, that weak reference will automatically turn to `nil`. That weak reference to `A` must also be a `var` and not a `let` as it can obviously change at any time when `A` is released from memory and that reference changes to `nil`.

Here's an example(^pointer-equality):

```swift
class Person: CustomDebugStringConvertible {
  let name: String
  let dog: Dog
  init(name: String, dog: Dog) {
    self.name = name
    self.dog = dog
    dog.owner = self
  }
  var debugDescription: String { return "<Person \(name)>" }
  deinit { print("\(self) deinit") }
}

class Dog: CustomDebugStringConvertible {
  let name: String
  weak var owner: Person? = nil
  init(name: String) {
    self.name = name
  }
  var debugDescription: String { return "<Dog \(name)>" }
  deinit { print("\(self) deinit") }
}

func demo1() {
  let dog = Dog(name: "Droopy") // dog has no owner yet
  do {
    // We create the person, mark it as owner of the dog
    let person = Person(name: "Bob", dog: dog)
    person.dog === dog // true
    dog.owner === person // true
    // then we exit the scope where the person object was created
  }
  // so the "person" variable no longer exists and the Person object
  // isn't strongly referenced by anybody anymore (the dog only has
  // a *weak* reference on it, not a strong one), so the Person has
  // been released from memory there
  dog.owner === nil // true, the owner was released from memory
  // so the dog doesn't have an owner anymore.
  // dog.owner was a weak reference, so that reference didn't prevent
  // the Person to be released from memory
}
// Then we exit the function and the dog variable doesn't exist anymore
// and the dog instance is being released from memory too
```

(^pointer-equality): Note that the `===` operator tests for reference equality, and is only true if the objects are the exact same instances — it's not just two instances happening to have the same values for all their fields (that would be `==` and would need the objects to be `Equatable`) but really reference/pointers equality.

When you execute that code above, both `deinit` methods will be called, proving that there is no reference cycles thanks to one of the reference being `weak`. 

If you removed the `weak` keyword from the `Dog`'s `var owner` property declaration, neither `deinit` will be called because each will retain the other and you'll get a reference cycle. And that assertion `dog.owner === nil` even outside the `do { }` block declaring the owner… will no longer be true, further proving the memory leak.


## weak captures

`[weak x] in …`

Particular case of `[weak self] in …`

## unowned captures

`[unowned x] in …` == `[weak x] in` + `x!`

- Might be killing ponies, except when both the object owning the closure and the captured variable are guaranteed to always be alive at the same time (like a UIBarButtonItem with a closure, retained by the UIViewController that is captured by the closure).
- One might still prefer `[weak x] in` followed by `guard xx = x else { return }`
- Link to the SE ML / proposal about a new `[guard x]` capture semantics that got rejected.

## Demo with a class

Instead of using `delay()` top-level function, now create a `class Pokeball` that contains a `let pokemons: [Pokemon]` and some `let execute: (Pokmon) -> String` or similar closure property, to demonstrate that the capture can last as long as the closure itself is alive and retained.

## Nested closures' capture semantic

Explain the case of nested closures

```swift
self.execute() { [weak self] in
  self?.execute() {
    print("self") // weak or strong here?
  }
}
```

## var capture lists

`[var x = y] in …`

## multiple captures in a capture list

`[a = x, b = y, c = z] in …`


## NoEscape

* Talk about `@noescape` and how it doesn't capture `self`.
* Link to my other article about "Being Lazy" and `lazy var x: T = { … }()` being auto-escaped implicitly
* Also talk about `@autoclosure`?

### Links

Here are some other interesting articles on the subject:

* ["weak, strong, unowned, oh my!" - a guide to references in Swift](http://krakendev.io/blog/weak-and-unowned-references-in-swift) post by [KrakenDev](https://twitter.com/allonsykraken)
* ["The weak, the Strong, and th Unowned" — Memory Management in Swift](https://realm.io/news/hector-matos-memory-management/) — talk by the same Kraken, a.k.a Hector Matos
