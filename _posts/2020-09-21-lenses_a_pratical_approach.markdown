---
layout: post
title:  "Lenses in a nutshell"
date:   2020-09-21 09:47:32 +0200
categories: jekyll update
---

# Table of Contents
* [Introduction](#introduction)
* [Composing Lenses](#composing-lenses)
    * [operator `>>>`](#operator)
* [Over](#over)
* [Zip](#zip)
* [Why Lenses?](#why-lenses)
* [Final considerations](#final-considerations)


<!--https://medium.com/javascript-scene/lenses-b85976cb0534-->

## Introduction

Most of the code from this post is available in a Swift [Playground](https://github.com/jrBordet/Lenses---an-introduction), and I strongly recommend to follow it while reading the post.

A Lens is basically a container with a getter and setter functions which focus on a part inside a whole.

Considering this, let me define a new `container` over a generic Whole (A) and a Part (B)

{% highlight swift %}

/**
Lens
- parameter A: a Whole.
- parameter B: a Part.
*/

public struct Lens <A, B> {
    let get: (A) -> B
    let set: (B, A) -> A
}
{% endhighlight %}

### getter

The getter takes a whole and returns the part of the object that the lens is focused on.

`get: (Whole) -> Part`

### setter

The setter takes a whole and a value to set the part to and returns a new whole with the part updated.

`set: (Part, Whole) -> Whole`

## Lens Laws

Every `Lens<Whole, Part>` should satisfy these three rules:

{% highlight swift %}
1. Lens.get(Lens.set(Part_a, Whole)) == Part_a
{% endhighlight %}

{% highlight swift %}
2. Lens.set(Lens.get(Part_a), Whole) == Whole
{% endhighlight %}

{% highlight swift %}
3. Lens.set(Vb, Lens.set(Part_a, Whole)) == Lens.set(Part_b, Whole)
{% endhighlight %}

1. set a `part` followed by a get always returns the `part` previously set

2. getting a `part` from the `whole` and setting it again returns the same container

3. applying N consecutive set functions on a `whole` is equal to set the last value on the initial `whole`

## Lens implementation

Let's start with a pratical example. Suppose that we have a model composed by a User and related Address.

{% highlight swift %}
struct User {
    let name: String
    let address: Address
}

struct Address {
    let street: String
    let city: String
}
{% endhighlight %}

Imagine that we want to write a `Lens` to focus on the name of the user.

{% highlight swift %}
Lens<User, String>(
    get: (User) -> String, // Give me a User and I will return the name
    set: (String, User) -> User // Give me a User and a name and I will return to you a brand new User
)
{% endhighlight %}

Our `Lens<User, String>` is self written.

{% highlight swift %}
let lensPersonName = Lens<User, String>(
    get: { person in
        person.name
}, set: { name, person in
    User(
        name: name,
        address: person.address
    )
})
{% endhighlight %}

Usage example

{% highlight swift %}
let name = lensPersonName.get(.one)
// Me

let newUser = lensPersonName.set("mini Me", .one)
  // name: "mini Me"
  // address:
    // street: "Street 01"
    // city: "NY"
    // building: nil
{% endhighlight %}

As you can see we have a new User based on `User.one` with just the field `name` updated, pretty cool.

It's really easy demonstrate the validity of the first and second law.

{% highlight swift %}
LensLaw<User, String>.getSet(lensUsernName, User.me, "name")
// true
{% endhighlight %}

{% highlight swift %}
LensLaw<User, String>.setGet(lensUsernName, User.me, "name")
// true
{% endhighlight %}

But... what about the third law? 

> applying N consecutive set functions on a `whole` is equal to set the last value on the initial `whole`

This means that we need to consider those two lenses:

`Lens<User, Address>` and `Lens<Address, String>`

{% highlight swift %}
func lensUserStreet(_ lhs: Lens<User, Address>, _ rhs: Lens<Address, String>) -> Lens<User, String> {
    Lens<User, String>(
        get: { (u: User) -> String in
            let address = lhs.get(u)
            let street = rhs.get(address)
            
            return street
    }, set: { (street: String, user: User) -> User in
        let newUser = lhs.set(rhs.set(street, lhs.get(user)), user)
        
        return newUser
    })
}
{% endhighlight %}

Considering the shape of the function we have written we are able to see a kind of pattern.

 {% highlight swift %}
(lhs: Lens<User, Address>, // Lens<A, B>
rhs: Lens<Address, String>) // Lens<B, C>
-> Lens<User, String> // Lens<A, C>
{% endhighlight %}

### Composing Lenses

Lets' start write our generic `compose` function just changing `User` with `A`, and `Address` with `B` and `String` with a generic part `C`
 
{% highlight swift %}
public func compose<A, B, C>(_ lhs: Lens<A, B>, _ rhs: Lens<B, C>) -> Lens<A, C> {
    Lens<A, C>(
        get: { a -> C in
            rhs.get(lhs.get(a))
    }, set: { c, a -> A in
        lhs.set(rhs.set(c, lhs.get(a)), a)
    })
}
{% endhighlight %}

Basically the idea behind a `compose` function is that apply `Lens(A, B)` & `Lens(B, C)` is equal to apply `Lens(A, C)`, and with `compose`we can create a brand new `lens` just composing them.

{% highlight swift %}
let lensUserCity = compose(lensUserAddress, lensAddressCity) // Lens<User, String>

lensUserCity.get(.one) // a String
lensUserCity.set("new york city", .one) // a new User
{% endhighlight %}

... and the third law could be validated.


{% highlight swift %}
LensLaw<User, String>.setSet(lensUserAddress, lensAddressStreet, User.me, "street check")
// true
{% endhighlight %}

### operator `>>>`

Usually the compose function is associated with an infix operator to increase readability.

{% highlight swift %}
infix operator >>>

public func >>> <A, B, C>(_ lhs: Lens<A, B>, _ rhs: Lens<B, C>) -> Lens<A, C> {
    compose(lhs, rhs)
}
{% endhighlight %}

{% highlight swift %}
compose(lensUserAddress, lensAddressCity)

lensUserAddress >>> lensAddressCity
{% endhighlight %}


## Over

For the next step we should ask to our selves: it is possible to apply a function from `a` to `b` in the context of any data type? We've already demonstrated that data type mapping is composition and similarly we can apply a function to the value of focus in a lens. Typically, that value would be of the same type, so it would be a function from `a` to `a`.

The lens map operation is commonly called `over`.

We can add to our Lens<A, B> a new function with a shape like that

{% highlight swift %}
public func over(_ f: @escaping (B) -> B) -> ((A) -> A) {
    return { a in
        let b = self.get(a)
        let transformedB = f(b)
        
        return self.set(transformedB, a)
    }
}
{% endhighlight %}

We can read the fuction `over` as: if you give me a way to transform a part, I will give you back a new whole with the part transformed.

And a pratical example is a lens that focus on a City of a given User and return a new User with the field city tranformed by a capitalization.

{% highlight swift %}

let lensUserCityCapitalized = lensUserCity.over { $0.capitalized }

lensUserCityCapitalized(.one) // User

//- name: "Me"
//▿ address:
//  - street: "Street 01"
//  - city: "Ny"
//  - building: nil
{% endhighlight %}

## Zip

The zip function mixes together two lenses with the same `whole`, so we can apply both at the same time by getting/setting two `parts` together.

{% highlight swift %}
/// - Parameters:
///   - lhs: a Lens<Whole, Part>
///   - rhs: a Lens<Whole, AnotherPart>
/// - Returns: a Lens<Whole, (Part, AnotherPart)>
public func zip<A, B, C>(
    _ lhs: Lens<A, B>,
    _ rhs: Lens<A, C>
) -> Lens<A, (B, C)> {
    Lens<A, (B, C)>(
        get: {
            (lhs.get($0), rhs.get($0))
    }, set: { (parts, whole) -> A in
        let partialWhole = lhs.set(parts.0, whole)
        return rhs.set(parts.1, partialWhole)
    })
}
{% endhighlight %}

An usage example

{% highlight swift %}
let update = (
    name: "new Me name",
    address: Address(street: "Some street", city: "Turin", building: nil)
)

let newMe = zip(lensUsernName, lensUserAddress).set(update, User.me)

let zipAndOver = zip(lensUsernName, lensUserAddress).over { (name: String, address: Address) -> (String, Address) in
    (
        name.lowercased() + " 😀",
        Address(
            street: address.street.capitalized,
            city: address.city.uppercased(),
            building: nil
        )
    )
}
{% endhighlight %}

And consider how much is easy to go deeper creating a `zip` focusing on n-values

{% highlight swift %}
public func zip2<A, B, C, D>(
    _ first: Lens<A, B>,
    _ second: Lens<A, C>,
    _ third: Lens<A, D>
) -> Lens<A, (B, (C, D))> {
    Lens<A, (B, (C, D))>(
        get: { whole -> (B, (C, D)) in
            zip(first, zip(second,third)).get(whole)
    }, set: { (parts, whole) -> A in
        zip(first, zip(second, third).set(parts, whole)
    })
}

public func zip3<A, B, C, D, E>(
    _ first: Lens<A, B>,
    _ second: Lens<A, C>,
    _ third: Lens<A, D>,
    _ fourth: Lens<A, E>
) -> Lens<A, (B, (C, (D, E)))> {
    Lens<A, (B, (C, (D, E)))>(
        get: { whole -> (B, (C, (D, E))) in
            zip(first, zip2(second, third, fourth)).get(whole)
    }, set: { (parts, whole) -> A in
        zip(first, zip2(second, third, fourth)).set(parts, whole)
    })
}
{% endhighlight %}

## Why Lenses?

<!-- https://sidburn.github.io/blog/2016/03/14/immutability-and-pure-functions -->

The use of Lenses is tightly connected with __state immutability__ concept.

In Functional Programming we create immutable data types. When we use them we do not change their value or their state directly, instead we create a brand new one. The literal meaning of Immutability is __unable to change__.

Consider that __state shape dependencies__ are a common source of coupling in software engineering. We have always many components that depends on the shape of some shared state, so if we need to later change the shape of that state we have to change logic in multiple places.

Lenses solves this problem allowing us to abstract state shape behind getters and setters.

Instead of littering our codebase with code that dives deep into the shape of a particular object we can create a Lens. If we need later to change the state shape, we can do so in the Lens, and none of the code that depends on the Lens will need to change.

> a small change in requirements should require only a small change in the system

## Quick recap

* [Composing Lenses](#composing-lenses) (`>>>`) compose function, that composes two lenses with matching part/whole
* [Over](#over) function, that modifies a property of a data structure based on the previous value, returning the new structure
* [Zip](#zip) function, that mixes together two lenses with the same whole part, so we can apply both at the same time, by retrieving/modifying two or more things together

## Final considerations

It's easy to notice that the code for implementing Lenses for our data types is really boilerplate: it has always the same shape. So, we should ask to ourselves if there is a better way to work with Lenses.

And the answer comes with Swift 4 that introduced the new __KeyPath__ type.

#### Functional Setter with KeyPath
{% highlight swift %}
let newBook = .galacticGuideForHitchhikers |> \Book.author.name *~ "Adams Noël"
{% endhighlight %}


#### Functional Getter with KeyPath
{% highlight swift %}

let name = .galacticGuideForHitchhikers |> ^\Book.author.name

// Filter operations
let book = books.filter(by(^\.author.name, "Massimo"))

// Sorting operations
let usersSorted = users.sorted(by: their(^\User.id, >))
{% endhighlight %}

Until a dedicated article you can check this code on [Caprice](https://github.com/jrBordet/Caprice)


<!--
You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

Jekyll requires blog post files to be named according to the following format:

`YEAR-MONTH-DAY-title.MARKUP`

Where `YEAR` is a four-digit number, `MONTH` and `DAY` are both two-digit numbers, and `MARKUP` is the file extension representing the format used in the file. After that, include the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets:

{% highlight swift %}
func incr() -> (Int) -> Int {
    return { i in
        return i + 1
    }
}
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
-->
