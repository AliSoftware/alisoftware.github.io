---
layout: post
title: Fun with Functions
date: 2015-08-28
categories: swift function operator
---

Today's article is about doing some fun with Swift functions, like functions returning functions, currying and operators on functions.

## Functions basics

Today we'll write functions that takes a `Int` and return a `Bool` telling if it matches a certain criteria. This kind of functions can then be used with `filter` to filter an array of integers.

So let's start with a simple one, a function that tells if a number is positive:

```swift
func isPositive(value: Int) -> Bool {
  return value > 0
}

[-4,-2,0,4,7,-8,3].filter(isPositive)
// returns: [4,7,3]
```

## More generic functions

Now what if we want to have a function that returns true if the number is a multiple of N? Sure we could write one function for `isEven`, one for `isMultipleOf3`, `isMultipleOf4`… but where to end?

Of course, a better solution is to have a function that takes that `N` as a parameter. But then if we write this, `filter` won't like it:

```swift
func isMultiple(multiple: Int, value: Int) -> Bool {
  return value % multiple == 0
}

[-4,-2,0,4,7,-8,3].filter(isMultiple)
// Error: filter expects Int->Bool and we gave it (Int,Int)->Bool instead
[-4,-2,0,4,7,-8,3].filter(isMultiple(3))
// Error: can't call `isMultiple` with only one parameter, it expects two
```

## Functions returning Functions

So what we really want is a way to generate a new `Int -> Bool` function (that will fit what `filter` expects) given a multiplier value. So let's write exactly that: a function that takes a `(multiplier: Int)` and returns… another function, of type `Int -> Bool`.

There is multiple ways to do that, the first being to declare a `Int -> Bool` function _inside_ our `isMultiple` function, and return that inner function:

```swift
func isMultiple(multiplier: Int) -> (Int -> Bool) {
  func multFunctionToReturn(value: Int) -> Bool {
    return value % multiplier == 0
  }
  return multFunctionToReturn
}
```

Here the inner function "captures" the `multiplier` parameter privided by the outer function to generate a taylored function and return it.

But the more common (and compact) way is to use closures instead. In Swift, functions and closures are interchangeable, so let's directly return a closure of type `Int -> Bool`:

```swift
func isMultiple(multiplier: Int) -> (Int -> Bool) {
  return { (value: Int) -> Bool in
    value % multiplier == 0
  }
}
```

## Currying

There is actually a third way to achieve that: using _currying_. [Currying](https://en.wikipedia.org/wiki/Currying) is the idea of transforming a function that takes multiple parameters into a function that takes one parameter and returns another function (which in turn takes the next parameter and return a function… until all parameters have been consumed and it returns the final value). This technique allows you to do partial application of a function, which can be very powerful and useful in some cases.

In Swift, we can very easily change a method with mulitple parameters into a currying method, by replacing commas separating the parameters with a closing parenthesis immediately followed by a reopening one. So let's reuse our very first implementation of `isMultiple` but transform that into a currying function:

```swift
func isMultiple(multiplier: Int)(value: Int) -> Bool {
  return value % multiplier == 0
}
```

This syntax may be a bit disturbing, so I personally prefer explicitly using a function returning a function (like in the previous paragraph), as in the end this currying form is exactly equivalent to this (which I find more explicit):

```swift
func isMultiple(multiplier: Int) -> (value: Int) -> Bool {
  return { value in value % multiplier == 0 }
}
```

But in the end it's up to you to use the syntax you prefer.

## Combining functions

What could be nice next is to combine those filters, to generate a new filter from it. For example, now that we have `isPositive` and `isMultiple`, how could we filter every number that are both positive and even?

Of course we could filter twice, like this:

```swift
[-4,-2,0,4,7,-8,3].filter(isPositive).filter(isMultiple(2))
```

But that's not very efficient since it makes the code loop twice into the array. And moreover, it could also be interesting to have numbers that are _either_ positive or even, and we wouldn't have a nice syntax for that, but only this:

```swift
[-4,-2,0,4,7,-8,3].filter { value in isPositive(value) || isMultiple(2)(value) }
```

Wouldn't it be cool instead to generate a new function by combining two `Int -> Bool` functions? It would indeed feel more natural to write something like this:

```swift
[-4,-2,0,4,7,-8,3].filter(isPositive || isMultiple(2))
```

Well, let's make that possible!

The `&&` and `||` operators already exist, so we just need to override them so they supports taking `Int -> Bool` functions as a parameter:

```swift
func || (lhs: Int->Bool, rhs: Int->Bool) -> (Int->Bool) {
  return { (value: Int) -> Bool in
    return lhs(value) || rhs(value)
  }
}
```

We could also do the same for `&&`. Let's use a more compat syntax for this one, using implicit `$0` parameters, to vary the examples:

```swift
func && (lhs: Int->Bool, rhs: Int->Bool) -> (Int->Bool) {
  return { lhs($0) && rhs($0) }
}
```

What about also having a way to negate a function, by overloading the prefix `!` operator too?

```swift
prefix func ! (f: Int->Bool) -> (Int->Bool) {
  return { value in !f(value) }
}
```

And now we could even write:

```swift
[-4,-2,0,4,7,-8,3].filter( !isPositive || isMultiple(3) )
// return [-4,-2,0,-8,3] (non-positive numbers + multiples of 3)
```

Isn't that beautiful?

One could even declare those shorthand aliases for negative and odd/even filters:

```swift
let isZero = { value in value == 0 }
let isPositiveOrZero = isPositive || isZero
let isNegative = !isPositive && !isZero
let isNegativeOrZero = !isPositive
let isEven = isMultiple(2)
let isOdd = !isEven
```

And this works like expected, even if those are functions, thanks to our overloading above!

## Conclusion

So with those simple tricks, you learned how having functions returning functions can be useful, how we could imagine some operators to combine functions returning `Bool` the same way we would've combined `Bool` directly, and were introduced to the concept of currying.

There is a lot more fun we could have with function. We could have made those overload of `&&`,`||` and `!` generic, so that if would work with any `T->Bool` functions whatever the type of `T` is for example. I could even talk about using currying to its full power, go crazy with Functional Programming and all (but a lot of other articles do that already, so no need for another one here I guess), but I think that'll be enough for today!

Happy Swifting all!
