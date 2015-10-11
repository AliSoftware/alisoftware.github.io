---
layout: post
title: "Thinking in Swift, Epilogue: Monads"
categories: swift
---

In [the last article](/swift/2015/10/11/thinking-in-swift-4/), we played a lot with `map` and `flatMap`, methods on the `Optional` and `Array` types. But what you probably didn't realised is that you were manipulating Monads without knowing. But what is a Monad?

## A word about Monads

In [the last articles](/swift/2015/10/11/thinking-in-swift-4/), we discovered that `map` and `flatMap` are similar on both `Array` and `Optional`. In fact, it's not an isolated case: there are plenty of types which have methods like `map` and `flatMap` with those type of signatures. That pattern is so common that this has a name: it's called a _Monad_.

You may have heard about the terms _Monad_ (and maybe also _Functor_) on the web, and seen all kind of comparisons used to try and explain them. But a lot of those comparisons make it more complicated that it is.

But in fact, monads and fonctors are really not that complicated.  
You already know two types that are both _Functors_ and _Monads_: those are `Array<T>` and `Optional<T>`. 

What makes them a _Monad_? And what makes them a _Functor_? Well that's quite simple actually:

**A functor** is a type that:

* wraps another inner type (like `Array<T>` or `Optional<T>` that wrap some `T`)
* has a method `map` with the signature `(T->U) -> Type<U>`

**A monad** is a type that:

* is a functor (so has an inner type `T` and the `map` method)
* also has a method `flatMap` with the signature `(T->Type<U>) -> Type<U>`

And that's all there is to know for _Monads_ and _Functors_!
**A _Monad_ is simply a type that has a `flatMap` method, and a _Functor_ is simply a type that has a `map` method**. Pretty simple, right?

## All kind of Monads

In practice, those methods can have other names than `map` and `flatMap`. For example, [a `Promise`](http://promisekit.org) is also a Monad, but both the `map` and `flatMap`-like methods are instead called `then`.

Think about it, look closely at the signature of `then` which takes the future return value, process it, and return either a new type, or a new Promise wrapping a new type… And yes, once again the same Monad pattern!

And there is a lot of other types that match the definition of a Monad. Like `Result`, etc.

## Chaining map() & flatMap()

What makes them powerful is generally that you can chain them too. Like you can take an initial `Array<T>`, apply it a `transform` using `map` to get an `Array<U>`, then apply another `transform` by chaining another `map` to transform that `Array<U>` into an `Array<Z>`, etc. That makes your code look like it takes an initial value, makes it pass thru a bunch of processing black boxes, and return the final product, like in a production chain. And that's when you can say you're doing _Functional Programming_!

But in the end, it doesn't really matter how you call them. As long as you realize there are `map` and `flatMap` methods on those types that can be really useful to you to transform one inner type into another, that's all that matter.

## What's next

This article concludes the series "Thinking in Swift". THe idea behind this series was to take an ObjC code and make you realize that writing a similar logic in Swift would require you to change your way of thinking. But fret not! I'll do a lot of other articles presenting Swift niceties in other contexts, even if I don't compare them to ObjC!

Happy Swifting, and…  
<center>![map-everywhere](/assets/map-all-the-things.jpg)</center>
