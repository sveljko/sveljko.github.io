---
layout: post
title: Writing simple code in C
lang: en
---

Just because you're writing in C, doesn't mean you have to write
bad/complex/long code. C is not the enemy, you can write bad code in
_any_ language, and you _can_ write good/simple/short code in C.
We'll show how some 20 lines of C string handling code can be reduced
two one or two lines.

## Everybody says C is bad for string handling

What "everybody" mean is that you have to do a "lot of work" to handle
strings in C, as there is no real string abstraction in C.  There's
just a convention, codified by the standard library, that an array of
`char`s that ends with the `NUL` character (`'\0'`) is a "string".

This is a problem, but, a lot of times, one doesn't need to "jump
through hoops" to handle strings in C. One just needs to take care.

## Let's see the bad code

What follows is code written in a commercial environment, by a
reasonably experienced and skilled developer who knows C. It is a good
representation of the terrible things people do in C, for no
particular reason.

```c
#define EXPECTED_STRING "Whatev"

int read_expected_string(int stream)
{
    char line[1024];
    size_t expected_len = strlen(EXPECTED_STRING);
    
    int count = read(stream, line, sizeof line);
    int line_length = (count < sizeof(line)) ? count : (sizeof(line) - 1); 

    if(line_length < (version_tag_len)) 
    { 
        return FAILURE; 
    } 

    line[line_length] = '\0'; 
    line[version_tag_len] = '\0'; 

    if(!strcmp(line, EXPECTED_STRING)) 
    { 
        return SUCCESS; 
    } 
    return FAILURE; 
}
```

Try to think about what this code wants to acomplish. Don't read
further until you figure it out or give up trying.

There are various issues with this code. Some can be taken care of by
a good optimizing compiler (like using `strlen()` to calculate the
length of a literal string, rather than `sizeof`). Some "genius"
compiler might even infer that the there is no need for
`line[line_length] = '\0'`, because `version_tag_len <= line_length`,
so the next line (`line[version_tag_len] = '\0';`) will cut the string
at least as short.

The name is also not great. With current name, one would be more
inclined to assume that `stream` would be read until the
`EXPECTED_STRING` is finally read. But, that is not the case.

But, even if we fix all of the above (or let (genius) compiler do it),
this is way too complex.

Code actually just wants to read some string from a stream (never mind
what is the stream - is it a file, socket, pipe or some other such
thing) and then compare it to some expected string. That's all.

## What would this look like in your favourite high-level language?

Assuming you have the stream library that works in such a way, you
should be able to get away with:

```ruby
def expect_string(stream)
    stream.read().start_with?(EXPECTED_STRING)
end
```

This is Ruby, but similar things can be done in many other languages.

OK, this obviously doesn't handle errors explicitly, assuming some
exception handling, and it doesn't limit the input length, but, you
might have `stream.read(max_bytes)` also.

The point is that this code is pretty easy to reason about.

## OK, back to C

Let's start by figuring out what is the cause of all this code.
Obviously, the author was in the mind-set that, in C, you have
to take care of your strings manually. Which is, true, of course.
But, you don't have to do it "each step of the way".

In abstract, the code above does this:

    read from stream
    calculate the position of the NUL char (to make a string)
    if read string is too short, return failure
    put NUL char to make the string
    if string is as expected return SUCCESS
    otherwise, return FAILURE

But, see, it doesn't have to do all this.  There's no need to
calculate, as there's no need to actually make the ASCIIZ
string. Comparison in `strcmp()` stops as soon as `NUL` is
encountered - it does _not_ check if the other string has `NUL` at the
same place.

Also, there's another reason this code is bad. It doesn't handle
errors explicitly, it "hides" that behind the "if read string is too
short", because, on error, `read()` will return `-1` and that is,
obviously, too short.  But, such "clever" code is bad, as one needs
to figure out this assumption from the "environment". That is, to
reason about such code, the code itself is not enough. Also, during
some code mutation, it will probably break. For example, change to use
some read()-like function which doesn't return `-1` on error.

So, let's see what we can get:

```c
int expect_string(int stream)
{
    char line[1024];
    int count = read(stream, line, sizeof line / sizeof line[0]);

    if (count < 0) {
        return FAILURE;
    }
    
    if (0 == strcmp(EXPECTED_STRING, line)) {
        return SUCCESS;
    }
    else {
        return FAILURE;
    }
}
```

Also, we now see that we don't care about the length of
`EXPECTED_STRING`, which means we can simply get it as a parameter for
this function, making it more general.  We can use some simple C
coding tricks to make it shorter:

```c
int expect_string(int stream, char const* expected)
{
    char l[1024];

    return ((read(stream, l, sizeof l / sizeof l[0]) > 0) && 
	    (0 == strcmp(expected, l))) ? 
		SUCCESS : FAILURE;
}
```

Here we broke the lines at 60 chars for "portable" readability. But,
the `return` can be one long line, which could be cut short if we also
know a few things about this code:

* The maximum length of the line is actually already a constant which can be re-used
* `SUCCESS == 0` and `FAILURE == -1`, as usual in C

```c
int expect_string(int sm, char const* ex)
{
    char l[MAX_LINE_LEN];

    return ((read(sm, l, MAX_LINE_LEN) > 0) && (0 == strcmp(ex, l))) - 1;
}
```

So, the line is now under 80 chars wide which was OK even in the days
of DOS.

Sure, this is still two lines, from which you can only escape if
you're in a single threaded environment and you can (re-)use a `static
char l[]`, or you're willing to sacrifice ease-of-use:

```c
int expect_string(int sm, char const* ex, char* ln, size_t max_len)
{
    return ((read(sm, ln, max_len) > 0) && (0 == strcmp(ex, ln))) - 1;
}
```

Now the user needs to allocate the "working buffer" (line) string and
pass its allocated length, which is rather inconvenient.

But, while the line is longer and more verbose than high-level code
variant, it actually handles the error explicitly (w/out exceptions,
which are expensive in so many ways) and does _not_ allocate _any_
(heap) memory, which means it is much faster.

## The C++ array reference w/template trick

In C++ you can use higher level abstractions, like `std::string` and
IOStreams, but you can also stay low-level but get extra goodies.  In
our case, the one-liner can be more user friendly if the caller has a
statically allocated string:

```c++
template <int N> inline
int expect_string(int sm, char const* ex, char (ln&)[N]) {
    return ((read(sm, ln, N) > 0) && (0 == strcmp(ex, ln))) - 1;
}
```

## Moral of the story

By carefully thinking about what you want to do in C, in our case,
with some strings, you can write code that does only _as much as it
has to_. Such code will usually be of similar verbosity as its
high-level code counterpart, but, if well-written, should be faster,
use less resources and handle errors explicitly.
