---
layout: post
title:  "So… what’s a Monad?"
date:   2020-09-21 09:47:32 +0200
categories: jekyll update
---

# Table of Contents
* [Introduction](#introduction)
* [Functors](#functors)
    * [Map](#map)
* [Applicative](#applicative)
* [Monads](#monads)
* [Conclusion](#conclusion)

<!--https://medium.com/javascript-scene/lenses-b85976cb0534-->

## Introduction

Most of the code from this post is available in a [Swift Package Manager](https://github.com/jrBordet/WhatsMonoids.git), and I strongly recommend to follow it while reading the post.

__spoiler__: in Swift we already have Monoids: `Result`, `Array` and `Optional` are all __monads__.

But...before describe what's a Monad is we should define some other types, like __functors__ and __applicative__. 

Basically keep in mind that the `M` world help us to transform `data`.

So let's start considering a simple value: `Int` and apply a function to this value:

{% highlight swift %}

func incr(_ i: Int) -> Int {
    i + 1
}

incr(1) // 2

{% endhighlight %}

Simple enough. Lets extend this by saying that any value can be in a context. For now you can think of a context as a `Container` where you can put a value in:

{% highlight swift %}

public enum Maybe<A> {
    case none
    case value(A)
}
    
{% endhighlight %}

Now when you apply a function to this value you will get different results depending on the context. 

This is the idea that Functors, Applicatives, Monads etc are all based on.

The `Maybe` type defines two related contexts:

* none
* value

In a second we will see how function application is different when something is a `.value(A)` versus a `.error`. 

{% highlight swift %}
let Maybe = Maybe<Int>(1)
        
guard case let .value(value) = Maybe else {
    fatalError()
}
        
XCTAssertEqual(value, 1)

let e = Maybe<Int>.error
    
{% endhighlight %}

First let’s talk about Functors!


# Functors
When a value is wrapped in a context you cannot simply apply a normal function to it as we have done before. 

We need an extra function, a `map`:
{% highlight swift %}
func map<B>(
    _ f: @escaping (A) -> B
    ) -> Maybe<B> {
    guard case let .value(t) = self else {
        return .none
    }
        
    return .value(f(t))
}
{% endhighlight %}

`map` have a really clear defined shape

`f: (A) -> B` is A function that describe how transform B from a given A

`-> Maybe<B>` return the same Container
    
And now we can use our brand new map to apply a function to `Maybe` and trasnform safely is content.

{% highlight swift %}
let result = Maybe<Int>(1).map(plusThree(_:))

guard case let Maybe.value(v) = result else {
    fatalError()
}
        
XCTAssertEqual(v, 4)

{% endhighlight %}

or with a simple syntax using Swift's autoclosure:

{% highlight swift %}
Maybe<Int>(1).map { plusThree($0) } // 4
{% endhighlight %}

So `map` shows us how it’s done! But how does `map` know how to apply the function? Let's go deep inside of the meaning of `map` function.

## map

> In many programming languages map is the name of a higher-order function that applies a given function to each element of a functor, e.g. a list, returning a list of results in the same order. It is often called apply-to-all when considered in functional form.

A generic `map` function should satisfy a couple of laws:
* Identity
* Composition

### identity law

> map(id) == id 

If the transformation we supply to map doesn’t do anything, then it leaves the structure fixed. Seems like a very reasonable property for map to have.

There’s a name for a function that does nothing but return its argument: the identity function. We can define it in full generality like so:

{% highlight swift %}
public func id<A>(_ a: A) -> A {
    return a
}
{% endhighlight %}

And let's me verify if our `map` satisfy the first law.

{% highlight swift %}
let Maybe = Maybe<Int>(1)

let result = Maybe.map(id)

XCTAssertEqual(result, Maybe)
{% endhighlight %}

And that's it!
    
### composition law or omptimisations

The composition law ensures that `map (f) >>> map (g) or map (f >>> g)` lead to the same result, or in other words that the composition of the `maps` is equal of the `map` of `composition`.

To demonstrate the second law we need a new function `pipe` to compose two functions.
 
{% highlight swift %}
public func pipe<A, B, C>(
    _ f: @escaping (A) -> B, 
    _ g: @escaping (B) -> C
    ) -> (A) -> C {
    return { a in
        g(f(a))
    }
}
{% endhighlight %}

Basically the idea behind a `pipe` function is that applying a function `f: (A) -> B` and `g: (B) -> C` is equal to apply just `z: (A) -> C`.

Define two simple functions to compute an `Int`

{% highlight swift %}
let incr: (Int) -> Int = { v in v + 1 }
let square: (Int) -> Int = { v in v * v }
{% endhighlight %}

Compose them with `pipe`

{% highlight swift %}
let incr_square = pipe(incr, square)
{% endhighlight %}

The result of composition is the same of chaining

{% highlight swift %}
let result_composition = Maybe
    .map(incr_square)

let result_chaining = Maybe
    .map(incr)
    .map(square)

XCTAssertEqual(result_composition, result_chaining)
{% endhighlight %}

And that's it, the composition of the `maps` is the `map` of `composition`, and the second law is verified.

And with the new `pipe` we can write more complex functionality. 

For semplicity let me define a new operator `<^>` (<$> in Haskell) for the `map` and bring it together with `pipe` composition.

{% highlight swift %}
func <^> <A, B>(f: @escaping (A) -> B, a: Maybe<A>) -> Maybe<B>
{% endhighlight %}

The left side is just our `map` function that take a value `A` and transform it in `B`, and the right side is our `Maybe` over a genric `A`. As I said is just an operator over the `map` function but it give us a great power to combine complex operations toghether.

{% highlight swift %}
let incr: (Int) -> Int = { $0 + 1 }
let square: (Int) -> Int = { $0 * $0 }

let result = pipe(incr, square) <^> Maybe<Int>(3)
// 16 
{% endhighlight %}

This code take a number `3` and increment it by `1` and then `square` the result.

{% highlight swift %}
let fancy = { (i: Int) in "fancy \(i)" }        
let fancy_incr = pipe(incr, fancy) <^> Maybe<Int>(3)
// "fancy 4"
{% endhighlight %}
    
#### Array as Functor

{% highlight swift %}
let transform = pipe({ $0 + 2 }, { $0 + 1 })
 
let result = transform <^> [0, 1, 2, 3] // [0, 1, 2, 3].map(transform)

XCTAssertEqual(result, [3, 4, 5, 6])
{% endhighlight %}

#### Optional as Functor

{% highlight swift %}
let a: Int? = 3

let transform = pipe({ $0 + 2 }, { $0 + 1 })

let result = transform <^> a // a.map(transform)

XCTAssertEqual(result, 6)
{% endhighlight %}

## Just what is a Functor, really?

A Functor is any type that defines how `map` applies to it. Here’s how map works:

1. `map` takes a function as `plusThree(_:)`
2. and a `functor` like `Maybe<Int>(1)`
3. and return a new `functor`

{% highlight swift %}

let result = Maybe<Int>(1).map { plusThree($0) }

// 1. .map(plusThree(_:))
// 2. Maybe<Int>(1) ~> $0
// 3. `result` is a new Maybe<Int>

{% endhighlight %}

## Applicative

With `functors` we have seen how to transform data using a specific function. With an `applicative functor` we map objects in an 'lifted world' using functions from the same 'lifted world'.

With an applicative our values are wrapped in a context just like `functors`, but our functions are wrapped in a context too: `Maybe<((A) -> B)>`

So an `applicative` know how to apply a _function_ wrapped in a context to a _value_ wrapped in a context:

{% highlight swift %}
func apply<B>(_ f: Maybe<((A) -> B)>) -> Maybe<B> {
    fatalError()
}
{% endhighlight %}
    
Implementation

{% highlight swift %}
switch f {
case .none:
    return .none
case let .value(transform): // (A) -> B
    switch self {
    case .none:
        return .none
    case let .value(a):
        return Maybe<B>(transform(a))
    }
}
{% endhighlight %}

Now pay attention to the first `value` case. 

Does nested switch statement remind you of anything? This code does pretty the same our `map` function does. So let’s refactor it a little bit and replace the nested switch with the `map` function. 

Brilliant!

{% highlight swift %}
func apply<B>(_ f: Maybe<((A) -> B)>) -> Maybe<B> {
    guard case let .value(transform) = f else {
        return .none
    }
    
    return self.map(transform)
}
{% endhighlight %}


Here’s something you can do with `applicatives` that you cannot do with `functors`. 

How do you apply a `function` that takes two arguments to two wrapped `values`?

{% highlight swift %}
func addition(_ a: Int, b: Int) -> Int { a + b }
{% endhighlight %}

We can also define <*> to help us with `apply`

{% highlight swift %}
let a = Maybe(41)
let b = Maybe(1)

let result = curry(addition) <^> a <*> b{% endhighlight %}

To clarify the previous example take a look at the implementation with `optional`. The result is exactly the same. This because our `Maybe` acts as an `optional`.

{% highlight swift %}
let a: Int? = 41
let b: Int? = 1

if let a = a, let b = b {
    let result = addition(a, b: b) // 42    
}
{% endhighlight %}

Look at an example on `Array`. Considering a sequence of `Int` as an `applicative functor` we can define an `apply` function even on it.

{% highlight swift %}
let result =  [ { $0 + 3 }, { $0 * 2 } ] <*> [1, 2, 3]
        
result == [4, 5, 6, 2, 4, 6]
{% endhighlight %}

## Monads

And finally `monads` apply a function that returns a wrapped value to a wrapped value. Monads have a function (>>= in Haskell, pronounced _bind_) to do this.

{% highlight swift %}
func flatMap<B>(
    _ f: @escaping(A) -> Maybe<B>
    ) -> Maybe<B> {
    guard case let .value(a) = self else {
        return .none
    }
    
    switch f(a) {
    case .none:
        return .none
    case let .value(v):
        return Maybe<B>(v)
    }
}
{% endhighlight %}

And we can define an infix operator >>- Here’s how it works:

{% highlight swift %}
Maybe(3) >>- half
// .none
Maybe(4) >>- half
// 2
Maybe.none >>- half
// .none
{% endhighlight %}

You can also chain these calls:

{% highlight swift %}
let result = Maybe(20) >>- half >>- half
// 5
{% endhighlight %}

### Monads's law

All `monads` type should obey the three monad laws:
* Left identity:	
* Right identity:
* Associativity

references at [Haskell - monad laws](https://wiki.haskell.org/Monad_laws)

For brievity we are going to consider just the associativty law, that lead us to define the monad composition operator (aka Kleisli operator)

{% highlight swift %}
func >=><A, B, C>(
    _ lhs: @escaping(A) -> Maybe<B>,
    _ rhs: @escaping(B) -> Maybe<C>
) -> (A) -> Maybe<C> 
{% endhighlight %}
    
The `Kleisli` operator unlock the power to compose `flatMap` in a consistent way.

{% highlight swift %}
let fancy: (Int) -> Maybe<String> = { Maybe.value("fancy \($0)") }

let compute =
    half(_:)
    >=> incr(_:)
    >=> fancy

let result = compute(100) 

// "fancy 51"
{% endhighlight %}

## Conclusion

`Functors`, `Applicative functors` and `Monads` have conquered the world of functional programming by providing general and powerful ways of describing effectful computations using pure functions. 

so, in a nutshell

A `functor` is a type that implements map.

An `applicative` is a type that implements apply.

A `monad` is a type that implements flatMap.

What we have done in this article is to define again the `Optional` type with our `Maybe` just to demonstrate that we can consider `Optional` as a `Monad`