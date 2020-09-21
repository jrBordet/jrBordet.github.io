---
layout: post
title:  "Lenses - an introduction"
date:   2020-09-21 09:47:32 +0200
categories: jekyll update
---

<!--https://medium.com/javascript-scene/lenses-b85976cb0534-->

## Introduction

A Lens is basically a container with a getter and setter functions which focus on a field inside a whole. Think about the container as the whole and the field as the part. The getter takes a whole and returns the part of the object that the lens is focused on.

The getter takes a whole and returns the part of the object that the lens is focused on.

`let get: (Whole) -> Part`

The setter takes a whole, and a value to set the part to and returns a new whole with the part updated.

`let set: (Part, Whole) -> Whole`

Considering this let me define our `container` over a generic Whole (A) and a Part (B)

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

let's start with a pratical example, so we have a model composed by a User and related address.

{% highlight swift %}
struct User {
    let name : String
    let address : Address
}

extension User {
    static var one = User(name: "One", address: .one)
}

struct Address {
    let street : String
    let city : String
}

extension Address {
    static var one = Address.init(street: "Street 01", city: "NY")
}
{% endhighlight %}

Imagine that we want to write a `Lens` to focus on the name of the user.

{% highlight swift %}
Lens<User, String>(
    get: (User) -> String, // Give me a User and I will return a String
    set: (String, User) -> User // Give me a User and a part and I will return to you a brand new User
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
let newUser = lensPersonName.set("new user One", .one)
{% endhighlight %}

And we can go deeper inside the user and focus on the entire address

{% highlight swift %}
let lensUserAddress = Lens<User, Address>(
    get: { $0.address},
    set: { User(name: $1.name, address: $0)}
)

let address = lensUserAddress.get(.one)
type(of: address)
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

## Why Lenses?

## Lens Laws

## Composing Lenses

## Over

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
