---
layout: post
title:  "Lenses - an introduction"
date:   2020-09-21 09:47:32 +0200
categories: jekyll update
---

# Table of Contents
* [Introduction](#introduction)
* [Composing Lenses](#composing-lenses)
    * [operator `>>>`](#operator)
* [Over](#over)
* [Why Lenses?](#why-lenses)
* [Zip](#zip)


<!--https://medium.com/javascript-scene/lenses-b85976cb0534-->

## Introduction

Most of the code from this post is available in a Swift [Playground](https://github.com/jrBordet/Lenses---an-introduction), and I strongly recommend to follow the Playground while reading the post.

A Lens is basically a container with a getter and setter functions which focus on a field inside a whole. 

Think about the container as the `whole` and the field as the `part`.

### getter

The getter takes a whole and returns the part of the object that the lens is focused on.

`get: (Whole) -> Part`

### setter

The setter takes a whole, and a value to set the part to and returns a new whole with the part updated.

`set: (Part, Whole) -> Whole`

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

Using our `Lens<User, String>` is self written.

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

And we can go deeper inside the user and focus onto the entire address.

{% highlight swift %}
let lensUserAddress = Lens<User, Address>(
    get: { $0.address},
    set: { User(name: $1.name, address: $0)}
)

let address = lensUserAddress.get(.one)
  // street: "Street 01"
  // city: "NY"
  // building: nil
  {% endhighlight %}

<!--
Why Lenses?
State shape dependencies are a common source of coupling in software. Many components may depend on the shape of some shared state, so if you need to later change the shape of that state, you have to change logic in multiple places.
Lenses allow you to abstract state shape behind getters and setters. Instead of littering your codebase with code that dives deep into the shape of a particular object, import a lens. If you later need to change the state shape, you can do so in the lens, and none of the code that depends on the lens will need to change.
This follows the principle that a small change in requirements should require only a small change in the system.
-->

<!--Think of the object as the whole and the field as the part. The getter takes a whole and returns the part of the object that the lens is focused on.-->

<!--A lens is a composable pair of pure getter and setter functions which focus on a particular field inside an object, and obey a set of axioms known as the lens laws. Think of the object as the whole and the field as the part. The getter takes a whole and returns the part of the object that the lens is focused on.-->

<!--If you google “functional lens” you’re probably going to end up with a definition like “functional getter and setter”: in this context the word “functional” really means “immutable”. There are many advantages in using immutable data models - I’m not going into it for this article - but modifying immutable models (i.e. generating new models with something changed) can be a chore, because the whole model must be reconstructed by taking the previous values where they didn’t change, and setting the new values where they did. But Swift offers particular kinds of types, called “value types”, that have value semantics, which basically means that they have no identity and only represent a “value”, i.e., some kind of “information” that is implicitly immutable: a piece of information cannot change, but new information can be created, rendering the previous obsolete.-->

## Composing Lenses

Consider the problem to create a lens that focus on the Street (field from Adress) on a given User. This means that we need a lens able to traverse the User to a part of it's address.

The problem: are we able to write a lens with a shape as `Lens<User, String>` focusing on the street of the address from a given user? And the answer is yes, and we should do that.

So consider those two lenses
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

Usage

{% highlight swift %}

// Get a new User based on User.me with a new Street
lensUserStreet(lensUserAddress, lensAddressStret).set("street update", .one)
// name: "Me"
// address:
//  street: "street update"
//  city: "NY"
//  building: nil

// Get the street from User.me
lensUserStreet(lensUserAddress, lensAddressStret).get(.me)
// Street 01
{% endhighlight %}

<!-- 
 -->

 Considering the shape of the function we have written, are we able to see a kind of pattern?

 {% highlight swift %}
(lhs: Lens<User, Address>, // Lens<A, B>
rhs: Lens<Address, String>) // Lens<B, C>
-> Lens<User, String> // Lens<A, C>
{% endhighlight %}


<!-- But, is really boilerplate. We should ask to ourselves: is there a way to write a generic function to do this work? -->

So, seems that we have the chance to write a more generic function to compose lenses, let's try to do that.

<!-- Considering that Lenses are functional getter and setter means that are composable like every function. -->
 
Lets' start write our generic `compose` function
 
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

with `compose`we can create a new lens just composing them

{% highlight swift %}
let lensUserCity = compose(lensUserAddress, lensAddressCity) // Lens<User, String>

lensUserCity.get(.one) // String
lensAddressCity.set("new york city", .one) // User
{% endhighlight %}

And `lensUserCity` is a new Lens (aka Lens<User, String>) that focus on a city from a given User.

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

	<!-- func zip<OtherPart, OtherLens: LensType>(_ other: OtherLens) -> Lens<WholeType,(PartType,OtherPart)> where OtherLens.WholeType == WholeType, OtherLens.PartType == OtherPart {
		return Lens<WholeType,(PartType,OtherPart)>(
			get: { (self.get($0),other.get($0)) },
			set: { other.set($0.1,self.set($0.0,$1)) })
	} -->

## Why Lenses?

The use of Lenses is tightly connected with __state immutability__ concept.

In Functional Programming we create immutable data types. When we use them we do not change their value or their state directly, instead we create a brand new one. The literal meaning of Immutability is __unable to change__.

Consider that __state shape dependencies__ are a common source of coupling in software. We have always many components that depends on the shape of some shared state, so if we need to later change the shape of that state we have to change logic in multiple places.

Lenses solves this problem allowing us to abstract state shape behind getters and setters.

Instead of littering our codebase with code that dives deep into the shape of a particular object we can create a Lens. If we need later to change the state shape, we can do so in the Lens, and none of the code that depends on the Lens will need to change.

> a small change in requirements should require only a small change in the system

## Lens Laws

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
