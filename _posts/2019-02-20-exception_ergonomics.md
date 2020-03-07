---
layout: post
title: 🚮 Exception ergonomics
lang: en
---


Exceptions should be application specific, rather than generic and
code/library specific.  They should be caught at the place where
there is enough information to handle them properly _and_ where
the handling takes the least amount of effort (code).

## Recap: what do we mean by exceptions

We mean the exceptions as defined by languages like C++, C#, Java,
Python, Ruby and others.  To some extent it also pertains to
CommonLisp's condition system and similar constructs in other
languages.  To a limited extent, we also mean Ada's exceptions.

We do _not_ mean processor exceptions and how to handle them
(you can't define your own processor exceptions, at least
not in mainstream CPUs).

So, we assume that you know the basics of exceptions in these
languages - what happens when an exception is thrown or caught, how to
catch them, can there be more than one "in flight", renaming, checked
exceptions, exception safety and so on.

## What we mean by exception ergonomics

It's "how to use exceptions to make your (coding) life easier and
avoid hurting yourself".

There are many horror stories about how using exceptions can be
terrible.  We will only incidentally reference them here.  It's
enough to just be aware that these exist and that one should be
very careful about using exceptions, or face dire consequences.

We will discuss:

1. When to throw exceptions
2. What exceptions to throw (their names and data)
3. When/where to catch them

We will not discuss things such as "exception safety", even though
they somewhat overlap with ergonomics.  But, their focus is on code
safety, and whatever the code ergonomics, safety should be preserved
always.

We will also not discuss exception mechanics, which can also differ
significantly between languages.  Things like "should we catch by
value, reference, pointer or maybe rvalue reference" in C++ and such,
while also somewhat related to ergonomics, have more implications to
exception safety and correctness, and should be done in a certain way,
ergonomics notwithstanding.


## How about not using exceptions at all

That's certainly an option. But, remember that:

1. Not all languages/toolchains allow "no exceptions" mode
2. If they don't, then some library you are using may throw
   exceptions, whether you like it or not.
3. If they do, then you may not be able to use some libaries, because
   those are not designed to work in an "no exceptions" mode.
4. Some languages, like Ada and Java, will actually throw exceptions
   "on their own", not only because some code raises them.
5. Horror stories aside, sometimes exceptions are a _good_ fit for the
   problem at hand.  Well written code with exceptions will make the
   "happy path" easy to read without error handling distractions _and_
   significantly reduce the amount of error handling code.  It's _not_
   easy to get to this "exceptions paradise" - it might be practically
   impossible, but, when it's possible, it's _very_ nice.

So, if it is possible in your code to not use exceptions _and_ your
code will be better of not using them, by all means, don't use them and
forget all about their ergonomics.

## Illustrating example

To illustrate the ideas we will use a real-world example.  It was a
moderately complex device which was connected to a PC, which handled
the GUI.  PC and the device communicated through a simple, device-
specific, Remote Procedure Call protocol.

The problem with RPCs is that communication can break, and thus _any_
RPC can fail for reasons that have nothing to do with the RPC itself
(say, RPC `GetTime()`, obviously has nothing to do with PC <-> device
communication).  So, you can "enrich" the RPC with a separate
out-parameter (`communication_success`), add a separate result (if
your language supports it) or play bit-tricks with the result code,
forcing all RPCs to have the same result type (some integer), like
`HRESULT` of some Windows APIs.

If you do not want to any of that, but want a "clean" RPC interface,
and you do not want something even nastier, like some `errno` or
`GetLastError()` interface which implies keeping a static (or, at
best, thread-local) variable with the result of last RPC call, your
only choice, in many languages, is to use exceptions.  That's what we
did in this case and, in general it worked out well.  There were
several issues which will cover here, including what to do to avoid
them.

The PC code was written in C++, so most examples will be in C++,
but the ideas are not C++ specific.


## Bad exception ergonomics advices

Interestingly enough, even though exceptions in the form we discuss
here have been around for ~30 years, there are not that many guidelines/
advices about their ergonomics.  But, pretty much all that one can find
are not very good.

### "Use exceptions for exceptional situations"

The most often heard one is the worst: 

> _Use exceptions for exceptional situations_

This is way too generic and ambiguous.  What one may consider
exceptional, another one may not.  Even worse, even if we agree on the
"exceptionality" of some situations, the same code might encounter, in
different contexts, different situations.  Consider "opening of a
file".  Does the calling code expect the file to exist or not?  If it
does, than failing to open should be an exception; otherwise it should
not.  But, that's only known in the calling code.

To mitigate this, some "open file" procedures (functions, methods,
subroutines...)  might offer modes, configurations and such (in
CommonLisp, one might use the flexible and powerful conditions
system).  But, those are just clutches for a fundamentally bad
interface.

### "Throw exceptions if procedure cannot meet post-conditions"

A slightly better one is

> _Throw exceptions if your procedure cannot achieve it's post-conditions_

While well-intended, it's not very applicable in practice.

Most procedure authors don't spend much time in contemplating the
post-conditions of the procedure.  One might argue that this is bad
and they should change their coding habits.  Nevertheless, this is
the situation in the real world.

Also, since such advice usually goes hand-in-hand with the advice that
"preconditions should not be exceptions, but asserts", it's hard to
figure out why two closely related concepts should be treated so
differently.  There is some validity in doing that, but, it's a
cognitive burden.

Even disregarding the "lazy programmer", this is tricky.
Post-conditions can change during the evolution of the code and
maintenance of a procedure.  Having to update our "throw if
postconditions fail" code can be a maintenance burden.  The thing is,
most of the time, to achive this, we would need to add some
"postcodition checking" code towards the end of our procedure.  This
can also be a performance issue, if the procedure is on a "hot path".
Then, one might be tempted to remove it in some "optimized build",
which is a bigger problem, as now these different builds behave
differently, exception-wise (which was one reason to avoid
"preconditions as exceptions").

At long last, what is a postcondition is often rather arbitrary.
Remembering our "open file", is "file successfully opene" a
post-condition?  If not, what is?  One might say "the file object
(structure, record, pointer...) returned is valid", but that is
ambiguous (is `nullptr` a valid pointer or `nil` or `None` a valid
object) and, at long last, any procedure worth a damn should _never_
return invalid objects.  Btw, it is one of the reasons why the
postconditions often change during evolution and maintenace.

### "Do as the standard library of you language does"

Well, it's certainly possible that some language has good exception
ergonomics in its standard library.  But, most mainstream languages
do not.

Most languages insist on throwing library-specific exceptions like
"Null pointer", "Index out of range", "File does not exist", etc.  As
we described elsewhere, this is a bad idea.

This does not extend to the language (runtime) itself.  Managed
languages like Java or C# may throw from their managed environment
(virtual machine), rather than standard library.  For example, they
may throw "Null pointer dereference".  Ada language was designed with
the idea that exception handling should be used for processor
exceptions (traps), such as divide by zero (or "access non-existent
memory address" - AKA page fault on processors with virtual memory
support).  Your code cannot raise such exceptions, even if it wanted
to.  Whatever exceptions as thus thrown "by the language itself" are
as they are, you need to live with them.  But they should not, in any
way, influence when to throw exceptions in your code or what
exceptions to throw.

## When to throw

Obviously, you throw when there is some error.  But, when is an error
"exception-worthy" versus being somehow indicated by some "special value"?

Essentially, you should throw when the code has no valid way of going
forward.  But, the problem is, that depends on the context.  As
discussed before, in one context, failure to open a file can be a
show-stopper (if the file has essential configuration data that has no
defaults), but in another context, it might be harmlessly ignored (if
a file contains some cache).

So:

> throw when you are certain that the code, under _any possible context_, cannot move forward

Obviously, opening a file should never throw, unless you make some
"intorelable file open" and use that procedure, instead of the regular
"file open" procedure in your application.

As we can see, this means that when you throw is essentially
application-specific.

### What about out-of-memory

As many have observed, out-of-memory is a rather special kind of
exception.  But, in reality, most of code will never be the one
that allocates memory, it will call some `malloc()`-like procedure.
Said procedure may throw or not, but, that is not our code.

In our code, when we do write some "memory manager", we should follow
the same advice, even if we do know that this is a kind-of special
situation.  So, in most situations, don't throw, but, if you know
that a particular memory manager really can't continue in any
context (essentially, some applicaiton-specific memory manager), then
you might throw.

### Remember the third option - simply terminate

In some situations, simply terminating is also an option.  For some
helper, script (like) code, or some batch-processing, it might be
perfectly fine to simply stop (abort) the program and display some
error message.  This can be done without involving exceptions and it
avoids any "error indicating values".

## What to throw

> Throw exceptions (codes, classes/objects) that are _application_
> specific.  Do _not_ throw generic or code/library specific
> exceptions.

This facilitates good exception handling.

### What not to throw - Python's 'KeyError' and similar

One of the best examples of "what _not_ to throw" exists in most languages
and their standard libraries.  Let's take Python for example.  The
standard dictionary type will throw an exception if you try to index
an element that is not in the dictionary.  The idea was that you would
get a nice error report if you don't catch it, but you can also catch
it and handle it if you wish.

But, really, how helpful is this report:

```python
Traceback (most recent call last):
  File "main.py", line 10, in <module>
    print(f(t))
  File "main.py", line 7, in f
    return a[y]
KeyError: 122
```

Sure, `y` is `122`, but, how did it get said value? What is `a` and
how come it doesn't have `122`?  This is a simple example, things can
be much worse, instead of `a` and `y`, you might get some complex
expression.  Not to mention that the call stack can be much, much
deeper, making it obfuscated.  You need to look at the code and
probably debug it (run it again).  A simple `main.py: line 7: Assert
KeyError`, reported before simply terminating the application would
suffice to give you the place to look for trouble and debug (if you
wish, you can still get the call stack at that point, you don't need
exception handling mechanics for that).

Even worse, how should you handle such an error?  This `a` is not known
to your code and you don't know how it's created.  Sure, you can
inspect the code, but, it might change and nobody will inform you to
update your code.  Essentially, such "code specific exceptions" can only
be handled "at the place of the call", like:

```python
    try:
        return a[y]
    except KeyError:
        return None
```

If the exception propagates ("escapes the calling function") it becomes,
as illustrated above, essentially _un-handleable_.

Now, if you know Python, you're probably aware that dictionaries have a
method `get` which does not throw, and the code can be rewritten as:

```python
    return a.get(y)
```

Which is kind-of the point.  The `get()` method is the one to use,
unless you're writing some short or ad-hoc scripts.  If you wish to handle
the missing key somehow, you still can:

```python
    x = a.get(y)
    if x is None:
        # handle it
    else:
        # use it
```

This is rather similar to a `try/catch` block, the differences are
superficial.

### Throw application specific exceptions

The idea is to throw an exception that the application can actually
handle.  Let's use our PC <-> Device specific RPC to illustrate.

Library/code specific exceptions would be something like "failed to
send data" or "response timeout".  These make as much sense as the
Python's `KeyError` described above.  No code other than the one
directly calling a RPC would know what to do with such exceptions.

But, an exception like "RPC failed" is application specific - at least
as much as can be specific in the context of executing a RPC.

#### Grey area - what is "application specific" in a library

Yes there is a grey area with-regards-to what is "application
specific" in a library.  In our example, the RPC will probably be
implemented in a library and the "failed RPC" can thus be thought of
as a library exception.  But, the thing is, it is geared towards the
application and the way it uses the library.  It is _not_ geared
towards how library works, which would be exemplified by exceptions
such as "response timeout".

Arguably, the best way would be to throw "RPC Such-and-such failed"
and have that be derived from "RPC failed" _and_ some "Such-and-such
failed", but that could be a kind of over-engineering.  Still, if done
well in a large-enough code base, could be very useful.

### Special consideration - checked exceptions

Checked exceptions are mostly known from Java, where some exceptions are
"checked", which means that the code _has_ to handle them, they can't be
left "unhandled".  There are other languages that support this, some
actually make all exceptions checked.

But, the question here is: if your languages supports checked exceptions,
when should you actually use them?

Of course, we don't want to go down to road of bad guidelines like
"use them one some exception really needs to be handled".

Important thing to consider is that checked exceptions have the nasty
effect of incuring a lot of "empty catch"es, just to "silence the
compiler".  So, it is rather obvious that it should be used for a
significant minority of exceptions.  But, that's not saying much, as
we might come up some "special" application which actually mostly
encounters this "minority" exceptions, and from its point of view,
they are actually a minority.

So, given what we discussed above, it should be easy to conclude what
to do here: raise a checked exception when your _application_ cannot,
_under any circumstances_, allow unhandling said exception.  For
example, real time applications cannot allow unhnadled exceptions.

Now, if you have a library that is meant to be used in different
applications, it obviously cannot throw checked exceptions and the
application itself needs to somehow make sure exceptions from that
library are handled.

## When/where to catch

> Catch when you have enough information to handle the error _and_
> where it makes the least amount of effort to do so.

The thing is, there might be many places where you could catch an
exception, but, only one (or few) require the least amount of effort.

Keep in mind that we do mean _actually_ handle the error.  We don't
mean "just write empty catch blocks", as is often done, especially in
Java, where one is forced to catch some "checked" exceptions.  A lot
of time you'll see terrible code like this:

```java
try {
    FileReader file = new FileReader("C:\\test\\a.txt");
}
catch (IOException ex) {
}
```
	
Sometimes the author may try to hide this with some logging, but,
that's just for show.  This is bad, the whole point of exceptions is
to not ignore them, yet that is precisely what the code is doing.
Yes, Java standard library is _terrible_ here.  Not only is opening a
file, which can be permitted to fail in _many_ situations, treated as
an exception _always_, the code is forced to handle this (rather than
just letting the application crash).

Now, it's hard to say any more on this, as this is now _very_
application specific.  But, since a lot of exceptions ergonmics is
application-specific, then we'll tackle this together with all other
aspects in a case study of sorts, that follows.


## Case study

We'll go back to our illustrating example and now treat is as a case
study.  What was actually done will be presented first, and then we'll
show how it fits to our guidelines and, when it doesn't fit, what
should have been done differently and how.

### What was done

The RPC library (it was a rather simple module, but let's call it
library to highlight it's usage) would raise a "RPC failed" exception
when any kind of communication error happened and the PC did not end
up getting a valid response.

This exception was then caught in the GUI event loop, in a helper
procedure that handled all GUI "commands" (started via menu, keyboard
shortcut, icon...).  The idea is that most commands would do some RPC
(most of them only one or two) and that the procedure for each command
would not handle (or even care) about exceptions, which would be
caught in a central place.  Something like:

```c++
try {
    switch (command) {
    case cmdGetTime:  cmdGetTime_exec(); break;
    case cmdTimeSet:  cmdTimeSet_exec(); break;
    ...
    }
}
catch (RPCException &ex) {
    PostMsg("Communication with device failed: %s", ex.what());
}
```

The problem arose with some commands that actually executed a lot of
RPCs.  The device had a rather complex configuration and the
requirement was that one can set the whole configuration with a single
command (reading it from a file on the PC).  Of course, this went
hand-in-hand with another command that would read the current
configuraiton from the device (and save it in a file on the PC).

This meant potentially hundreds of RPCs executed in one command.
Reading is not so bad - if any fails, one could say that reading of
configuration failed and report that to the user.  Of course, that was
deemed a too bad UX, as this could take a while (minutes), since the
communication link was slow.  So, some auto-retry logic was added,
which meant that the "read config" function would need to handle the
exceptions itself.  Still, not too bad:

```c++
retries = 0;
for (e = cfg_element.begin(); e != cfg_element.end(); ++e) {
    try {
        apply_config_element(*e);
    }
    catch (RPCException& ex) {
        if (++retries == 3) {
            break;
        }
    }
}
```

Of course, the try/catch we showed above now doesn't make sense for
this particular command (this catch block made the one "above it"
useless), but, there were some commands that did not do any RPCs, so,
that was OK - it was already useless for some commands.

The big problem was setting the configuration.  While adding a retry
was the first step, the problem was that even after a few retries, it
may fail, prompting an attempt to return to the previous
configuration.  You guessed it, this rollback of sorts can also fail.
This code was terrible, riddled with `try/catch` blocks until it was
deemed "good enough", but it was never really good enough.  It was so
terrible that we won't show it here.

### How it fits to our guidelines

The "when" to throw fits well.  If a RPC cannot be completed, it makes
no sense to move forward in any context.

The "what" to throw fits "good enough".  As discussed before, "RPC
failed" is application-specific enough.  It would have been nicer to
have derived classes for "reading time failed", "setting time failed",
etc, but, it is not essential.

For all the simple commands, which executed one or two RPCs, the
"when" to catch was good.  It was when we had enough info (some GUI
context) to display an appropriate message to the user, informing her
that her command failed so she can decide what to do next.  It was
also the least amount of effort, as it was done in a few lines of
code, rather in one of a about hundred commands, amounting to hundreds
of lines of code.

But, for the whole-configuration commands, this was obviously not
good.

### What should have been done differently

Let's set aside reading for now.  One could argue that it's good
enough, so let it be.

But, setting the configuration warrants some analysis.  When
thinking about such things, it usually helps to disregard the
"exception mechanics" and see what we want to do here:

1. Apply a configuration
2. If that fails, retry a few times
3. If it fails still, rollback
4. If rollback fails, retry a few times
5. If rollback fails still, give up and report an error.

Now, if we can see that "rollback" is merely applying the previous
configuration, there is step `0.` - read current configuration.

Thinking further, there could be other reasons that applying a
configuration can fail.  For example, there might be something wrong
with our configuration data (file got corrupt).  So, this is not only
about exceptions.  This will likely be reported as an error in, say,
the result of an RPC (our `SetTime()` might return `-1`).

Thus, it doesn't make sense to tie the failing of applying a
configuration to exceptions and our command procedure should
not handle exceptions at all and merely call a few helper functions,
essentially coding the above several steps:

```c++
    current_config = read_device_config();
    if (apply_config(user_config) != 0) {
        PostMsg("Failed to set the user configuration");
        if (apply_config(current_config) != 0) {
            PostMsg("Failed to roll back");
        }
    }
```

Obviously, the `read_device_config()` and `apply_config` would handle
the retries.  They can be trivial:

```c++
    for (retries = 0; retries < 5; ++retries) {
        try {
            read_device_config_raw(config)
            break;
        }
        catch (RCPException &ex) {
            continue; // looks like ignoring exceptions!
        }
    }
    if (5 == retries) {
        return -1;
    }
```

Where `read_device_config_raw()` can just go its merry way and blissfully
ignore exceptions.

The `apply_config()` can be structurally the same, just calling a
different `_raw()` and maybe having a different number of retries.  Of
course, for optimization, one could retry only the last piece of
information, not restarting at the beggining.  That would change the
structure of `apply_config()`, but not by much.

We can see here that it is not always less amount of effort to handle
the exceptions "up the call stack", sometimes it's actually quite the
opposite.

It's also interesting that an (essentially) empty catch block actually
means handling the exception in this code structure.  We could write
this differently if that bothers us, but, this idiom is fairly well
known in C/C++ - actually it's one way to kind-of simulate exceptions
when there are none (in C, or C++ in "no exceptions" mode).


## Moral of the story

Exception handling can be terrible but can also be quite effective.
If you can avoid it completely, consider doing so.  If you cannot,
apply the guidelines presented above to increase the chances of
achieving this efficiency.  Also remember that exception handling is
weird, most advice you heard about their ergonomics is not good and
they kind of have rules of their own, like empty catch block can
actually represent handling rather than ignoring and it's not always
better to handle exceptions up-the-stack.
