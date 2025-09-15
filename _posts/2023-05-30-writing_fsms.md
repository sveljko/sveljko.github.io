---
layout: post
title: 📠 Writing FSMs
lang: en
---

When explaining FSMs, people often focus on theory or
a particular, usually "advanced" approach of writing
them. Experience shows that the "low level" code
organization of FSMs greatly influences their quality.


## Recap: Finite State Machine

To avoid theoretical musings, FSM essentially represents
some "code divided", into several parts that are executed
at different times. Because one needs to know what code to
execute at what time / on what event, some state needs to
be kept.

For example, a traffic light will, either after some time(r)
expires, or on some other event, like synchronization with
other traffic lights or a detector of empty/full crossroad,
change the light. It would have states like: _red_,
_red-and-yellow_, _green_, _blinking-green_, _yellow_ (the
actual states may depend on jurisdiction).

While there are FSM that care only about state, those mostly
exist in hardware. In software, we usually care about the
event(s). In the simplest form, we will have a simple list
of events. For the traffic light, those may be 
_timer-expired_ (let's say there's only one timer which
pertains to any and all states / state transitions) and 
_crossroad-filled_ and _crossroad-emptied_. Depending on
the state and the event, some action will be performed
(or the event ignored) and state changed or kept. For the
traffic light, timer triggers change to "next state" while
filled crossroad will cause going to _red_ and emptied will
cause _green_ (for one direction).

FSMs can be presented in a tabular or a graphic manner.
There are several general ways of drawing FSMs. We won't 
go into that here.


## How do you code this

There are several basic ways and then some variants thereof.

### A FSM in one

The most straight-forward way is to put the whole FSM into
one function/procedure. Lets see this in C:

```c
enum State {
    red,
    red_yellow,
    green,
    blinking_green,
    yellow
};
enum Event {
    timeout,
    filled,
    emptied
};

void light(enum State *state, enum Event event) {
    switch (*state) {
    case red:
        switch (event) {
        case timeout:
            yellow(ON);
            *state = red_yellow;
            break;
        case emptied:
            red(OFF);
            green(ON);
            *state = green;
            break;
        }
        break;
    case red_yellow:
        switch (event) {
        case timeout:
        case emptied:
            red(OFF); yellow(OFF);
            green(ON);
            *state = green;
            break;
        }
        break;
    case green:
        switch (event) {
        case timeout:
            green(BLINK);
            *state = blinking_green;
            break;
        case filled:
            green(OFF);
            red(ON);
            *state = red;
            break;
        }
        break;
    case blinking_green:
        switch (event) {
        case timeout:
            green(OFF);
            yellow(ON);
            *state = yellow;
            break;
        case filled:
            green(OFF);
            red(ON);
            *state = red;
            break;
        }
        break;
    case yellow:
        switch (event) {
        case timeout:
        case filled:
            yellow(OFF);
            red(ON);
            *state = red;
            break;
        }
        break;
    }
}
```

OK, so, that's a lot. Because of the "ceremony", this code
is much longer than some linear code that does similar work.
There are a few things to note:

* unless the processing of two events in the same state is
  exactly the same, code reuse is very hard. We can see that
  processing of `filled` and `emptied` is basically the same
  across states, but we would "break" the organization of
  the code if we were to process them "out of place". This
  would also be a maintenance nightmare if we later figure
  out that processing is not actually the same in all 
  states.
* The action is usually at an unintuitive place. We turn
  the light, say, yellow in state _blinking-green_ not
  _yellow_. Not where you would expect to find it. Now, if 
  the whole FSM is in the same piece of code, that's not so 
  bad, though these two states might be dozens of lines 
  apart.
* Real-world actions are usually much longer. One either
  leaves them in the FSM and risks having a 
  `too-long-to-handle` code, or makes helper functions
  which then make the FSM hard(er) to reason about.
* Real-world FSMs usually have some more state, which then
  influences the next state. Say our traffic light receives
  "car count" instead of "ready made" events like above.
  Then it would keep the "last car count" and then figure
  out if the number of cars has increased or decreased to
  figure out to which state to go to. This makes FSMs even
  harder to follow.


### One per state

This is popular way to write FSMs. We'll not show the
whole FSM, just one state:

```c
enum State light_red(enum Event event) {
    switch (event) {
    case timeout:
        yellow(ON);
        return red_yellow;
    case emptied:
        red(OFF);
        green(ON);
        return green;
    }
}
```

Now, that's much "easier on the eyes", but:

* The "odd code placement" is a much bigger problem now,
  because the code-per-state is much further apart now.
  In practice, this significantly influences FSM
  comprehension.
* Some coding errors, like staying in the same state
  instead of changing it, are harder to spot.
* It's even harder to do code reuse between states.

It is very popular to define a function pointer for
this and then instead of something like:
```c
    switch (state) {
    case red: state = light_red(event); break;
    case red_yellow: state = light_red_yellow(event); break;
    ...
    }
```
have:

```c
    statefun statef[] = {
        light_red, light_red_yellow, light_green,
        light_blinking_green, light_yellow
    };
    ...
    state = statef[state](event);
```

This sounds "cool" and is a little less maintenance.
But one looses the ability to "track in code" and thus
reason about the code. Especially if this is some real-time 
code, where debugging is limited, this can be a source of
many hours wasted.

The essentially same technique is the basis of the
Object Oriented `State` Pattern. While OO does provide
some further ergonomic benefits (less maintenance), the
essential problems remain.


### One per event

This is not a very popular way, though it appears in
real-world code basically as often as the others. Again,
just a sketch:

```c
void timeout(enum State *state) {
    switch (*state) {
    case red:
        yellow(ON);
        break;
    case red_yellow:
        red(OFF); yellow(OFF);
        green(ON);
        break;
    case green:
        green(BLINK);
        break;
    case blinking_green:
        green(OFF);
        yellow(ON);
        break;
    case yellow:
        yellow(OFF);
        red(ON);
        break;
    }
    if (yellow == state) {
        *state = red;
    }
    else {
        *state = *state + 1;
    }
}
void filled(enum State *state) {
    yellow(OFF); green(OFF);
    red(ON);
    *state = red;
}
```

As we can see, in practice, event processing is same
or similar in all or many states, which can bring much
savings.

* The "same processing" of same event in different states
  is often misleading. One often finds later on that there
  are many subtle differences which are usually patched-on
  to existing code, making it a maintenance nightmare after
  a while.
* Since the emphasis, when talking about FSMs, is usually
  on the _state_, then this code organization can be
  stange for the unitiated.
* On the other hand, one can place particular event
  handling at a certain place in code, rather than a 
  function which is then called from said palce. If the 
  event processing is short, this can help reasoning about 
  code. Also, on constrained systems, it can save valuable 
  stack space.


### One per transition

This is usually too weird for explicit code. It looks _very_
smart with function pointers.

```c
void ignore(enum State*) {}
void red_timeout(enum State *state) {
    yellow(ON);
    *state = red_yellow;
}
enum State red_emptied() {
    red(OFF);
    return green;
}

transition_t transit[STATECOUNT][EVENTCOUNT] = {
    {red_timeout, ignore, red_emptied},
    {red_yellow_timeout, ignore, red_yellow_emptied},
    ...
}
```

Because it looks smart, this is a favorite for many 
"this is how you do FSMs" tutorials. Yet, it's not a very
good scheme. You have to create a separate function for
each transition, making them very distant from each other,
hurting code comprehension. From this nice state transition
matrix, you don't see what is the _resulting_ state of a
transtion, which hurts FSM comprehension. You have to
explicitly ignore an event, which is usually a chore and 
source of bugs.

For C99 or similarly in some other language, this can be 
more readable and easier to maintain:

```
transition_t transit[STATECOUNT][EVENTCOUNT] = {
    [red] = {red_timeout, ignore, red_emptied},
    [red_yellow] = {red_yellow_timeout, ignore, red_yellow_emptied},
    ...
}
```


### Full-blown table based FSM

In this scheme, a table defines the state, event,
transition function and the new state. Some like it tabled:

```c
void red_timeout(void) { yellow(ON); }
void red_emptied(void) { red(OFF); green(ON); }
row fsm[] = {
    { red, timeout, red_timeout, red_yellow },
    { red, emptied, red_emptied, green },
    ...
}
```

This seems weird, because there's not much to be gained
by putting such stuff in a table, unless you have some
"dynamic" FSM which changes during runtime, which is
very odd, as the purpose of the FSM is to handle things
differently based on state, so why change it rather than
enhanced it by adding new states and/or events?

From a certain POV, this looks even smarter than
"function per transition" and some authors "swear by it".

## State entry/exit

One way of reducing cognitive load is to treat
state entry or exit as an event. This often makes
code-positioning more intuitive: we set the red light
on upon _entering_ the `red` state. Often it also leads
to less code, as we set the red ligth on in only this
one place, the entry to state `red`, rather than in each
action before setting the state to `red`. There's a few
ways to do this:

1. If the whole FSM is in one function, than a
   a `goto` can be used. Ugly, but works pretty
   well in practice. Like:

```c
    switch (*state) {
enter_red:
        red(ON);
        *state = red;
        break;
    case red:
        switch (event) {
        case timeout:
            yellow(ON);
            goto enter_red_yellow;
        case emptied:
            red(OFF);
            green(ON);
            goto enter_green;
        }
```
   
    This trick can't then be used for exit, too.
2. Helper function can be utilized. Either one
   per entry/exit or grouped.

```c
void leave(enum State state) {
    switch (state) {
    case red: break;
    case yellow_red: red(OFF); yellow(OFF); break;
    case green: break;
    case blinking_green: green(OFF); break;
    case yellow: yellow(OFF); break;
    }
}
void enter_red(enum State *state) {
    leave(*state);
    red(ON);
    *state = red;
}
    ...
    switch (*state) {
        ...
    case yellow:
        switch (event) {
        case timeout:
        case emptied:
            enter_red(state);
            break;
        }
```

   We did it like this because usually leaving the
   state involves less work then entering it, so
   having all leavings in one function makes sense.
   The alternative would be to call `leave_yellow()`
   in the FSM function itself, thus having separate
   functions for each state leaving.
   This has the problem of reduced FSM comprehension,
   and this optimization of having one common `leave()`
   which is then called from each `enter_` doesn't help,
   because it lacks consistency.
3. Entry and exit can be proclaimed events just like any
   other. It makes using the FSM harder, but not by much.

```c
enum State light(enum State state, enum Event event) {
    switch (state) {
    case red:
        switch (event) {
        case entry:
            red(ON);
            return red;
        case exit:
            return red;
        case timeout:
            yellow(ON);
            return red_yellow;
        case emptied:
            red(OFF);
            green(ON);
            return green;
        }
    ...

    new_state = light(state, event);
    if (new_state != state) {
        light(state, leave);
        state = lignt(new_state, entry)
    }
```
    Similar "two state dance" needs to be done for any
    of the schemes.


## Auxiliary data

Say there's a railroad crossing down the road. We want
to keep our light _red_ while said crossing is closed,
because otherwise we could have a traffic jam in our
hands. Sure, our red can cause jams in the road before
us, but if said road is longer than the road till the
railroad crossing, that's fine.

One way to handle this would be to add additional states.
In this case, just `red_railroad_closed` is needed -
we assume that "old" red is `red_railroad_open`. This is
because if the railroad crossing is closed, we always go
to red.

There are scenarios where this would involve adding many
more states, which would be a cognitive burden. Also,
if the auxiliary data is not so limited, say if we have
a smart camera that detects when some car goes through
red light and then we count said times and if the count
is too high, we go to some special mode, like
_blinking-yellow_ which signifies "out of order" in some
jurisdictions. This count needs to be kept in each state,
so it's not feasible to implement it just by adding more
states.

So, here's how we would handle the railroad:
```c
struct FSM {
    enum State state;
    bool rairoad_open;
};
void light(struct FSM *fsm, enum Event event) {
    switch (fsm->state) {
    switch (state) {
    case red:
        switch (event) {
        case timeout:
            if (fsm->railroad_open) {
                yellow(ON);
                fsm->state = red_yellow;
            }
            break;
        case emptied:
            if (fsm->railroad_open) {
                red(OFF);
                green(ON);
                return green;
            }
            break;
        case railroad_open:
            fsm->railroad_open = true;
            break;
        }
        break;
    case red_yellow:
        switch (event) {
        case railroad_closed:
            yellow(OFF);
            fsm->railroad_open = false;
            fsm->state = red;
            break;
        }
        ...
```

## Invalid events

So far we were analyzing a real-world-ish example,
wherein no events are invalid. Anything can happen and
we need to handle it. We can ignore some events in some
states, but we can't halt the program, or some such thing.

But FSMs need not be real-world-ish. A FSM is very often
used for analyzing text - "lexing". Say we have a simple
lexer that supports quoting, which means `string` in many
programming languages. We have some three states:

1. Out of string
2. Inside string
3. Escape sequence inside string

While this much is similar in most languages, there's
something that's quite different - handling of newline
characters inside strings (with possibly different
handling for escape sequences). Some allow them, some have 
special handling and some forbit them. If our lexer is
to prohibit them, then a newline character in invalid
in the _inside-string_ state.

How does one handle "invalid" case depends on the code
and its environment. Maybe merely an `assert` is fine,
or a program halt is in order, or something completely
different. But how is such FSM code organized?

The "function per transition" is actually great for this,
as one can simply sprinkle some `invalid` function in
all the relevant places in the matrix. For all other 
schemes, one needs to expliclity put a call to `invalid()` 
in all state+event combinations.


## Which should I choose

It's hard to be very precise about this, but some general
guidelines are:

1. If it can fit in several dozen lines, one function
   for the whole FSM is the best.
2. In most FSMs that process something, rather than control
   something, one function per event is usually the best.
3. If (and only if) FSM is presented in a tabular manner,
   using function per transition is worth considering
4. One would be hard-pressed to justify a full-blown table
   approach.
5. If everything else fails, use function per state.

Outside of that, use your engineering judgement. Just
like there's no "one true way to write code", there's no
"one true way to write FSMs".

But do familiarise yourself with all the variants mentioned 
here and ignore the usual "this is the way" presentations
and tutorials on "how to write FSMs".