---
layout: post
title: send(), recv() and the corner cases
---

Designing, implementing and documenting corner cases is always "fun". Obviously, I'm kidding.
It's quite the opposite. It's boring.

We tend to assume how these will work, that it's OK and we don't need to worry about it too much.

But, this is software, it has no soul and plays no favorites to any values and/or (corner) cases.
Corner case should be "just another case".

So, let's take an example of the well-known sockets functions `send()` and `recv()`.
Both take a buffer (to send or receive) and a length (of said buffer). Well, at least in C,
in some higher lever language an abstraction of a "vector" or some such thing might be provided.
But, even if it is, it doesn't change the fundamentals of what we're going to describe here.

Depending on your sockets implementation, the length might be signed or unsigned integer of
various size. But, let's say that, even if it's signed, passing negative values is an obvious
error and we don't care much.

What we want to explore is setting the length to be zero: `0`.

When you think about it, it doesn't make sense. You want to send _nothing_? Just don't `send` at all.
You want to receive _nothing_? Good for you, but keep it to yourself.

So, why would you do it? Well, you might have some generic code that takes the length from somewhere
and you just passes it along. You don't want to sow `if (length > 0)` all around your calls to `send()`
/`recv()`.

That makes sense, so, what shall and will `send()` and `recv()` do in this cases.

In all the implementations I found, this is not documented. It's always something like:

> This function will send `length` number of bytes from `buffer`.

Sure, there's other text there, but, in general, there's nothing describing what happens if `length==0`.
Is that an error? And what error is reported (via `errno`, in most sockets)? If it's not an error,
what's the behavior? Is it different depending on some socket options, foremost the "(non-)blocking I/O".
Also, is it the same for connection-oriented sockets (like TCP/IP) and for datagram-oriented (like UDP/IP)?

What's the "big deal here"? Well, most, if not all, sockets also document this:

> The function returns `-1` on error (with error code in `errno`) or the number of bytes
> sent (_received_ for `recv()`).
> The return value (result) is `0` if the socket was lost (shutdown).

The "socket was lost" obviously is meant for connection-oriented sockets (actually, you use `sendto()`
for datagram sockets). Also, sure, not all sockets use `errno`, Windows uses
[WSAGetLastError](https://msdn.microsoft.com/en-us/library/windows/desktop/ms741580(v=vs.85).aspx)
and other libraries have other means of indicating the error), but, let's not get lost in the details
here. Assume that `errno` means "actual errno or its cousin in a particular sockets implementation".

So, if the result is the number of bytes written, and you "told" the function to write zero bytes,
it makes sense that it thinks it succeeded and to return `0`. But, that is at odds with the idea
that `0` indicates that the connection was lost.

The thing is, you're not sure "what to think here".

This is a perfect example of a corner case that was not thought about. At design time, a better
interface could have been devised. At (post) implementation time, a better description/specification
could have been done, indicating what happens.

You're probably wondering what actually happens? I didn't make a detailed survey, but, my limited
testing shows this:

* `recv(length=0)` will return `-1` with "WOULDBLOCK" in `errno` if the socket is non-blocking
* `recv(length=0)` will return `0` when the socket is blocking, because it will never read `0` bytes.
  it will essentially wait until the other side closes the socket, and then return `0`.
* similar goes for `send(length=0)`
* I didn't try `sento()` nor `recvfrom()`

Sure, from a certain POV, it makes sense, but, it's actually bad, as the same fundamental issue,
which is _bad_ usage of the API, has different error indicators depending on some setting which
has _nothing_ to do with the actual problem at hand.

As to not only point to errors, let's think about solutions.

The immediate solution would be to make `length=0` an error, return `-1` always with a special
error indicator `BUFFER_CANT_BE_EMPTY` (a particular sockets implementation might have something
like that already, if not, make a new one).

A better interface would have been to not have "special values" for the result. Have the result
always be error (or `0` if there's no error) and give the number of bytes actually sent/received
back in an (in-)out parameter. The closing of the connection would produce an error 
"CONNECTION_CLOSED". In this case, for `length=0`, simply return `0` (no error).

Basically, the moral of the story is that one does need to handle corner cases just like any other
cases, even if they don't "make sense" and "don't matter much" ("who cares what happens, as long
as nothing crashes"). The amount of time lost trying to makes sense of it all when you do find
yourself in the corner (case) is way too big, compared to a little documentation/specification
and making the code actually work per spec.
