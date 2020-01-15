---
layout: post
title: ⏰ Real-time programming primer
lang: en
---

It's hard to explain what is "real time programming" and how to do it.
Instead, we'll show some "everyday" code that is not "real time" and
how to go about making it "real time".  As we'll see, no, it's not
enough just to avoid dynamic memory allocation - it's not even
_required_.

## Real-time: what is it?

There are several definitions of this. None are very good, frankly.
Even the term itself is severely lacking. I mean, if some programs are
"real-time", what are the others? "Unreal-time"?

In general, a real-time program (software) deals with some real-world
events and needs to respond to those in a defined time ("deadline").
Real-world is taken broadly here, passage of time is also a
"real-world" event.

Think of an elevator controller, it responds to passengers pressing
buttons and opening/closing doors and if it doesn't stop the elevator
in time, it may miss the floor, or worse.

### OK, what's so special about real-time SW?

There are a few points of interest:

1. RTSW must not crash or otherwise stop performing the main features
   (sure, playing music can stop, but, not the elevator control)
2. it can't degrade performance "below" deadline
3. it usually works "non-stop", 24x7, unless in maintenance mode
4. it's often also embedded, that is, has constrained memory, CPU
   and other resources
5. Often, there are no "Back-ups" and even if there are, the
   "take-over" has to be performed quickly and without "going beyond
   deadlines", though, some delay might be acceptable at certain
   points in time. Since most people sleep at 04:00, it's OK to update
   the elevator SW at that time. The one party girl that's coming home
   at that time can wait a few minutes (or, better yet, start making
   better life choices).


### How is this different from a Web server?

Web servers, well, at least the successful ones, deal with _high
volume_ of transactions. Sure, it's better that they don't consume
much resources, don't crash, etc., but, it's acceptable to have some
degradation of performance once-in-a-while. Also, usually there are a
_lot_ of backups.

For example, Facebook actually does upgrades by deploying new code to
a subset of servers, which is thousands, then waits a while to see if
something crashes or fundamental features don't work. Obviously, often
it does, they roll back and fix the issues before having another go at
it. You can't do that in an elevator.


## How does one go about writing RTSW?

Let's start with our primer. We'll take a pretty "bland" piece of
code, a "delete directory function" that's "unreal time" SW and then
transform it. 

### Delete directory, everyday-style

Let's assume that our directory deletion is crucial to the
real-time performance of some RT system. It's hard to imagine that
being the case for an elevator, but, elevator is not the only
RT system, just the one we like to abuse.

Here is the "everyday" code, in C (most RTSW is written in C), using
POSIX API for directory traversal:

```c
int delete_dir(char const* dirname)
{
    int  rslt = -1;
    DIR* dir  = opendir(dirstack);
    if (NULL == dir) {
        return -1;
    }
    do {
        char           f[MAX_PATH + 1];
        struct stat    statbuf;
        struct dirent* entry = readdir(dir);
        if (NULL == entry) {
            break;
        }
        if (!strcmp(entry->d_name, ".")
            || !strcmp(entry->d_name, "..")) {
            continue;
        }
        snprintf(
            f, sizeof f, "%s/%s", dirname, entry->d_name);
        rslt = stat(f, &statbuf);
        if (0 == rslt) {
            if (S_ISDIR(statbuf.st_mode)) {
                rslt = delete_dir(f);
            }
            else {
                rslt = remove(f);
            }
        }
    } while (0 == rslt);

    closedir(dir);
    remove(dirname);

    return rslt;
}
```

Obviously, we don't do much in the way of error handling and such,
but, this is illustration code, not production-ready code.

You may notice that we used stack allocation for `f` (file name)
instead of dynamic allocation (`malloc()`), which is one of the first
things people associate with "real time software".  It's interesting
enough to deserve a side-bar.

### Dynamic memory allocation in RTSW

You've probably heard the "you can't have dynamic memory allocation
in RTSW". Which is mostly true, but it also depends on your allocator
and what you use the memory for.

There are some real-time memory allocators. For example, the
stack-memory allocator is real time, because it always does just a few
instructions.  Also, there are some "bitmap" allocators that are
real-time (time-bound).

If you use such allocators for features that are "OK" to fail if you
exhaust your dynamic memory, then you're OK, too. So, music playing in
our elevator can allocate buffers via dynamic memory and stop playing,
or, say, restart the tune when out of memory. But, the elevator
control can't use dynamic allocation, because it must not fail.

### On with the show - remove recursion

It is not strictly prohibited to use recursion in RTSW. But, you can't
use _boundless_ recursion. That is, if you know / can prove that your
recursion will be at most N levels deep and that you do have enough
memory for the stack in that case, you're OK.

For example, say you're developing SW for a cross-bar switch. Finding
a path to connect two points can be recursive, in general, because the
upper bound for the level of recursion is the number of rods/bars that
you can (cross-)connect. Because it doesn't make sense to connect
twice to the same bar for the same connection. So, if the number of
bars is small enough to fit in your available stack memory, you're
good.

But, obviously, a directory tree can be _very_ deep and overflow the
stack memory.

So, we need to remove the recursion.

In general, this involves handling the stack ourselves. In this case,
the stack is that of (sub-)directories to delete.

By default, one would make either a list or an array to represent said
stack. If using a list, it would take elements from a pool (an array,
usually). In both cases, one would need to know the maximum level of
the stack (in our case, depth of the directory tree). In some
situations, you might know that (like the cross-bar example). But, for
directories, you mostly don't. The whole point is for one to be able
to nest them directories.

But, we can observe that this "stack of directories" has a well known
representation - the `dir/subdir/subdir....`. So, instead of keeping a
stack, we can simply keep a string. We start with:

	dir
	
then we have:

	dir/subdir
	
then:

	dir/subdir/subdir
	
and once that is done, we're back at:

	dir/subdir
	
basically, just add and remove the last dir from the string, maintaining
a stack "inside" the string.

A nice point here is that, on most systems, there is some reasonable
limit on how much chars is there in a file-path, at most. Sure, we all
have cursed some SW that can't handle paths longer than 260
characters, thus can't copy some backups that accumulate such long
paths. But, in general, it's not reasonable to expect a user to type
such long paths or use them in any meaningful way. So, there is some
`MAX_PATH` to be used for a particular system (and we already used it
in the "everyday" code).

Now, we can refactor our deleter:

```c
int delete_dir(char const* dirname)
{
    char dirstack[MAX_PATH + 1];
    int  rslt = 0;

    snprintf(dirstack, sizeof dirstack, "%s", dirname);
    while ((0 == rslt) && (dirstack[0] != '\0')) {
        bool pushed = false;
        DIR* dir    = opendir(dirstack);
        if (NULL == dir) {
            rslt = -1;
            break;
        }
        do {
            char           f[MAX_PATH + 1];
            struct stat    statbuf;
            struct dirent* entry = readdir(dir);
            if (NULL == entry) {
                break;
            }
            if (!strcmp(entry->d_name, ".")
                || !strcmp(entry->d_name, "..")) {
                continue;
            }
            snprintf(f,
                     sizeof f,
                     "%s/%s",
                     dirstack,
                     entry->d_name);
            rslt = stat(f, &statbuf);
            if (0 == rslt) {
                if (S_ISDIR(statbuf.st_mode)) {
                    strcpy(dirstack, f);
                    pushed = true;
                    break;
                }
                else {
                    rslt = remove(f);
                }
            }
        } while (0 == rslt);

        closedir(dir);
        if (!pushed && (rslt == 0)) {
            rslt = remove(dirstack);
            pop(dirstack);
        }
    }

    return rslt;
}
```

If you missed it, we have a new helper function:

```c
static void pop(char* dirstack)
{
    char* s = strrchr(dirstack, '/');
    if (s != NULL) {
        *s = '\0';
    }
    else {
        *dirstack = '\0';
    }
}
```

This code is significantly more complex and harder to reason about
than the "everyday" code.

But, we have solved the problem of recursion and are not using any
dynamic allocation. Sure, `opendir()` might use some, but, let's
assume that this is some "real time POSIX" and that this is "OK".

Are we done? No, not really...

### You'd better stop

Just like we can't have memory-boundless code, we can't have
time-boundless code. But, since directory tree can be deep, it is, for
all intents and purposes, boundless.

In the elevator, the much plays while the car is moving. So, our SW
has to control direction and speed of the car and brakes to stop the
car, all in the same time as playing. One CPU (core) can't do all of
that really at the same time, but we can do most of the stuff "fully"
when we need to do them. But, deleting a directory tree can be very
time consuming for big trees, so we can't just stop doing everyting
else.

So, we have to do what we often do in RTSW: "break up long
calculations".  We have to allocate some time to do the work, stop
once it's exhausted, then come back later and do some more, repeating
that until we're done.

This usually involves maintaining some state, usually via some Finite
State Machine. But, in this case, we're lucky, sort of.  The
filesystem _is_ such a state and it's already maintained.  So, we're OK
with simply stopping and "re-trying" the next time.

How we come up with the amount of time to allocate is very system-dependent,
so our function will get a new parameter that defines this:

```c
int delete_dir(char const* dirname, int ms_limit)
{
    char            dirstack[MAX_PATH + 1];
    struct timespec tend;
    int rslt = clock_gettime(CLOCK_MONOTONIC, &tend);
    if (0 == rslt) {
        tend.tv_nsec =
            (tend.tv_nsec + ms_limit * NANO_IN_MILLI)
            % NANO_IN_UNIT;
        tend.tv_sec +=
            (tend.tv_nsec + ms_limit * NANO_IN_MILLI)
            / NANO_IN_UNIT;
    }

    snprintf(dirstack, sizeof dirstack, "%s", dirname);
    while ((0 == rslt) && (dirstack[0] != '\0')) {
        bool pushed = false;
        DIR* dir    = opendir(dirstack);
        if (NULL == dir) {
            rslt = -1;
            break;
        }
        do {
            char           f[MAX_PATH + 1];
            struct stat    statbuf;
            struct dirent* entry;

            if (are_we_there_yet(&tend)) {
                rslt = +1;
                break;
            }
            entry = readdir(dir);
            if (NULL == entry) {
                break;
            }
            if (!strcmp(entry->d_name, ".")
                || !strcmp(entry->d_name, "..")) {
                continue;
            }
            snprintf(f,
                     sizeof f,
                     "%s/%s",
                     dirstack,
                     entry->d_name);
            rslt = stat(f, &statbuf);
            if (0 == rslt) {
                if (S_ISDIR(statbuf.st_mode)) {
                    strcpy(dirstack, f);
                    pushed = true;
                    break;
                }
                else {
                    rslt = remove(f);
                }
            }
        } while (0 == rslt);

        closedir(dir);
        if (!pushed && (rslt == 0)) {
            rslt = remove(dirstack);
            pop(dirstack);
        }
    }

    return rslt;
}
```

Yes, we acquired a new helper function:
```c
static bool are_we_there_yet(struct timespec* there)
{
    struct timespec t;
    int rslt = clock_gettime(CLOCK_MONOTONIC, &t);
    if (rslt != 0) {
        return true;
    }
    return (t.tv_sec > there->tv_sec)
           || ((t.tv_sec == there->tv_sec)
               && (t.tv_nsec > there->tv_nsec));
}
```

and some constants:
```c
#define NANO_IN_MILLI (1000*1000)
#define MILLI_IN_UNIT 1000
#define NANO_IN_UNIT (NANO_IN_MILLI * MILLI_IN_UNIT)
```

Here we're using `clock_gettime(CLOCK_MONOTONIC,...)` to keep track of
time, but, that's just to "keep us within POSIX", any similar API
would do.

Obviously, this is even more complex now, and harder to use - we need
to figure out the amount of milliseconds to pass to this funciton and
we have to handle `+1` as an indicator "call me later".

### But, what about threads/tasks?

You're probably wondering about RTOS-es and their support for
tasks/threads.  We could run this in such a (real-time) thread and not
worry about time spent. Couldn't we?

Well, first, a lot of RTSW doesn't have a RTOS, or the RTOS is simply
"not up to the task".

But, more fundamentally, using threads involves some coordination,
possibly locks. Locks often "grace us" with deadlocks or priority
inversion and such nice things, and we obviously can't have
those. There is the famous story of the Mars Pathfinder Lander SW that
crashed on the surface of Mars because of a deadlock - not to be
confused with the Mars Polar Lander that physically crashed, for
unknown reasons.

Even without locks, you'll have some "atomics", which often degrade
the performance and make "deadline calculations" much harder. These
often "grace us" with "lockless algorithms", which are notoriously
hard to implement without bugs, besides being hard on deadlines.

Thus, in general, putting "long calculations" in tasks/threads is most
often _not_ the way to go in RTSW, at least not for essential
features.  Tasks/threads are mostly to encapsulate different things
that the RTSW does and are triggered on specific events, like timers
("periodic tasks") or pin status change (representing state of an
elevator push button, for example). For the most part, they abstract
the CPU intterupts/exceptions and provide some higher-level way of
controlling them.


## Moral of the story

Real-time software is harder to develop simply because it has more
things to consider/handle than "everyday" SW, even some specific
software as Web Servers.

But, the good news is that you can, often, start from an everyday
piece of code, and mold it into RTSW in a more-or-less systematic way.
Even if refactoring later looses any semblence of the original code,
still, you can have a not-so-rocky path to v1.0.
