---
layout: post
title: 🚮 C++ non-virtual dynamic dispatch
lang: en
---

C++ provides dynamic dispatch via virtual functions. Most common
implementation uses "v-tables", with each object having a pointer
to said v-table.  In some cases, this is undesireable. There are
other reasons to avoid virtual functions and here we present a
low-effort way of doing so.

## Recap: Dynamic dispatch, what is it?

Dynamic dispatch is a form of polymorphism.  If we have a pointer
or reference to a base class, we would like to call a "polymorphic"
member function on it and have the function of the actual class
be called.  The usual example is with some pets:

```c++
class Animal {
public:
    virtual void sound() = 0;
};

class Dog : public Animal {
    void sound() override { std::cout << "Aw-aw!\n"; }
};
class Cat : public Animal {
    void sound() override { std::cout << "Meow!\n"; }
};
```

So, if we have an `Animal* p` and call `p->sound()`, what we get
will depend on the actual object "behind the pointer".

### v-table?

The usual way of providing for dynamic dispatch in C++ is for
the compiler to generate a table of function pointers, put
a pointer to said table in each `Animal`-derived class and 
fill it with pointers to the correct virtual functions.

Something like this was done in the `cfront`, the first C++
compiler, which generated C code:

```c
struct Animal_vtable {
    void (*sound)(Animal* this);
};

void Dog_sound(Animal* this) { puts("Aw-aw"); }
struct Dog {
    Animal_vtable vtable;
};
Animal_vtable Dog_vtable = { Dog_sound };
void Dog_construct(Dog* p) { p->vtable = Dog_vtable; }

struct Cat {
    Animal_vtable vtable;
};
void Cat_sound(Animal* this) { puts("Meow"); }
Animal_vtable Cat_vtable = { Cat_sound };
void Cat_construct(Cat* p) { p->vtable = Cat_vtable; }
```


### Why avoid virtual functions?

The fundamental problem lies with v-tables. They occupy some
memory in _each_ object, which can be a lot, especially in
embedded or otherwise constrained systems.

In context of multiple inheritance, v-tables pose even greater
issues, and that's why you usually hear that you should avoid
multiple inheritance in C++ and why C++ inspired OO languages,
like Java and C# dismiss multiple inheritance altogether.

There are also some language-level issues, like knowing when a
functions was overrided or not (especially in C++98, when there was no
`override` keyword), made more comples because of function
overloading.

Finally, v-tables are slow. Here's what goes on:

```c
void Animal_sound(Animal* this) {
    this->vtable.sound(this);
}
```

No, I'm not stuttering, there are two `this`-es there. It took C++
compilers more than 30 years to start eliminating these slow calls,
and they are still not doing a great job (but do waste significant
time and (electric) energy trying).


### Why not do it like in the C examples above?

Because it's tedious and error prone.  Yes, some C frameworks actually
do something like this and some C++ "ports" of such frameworks do, to.

But, having done similar things myself a few times, I must say that,
for more than a few small and closely-held classes, this is not worth
it.

## Motivating example

Say you have an embedded real-time system that you'd like to debug.
Since it handles a lot of events in small time, you can't use a 
debugger and you can't save the events in a log file to examine.

You save the events in memory, in some kind of buffer and, when the
test has run its course, transfer this buffer to a computer for study
(say, via UART or Ethernet).

Now, this is a constrained system, not much memory there.  So, you
don't hold strings, but several different classes for different events
that you want to analyze.  Saving them, converting them to CSV or JSON
or whatever, is, of course, different for each.  But, having a v-table
in each log record (object) is overkill and you need to save some ID
of the record type anyway (for offline analysis). These are usually
called "typecodes".

## On with the show

What we want is a way of doing not-much-more than what we do in
"regular" C++. Adding a declaration of a virtual function in base C++
class and defining the overriding functions in derived classes. We
do expect to do more than that, of course, but, not much more.

Foremost, we want to avoid having to "manually" dispatch based on
typecodes and having to manually call some "registration" function to
register our class/virtual function. This is _very_ error prone.

The tricks we wil use are `constexpr` and `CRTP` (Curriously Recurring
Template Pattern).

Let's start with typecodes. Let's use an enum:

```c++
enum log_event {
    external_ev_1,
	external_ev_2,
	internal_ev_1,
	internal_ev_2,
	max_log_event
};
```

Since they are easy to add and it's useful to have them in one place
(where you can actually assing values to the symbols, if you wish), we
will not try to avoid typecodes, even though there are some
borderline- valid ways to do that.

Then we have the "virtual function declaration":

```c++
using fn_print = void (*)(base const&, char const*);
static fn_print a_print[max_log_event];
```

Yes, it's a little more verbose than in plain C++, but, you only do it
once per virtual function.

Unfortunately, we can't have "pure" abstract functions, we need to
implement the dynamic dispatch in the base class:

```c++
struct base {
    uint16_t              diff_us : 13;
    const log_event       ev      : 3;

    void print(char const* s) const { a_print[ev](*this, s); }

    base(log_event e) : ev(e) {}
};

```

You can see that we have another member, `diff_us`, the difference, in
microseconds, since the previous log entry, and we are short on
memory, so we are packing with bit-fields.

Also, we disallow changing of the `ev` typecode after construction,
giving a similar level of protection as with v-tables.

That's for the intro, now comes the "main event", the helper, CRTP
class to be used by all log entry classes:

```c++
template <class T, event_log evl> struct base_crtp : base {
    static const event_log evl_id_;
    static void call_print(base const& b, char const* s)
    {
        static_cast<T const&>(o).print(s);
    }
    static constexpr dog register()
    {
        return a_print[evl] = call_print, evl;
    }
    base_crtp() : base(evl_id_) {}
};
template <class T, event_log evl>
const dog base_crtp<T, evl>::evl_id_ = base_crtp<T, evl>::register();
```

This CRTP class is used to "automagically":

* set-up the "virtual function pointer entry"
* set the type-code

As we can see, the `constexpr` function `register` is used from a
definition of the static constant data member `evl_id_`, which means
it will not occupy any memory.  It will, as a side-effect initialize
the virtual function point entry. A good optimizing compiler will
actually just put the `call_print<T>` in the `a_print[evl]` memory,
w/out any generated code.

A more recent C++ compiler will allow for avoiding the comma operator
in the `constexpr` function, making for nicer-looking code.

The thing is, now there's very little work to do to implement the
derived class:

```c++
struct log_ext_ev_1 : base_crtp<log_ext_ev_1, external_ev_1> {
    void print(char const* s) {
	   // whatever is appropriate for `log_ext_ev_1`
	}
	/// whatever else you need here...
};
```

So, this is just a little more than C++ virtual functions.

### Benefits

This should actually be faster than v-tables. The "dispatch"
function in `base` is inlined, so, this is just a call to a
function from a static array, which is fater than vtable,
which is in a pointer in the object - both not known at
build time.

The `call_XXX<>` should also be inlined, because the CRTP is
instantiated when the class is defined, so, it will "see" the `XXX`
virtual function in the derived class.

Other than that, you can have the typecodes "in whatever form you like
them", which can save significant memory.

Also, this does not influence your destructors in any way.  Whether
you choose to have a virtual destructor or not is not influenced by
having one or more these "CRTP induced" non-virtual dynamically
dispatched functions.

Once the groundwork is done, this is not much more work than adding a
new virtual function and is scarecely anu more work than adding a new
class.

### Issues

You may make a typo and give the typecode for another derived
class as the argument to the CRTP class. This can be caught
with an assert in the `constexpr register` function, which
would check if the entry in the dispatch table is already filled.

Also the dispatch function in `base` could `assert` that the entry for
the typecode is not empty (`nullptr`) before calling it.

### Some other uses

Your typecodes need not be integers. Could be strings, for example. Of
course, then you won't be able to (ab)use arrays, but some
`unordered_map<>` or some such thing, which will be slower, but, you
might not care about speed.

You might already have some typecodes.  For example, you might be
handling some protocol, where the messages have their IDs which
are carried in the message contents.  Handling them, (pretty)
printing them, and such, can then be done on the messages "themselves",
provided that you can make your compiler align the `struct message`
correctly.

You might have typecodes in some database, or some (configuration)
files. 

Also, one class "hierarchy" can do some stuff for a typecode, while
other(s) can do other stuff, but all will be linked via the typecode,
while no such connection can exist with virtual functions, whereas one
hierarchy has nothing to do with the other (lest you combine them via
multiple inheritance, which you most probably _don't_ want to do).

In such cases, it's actually useful that there is a "traceable" line
from the already existing typecode to your class that implements some
functions pertaing to said typecode.

### The compile-time counter

There are several implementations of compile time counters in C++ on
the Internet. Some use obsolete language features, some have big
limitations, or other issues, some are hard to figure out if they are
legal.  In general, one should probably steer clear away from such
things.

If you _really_ don't care about the values of the typecodes, you
can always do some preprocessing of your own. For example, use
something like:

```c++
struct log_ext_ev_1 : base_crtp<log_ext_ev_1, @@NEXT_LOG_EVENT> {
```

Then have a simple script that would go through the source code and
replace each `@@NEXT_LOG_EVENT` with subsequent numbers, and then
also replace `@@MAX_LOG_EVENT`:

```c++
static fn_print a_print[@@MAX_LOG_EVENT];
```

Also, in some future version of the C++ standard, there might be some
provisions for making something like a compile-time counter,
especially with compile-time reflection and code generation
(insertion).


## Moral of the story

Just because a programming language (C++ in this case) provides a
facility to achieve something, doesn't mean you _have_ to use said
facility. You can achieve that in other ways, if you need other
characteristics of the solution.  Just don't make it too hard or
fragile, 'cause then it probably won't be worth it.
