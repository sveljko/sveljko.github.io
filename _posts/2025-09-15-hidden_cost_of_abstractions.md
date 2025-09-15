---
layout: post
title: 🔎 Hidden cost of abstractions
lang: en
---

Abstractions are often good and even neccessary. But,
it's easy to misjudge their costs. Even "zero cost abstractions" such as C++ advertises have hidden costs. Let's explore one involving multipe inheritance and `null` pointers.

## Recap: Multiple inheritance

Like many early OO languages, C++ allows to essentially
unconstrained inheritance from more than one base class.

```c++
class Animal {
public: virtual std::string name() const;
};

class Bipedal {
public: virtual bool can_jump() const;
};

class Ape : public Animal, public Bipedal {
public:
    std::string() name() { return "Ape"; }
    bool can_jump() { return false; }
};
```

Now, there might be issues if the base classes are not
distinct, but we're not interested in them here.


## Recap: null pointers

C++ allows pointers to objects to be `null`, that is,
they can point to "nothing", or, in other words, 
_optionally_ point to an object. There are many issues
with this, including that one can do pointer arithmetic
on them and get weird results. Again, we're just interested
in their interaction with multiple inheritance.

The `null` pointer is spelled `nullptr` in C++11 and above,
but it's value is an actual `0` (which is how it was
spelled before C++11).


## Let's get to it

We're gonna provide an example right away:

```c++
int process(Bipedal* p) { ... }

int interesting(Ape* ape) {
    return process(p);
}
```

The function `interesting` receives a pointer to an
object of class `Ape`, which inherits `Animal` and
`Bipedal`. All it ever does is forward its argument to
the `do_stuff_with` which accepts a pointer to `Bipedal`,
which is OK because of said multiple inheritance.

This should be very fast, right? It just pases the pointer
along. C++ has zero-cost abstractions, doesn't it?

## What's wrong with this... inheritance

So how does multiple inheritance work? There are several
ways to do, but the simplest is to just put the base class'
members "one by one", like:

```c++
class Ape_Impl { // sketch, not real C++ code
    Animal base_Animal;
    Bipedal base_Bipedal;
};
```

So, if we want to pass the pointer to `Bipedal`, we can't
pass the pointer as received. We have to adjust it, like:

```c++
int interesting(Ape* ape) { // still sketch
    return process(&ape->base_Bipedal);
               // (ape + offsetof(Ape, Bipedal)) 
}
```

OK, so, there's a little hidden cost, but one would expect
as much. Something more sinister is lurking... 

What if we pass the `null` pointer?

```c++
   interesting(nullptr);
```

If `interesting` would work as sketched above, then it
would pass `0 + offsetof(Ape, Bipedal)`, which is `4`
or `8` or something else, depending on your environment,
but it surely ain't `0`.

To "preserve the nullness`, what the compiler needs to do
is much more involved:

```c++
int interesting(Ape* ape) { // sketch even still
    return process((nullptr == ape) ? null : &ape->base_Bipedal);
}
```

Yup, it introduces _a branch_. This isn't just some
addition, which is most probably negligible cost even in
constrained systems. This branch, which might be hard to
predict, might be a _significant_ slowdown.

You can't really "escape" this by using a reference,
unless, of course, you do know that this pointer can
never be `null`. Since reference can't formally be 
`null`, or rather, having a null reference is Undefined 
Behavior, then all bets are off and one can't predict what
a compiler would do. Actually, this post was inspired by
a crash that was the consequence of such an attempt.

In this simplified example, this is easy to spot. But
if the class inherits from many classes, even if they
are "interface classes" (no data members), and if the
`interesting` is a template, it's _hard_ to see. 

While we analyzed a C++ feature set, no system is free
of similar things, this was just an example from practice.


## Moral of the story

When designing, think about hidden costs of abstractions
you introduce, maybe they're "bad" enough to warrant an 
alternative approach. When using abstractions, be mindful 
of hidden costs, figure out if hidden costs are prohibitive.

Like all engineering, it's a tradeoff, the simplification
that abstraction brings to the design vs the costs thereof.
Just don't forget the hidden costs while "tradeoffing".
