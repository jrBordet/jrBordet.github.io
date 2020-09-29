---
layout: post
title:  "So… what’s a Monad?"
date:   2020-09-21 09:47:32 +0200
categories: jekyll update
---

# Table of Contents
* [Introduction](#introduction)


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
