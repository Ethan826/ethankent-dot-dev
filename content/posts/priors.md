---
title: "Priors"
date: 2019-12-02T12:50:00-06:00
draft: false
---

There's a lyric in a They Might Be Giants song:

> Now you're the only one here,<br>
> Who can tell me if it's true:<br>
> That you love me,<br>
> And I love me.^[[_Kiss Me, Son of God_](https://youtu.be/gLcp8Dm-ejU?t=67), _Lincoln_ (Bar/None Records 1988).]

The lyric is effective because it uses our expectations of rhyme scheme,
parallelism, and reciprocity to have us all but certain that the final line
of the verse will end with _you_. And the surprise serves a purpose: it uses
the flouting of our expectations as a way of making us feel the potency of
the narrator's self-centeredness.

The same thing happens when we read code: our expectations are set by
familiar patterns. When something confounds our expectations, we will likely
assume it serves a purpose. Sometimes our priors our so strong that—like an
optical illusion—we see what we expect rather than what is actually there.

In practice, that means that a departure from common patterns may present in
at least two ways:

1. If the difference is something you notice but serves no purpose, you may
   become confused, because you assume the difference _does_ serve a purpose.
2. If the difference is subtle or your prior is strong enough, you may not
   notice the difference.

## Departing from normal patterns for no reason is confusing.

Consider this code:

```ruby
def do_that_thing
  return :thingy_1 if @foo.bar == :baz
  @blip.thingy_kind
end
```

I would expect this code to be written using a very famous thing called an
`else` statement:

```ruby
def do_that_thing
  if @foo.bar == :baz
    :thingy_1
  else
    @blip.thingy_kind
  end
end
```

or even

```ruby
def do_that_thing
  @foo.bar == :baz ? :thingy_1 : @blip.thingy_kind
end
```

I also have the general prior that code should have as few execution paths as
possible, and expect that short-circuit returns will generally be used to
bypass significant amounts of code.

So the first code listing winds up taking some extra time to think through.
"Hmm," I think, "I wonder why this isn't an `if`/`else`. I must be missing
something here. This isn't a short-circuit that avoids a big chunk of code.
Is there something about the conditional that prevents it from working with
an `if`/`else`? No. Why is it like this? I guess there's no particular
reason. Weird. Wait, what was I about to do? I better check in on Slack."

## Doing something important in a subtle, unexpected way is dangerous.

I have seen this pattern several times:

```ruby
def foo(my_arg)
  if (result = expensive_computation(my_arg))
    @my_orm.computed_value = result
  else
    MyLoggingBuddy.log_nobody_reads("It has all gone terribly wrong")
  end
end
```

My thinking here is "Okay, check to see if `result` equals
`expensive_computation`. Wait, where is `result` defined? Is that a method
call? It can't be a local. Looks like there's a bug here, too: that's a
single equals sign. _Oh, wait_: is that intentional? Is this assigning the
result of calling `expensive_computation` to `result`, and then evaluating
`result` for truthiness? Ugh—I see _why_ this is happening, and I guess I
vaguely remember that it's an idiom in Ruby to use parentheses in an `if`
statement if you're doing this, but that is super easy to miss."

My strong prior is that it is an extremely common error to use `=` instead of
`==` in a conditional. Another (albeit weaker) prior is that code will
generally separate distinct kinds of operations—avoiding things like
assignment within control flow.^[That is, unless language constructs exist to
do exactly that, as in `match` and `if let` statement in some languages.]
Here, things run so strongly against my priors that I may not even see what's
going on: my eyes don't see that single equals sign the first time around.

<center>❧&nbsp;❧&nbsp;❧</center>

If you do something obvious that differs from your reader's priors for no
particular reason, your reader will likely assume it _was_ for good reason
and will waste time trying to figure out what that reason was. Even worse, if
you do something subtle that differs from you reader's priors for an
important reason, your reader may miss what's going on entirely, as we tend
to pay little attention to something familiar.
