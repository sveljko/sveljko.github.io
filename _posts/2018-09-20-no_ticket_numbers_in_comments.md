---
layout: post
title: 🗄 No to ticket numbers in comments
lang: en
---


Often, a task you're working on has some identificator from some
"ticketing" system, like currently popular JIRA.  Usually it's a short
mnemonic followed by a number, something like: `MSP-5443`.  It's nice
to put this ID in stuff.  For example, if you put it in your VCS
comments (on commit/check-in), JIRA can link VCS commit to the ticket.
But, in source code, it can be distracting, become outdated and,
basically, is a maintenance nightmare.

## Why would you even do such a thing?

Well, the idea looks good. If you write down the ticket ID, it saves
you the trouble of describing at length tge problem or the features
you implement.  Anyone can just see the ticket and go to the ticketing
system for explanation.  Hell, some advanced development environment
might actually enable you to "click" on the ticket ID in your comment
and jump to it in the ticket system (Web)UI.


## That looks cool - how come it's not

Although they are somewhat connected, let's examine the reasons
separately.

### It is a distraction

When writing code, the best favor you can do for the future reader
(which might also be you) is to make the code as focused
(non-distracting) as possible.

The ticket number is distracting, because it has no meaning in itself,
thus it prompts the user to seek its meaning. This is a distraction,
even in the "high level IDE" which makes the distraction "as quick as
possible".

Even worse, tickets usually have various data, like "project", "epic",
"milestone", "linked tickets", "severity", "estimated time", "due
date" and many others which have _nothing_ to do with the code. So,
it's not easy to figure out "the things that are relevant to the
code".

You're much better off explaining what needs to be explained in the
comment.

### Ticketing systems are not forever

Well, they are not diamonds, so, that should have been obvious.

OK, jokes aside, ticketing systems are very "fragile". They are often
changed, which usually means that tickets don't get transferred to the
new system and the old is shut down quickly after the new one "gets
online".

Also, since they are separate and mostly treated as less important
than code (and rightfully so), they also loose data often, even
without the change/upgrade.

So, if your codebase is long-lived (more than a few years), the numbers
will become useles. Well, at least to you. They can be interesting
to software archeologists.

### Changes might not be localized

Remember Y2K? There was a joke that someone took the Y2K project to
mean "change all Ys to Ks in our user reports".

But, seriously, similar things happen. For example, someone might
request that user reports start using "initali_z_e" instead of
"initili_s_e". There will be changes "all over the place" and
"marking" them all with ticket ID would be futile.

Even some less contrived example suffers from the same problem.  You
might need to change code in several places and putting ticket IDs in
all of them is a chore. OTOH, if you don't put it in all places, you
need to think about "where and why to put them".

### It's next to impossible to maintain it

This is best explained with an example.

Suppose our well-intended programmer wrote something like this:

```ruby
# MSP-5443: Crash on adding new data to existing item
#           while dragging a mouse.
if dragging then
    allow_adding_data = false
end
```
	
It's a contrived example, but, taken out from real code (obviously,
obfuscated). Yes, this might not be the best fix here, but, it's just
to prove that this was "inspired by real events".

This seems well and clear. But, there comes another bug or feature,
say, adding data via keyboard. That prompts various changes,
including one in our snippet:

```ruby
# MSP-5443: Crash on adding new data to existing item
#            while dragging a mouse.
if dragging and not add_by_keyboard then
    allow_adding_data = false
end
```
	
Obviously, at this point, the comment is out-of-date. Sure, it's still
_somewhat_ relevant, but, the reader might conclude that _this_ is the
code done for MSP-5443. It is not. This is the code done for MSP-5443
and MSP-6554. So, to maintain this, we would:

```ruby
# MSP-5443: Crash on adding new data to existing item
#            while dragging a mouse.
# MSP-6554: Allow adding data via keyboard
if dragging and not add_by_keyboard then
    allow_adding_data = false
end
```

But, what if another ticket comes in and we need to update this code?
I think you can guess where this is going... and it's not a good place
to go to.


## From a code POV

Think of ticket IDs in comments as pointers to memory you don't own
or have any sense of control about.

It may be deleted without you noticing and you'll have a "use after
free" of sorts.

Or you may forget to update it, and it points to something completely
wrong.

## It's not just ticketing systems

The same problem exists with contract numbers, VCS commit/checkin
IDs (`this was added in commit 4455fa2`) and similar "external
references (pointers)".

## Let's not get carried away, here

This is different from, say, referring to a standard by its code,
like: `IETF RFC 3543` or `ITU-T Q.703:1992` (for some standards, you
need to refer to year or version). You use such things to "explain why
you're doing something", or "why some constant(s) have some value(s)".

Also, providing a reference to a book or an article/paper, for example
to "point" to the definition of an algorithm used, is fine.

Obviously, you're pointing to something that is stable, which makes it
OK. Like having a pointer to some memory you know will not be freed,
moved or otherwise messed up with _and_ your code, even if changed,
will still point to that same memory.


## Moral of the story

Things that look good and are done with the best of intentions, often
are not really such and have bad, unintended consequences.
