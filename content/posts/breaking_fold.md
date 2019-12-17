---
title: "Breaking out of the fold"
date: 2019-12-15T15:58:10-06:00
draft: false
author: Ethan Kent
---

This article addresses an issue [alluded
to](https://ethankent.dev/posts/i_fold#fold-can-be-used-to-define-all-the-other-important-higher-order-functions)
in my previous one. Namely, we sometimes want to break out of the iteration
within `fold`, but it is not easy to do that in many languages.

## `fold` has a problem with short-circuiting.

In defining `any` and `all` by using `fold`, as we did in the [previous
article](https://ethankent.dev/posts/i_fold#fold-can-be-used-to-define-all-the-other-important-higher-order-functions),
we did not short-circuit. But we'd get a better constant factor for our time
complexity if we can `break` from those methods once we know the final
outcome.[^when-to-short-circuit] Let's see if we can do this with Ruby (using
the built-in `reduce` method):

[^when-to-short-circuit]: For `any` that means once we encounter the first value that causes the anonymous function to return a truthy value, and for `all` that means the first value that causes the anonymous function to return a falsey value.

```ruby
# `any` using `reduce`; we're going to skip the actual function definition as I
# don't want to distract with Ruby's weird blocks and lambdas.
!![false, true, false, false, false].reduce(false) do |a, e|
  puts e ? "Breaking" : "Continuing"
  break true if e
  a # We'd actually use `each_with_object`, but that's not the point here.
end

# Continuing
# Continuing
# Continuing
# Continuing
# Continuing
# => false

!![false, true, false, false, false].reduce(false) do |a, e|
  puts e ? "Breaking" : "Continuing"
  break true if e
  a
end
# Continuing
# Breaking
# => true

# "Breaking" + no more output means short circuiting. Voila!
```

In other words, Ruby gives us a version of `fold` that allows us to
short-circuit the looping. This isn't possible in all languages, however.
Consider this JavaScript:

```javascript
const fold = (array, fn, initialAccumulator) => {
  let accumulator =
    typeof initialAccumulator === "undefined" ? [] : initialAccumulator;
  for (const element of array) {
    accumulator = fn(accumulator, element);
  }
  return accumulator;
};

const any = (array, fn) => {
  return fold(
    array,
    (truthFlag, element) => {
      if (fn(element)) {
        break true; // This isn't valid syntax.
      } else {
        return truthFlag;
      }
    },
    false,
  );
};
```

In Ruby, putting `break` in a block is pretty powerful. It causes a return
from the function that `yield`ed to that block. In other words, it _reaches
outside of its own scope_ and causes the function that called _it_ to return.
What's more, `break` is an expression that can make that outer function
return the value you give it: `break true`, for example. If this seems subtle
and in the weeds, the takeaway is that Ruby lets you hit the eject button
from way down inside the scope of the thing[^blocks] passed to `reduce`
during one particular iteration of that function, reach up to the `reduce`
itself, and make the whole `reduce` bail out.

[^blocks]: I use _thing_ advisedy: it's important that it's a block rather than a function in Ruby.

That simply isn't possible in JavaScript as a `break` doesn't have the power
to cause an outer function to return. The JavaScript interpreter looks at the
`break` and thinks we must be talking about the anonymous function it's
inside of, the one that begins `(accumulator, element) =>`. And it says,
"What do you mean, `break`? I don't see any loop inside my scope!" The
interpreter doesn't take any notice of the fact that the anonymous function
is called from inside a loop. That's a different scope, and as I said,
JavaScript doesn't let something inside the anonymous function pop its head
out of its own scope and then tinker with the control flow of the loop it's
situated inside of, namely the `fold` itself.

Think of it this way: imagine we wrote the function separately, like this:

```javascript
// Assume `fold`, as defined above, is in scope.

const anyHelper = fn => {
  // Return a function appropriate for passing to `fold`. We will need to curry
  // `anyHelper`, closing over `fn`, so that it matches the function signature
  // expected by `fold`'s second argument.
  return (truthFlag, element) => {
    if (fn(element)) {
      break true; // Invalid syntax. But imagine this worked for now.
    } else {
      return truthFlag;
    }
  };
};

const any = (array, fn) => {
  const curriedAnyHelper = anyHelper(fn);

  return fold(array, curriedAnyHelper, false);
};

console.log(any([true, false, false], x => x));
```

But now somebody comes along and tries to call `anyHelper` directly.

```javascript
anyHelper(x => x)(false, true);
```

That's a weird thing to do, to be sure; it's a pretty useless function
outside of `fold`. But let's go with it. Now `break` makes absolutely no
sense. And we don't really want to have function syntax be valid or invalid
based on its enclosing scope.[^context-specific-syntax] There are several
other considerations weighing against this.[^other-reasons]

[^context-specific-syntax]: That's different than simply introducing a bug based on something going on in the enclosing scope. We can easily do that with a function that closes over an undefined variable or a variable of the wrong type. But that's a runtime issue. Again, it's not that the _syntax_ became invalid based on a runtime binding.
[^other-reasons]: First, how far up the call stack should `break` be allowed to traverse? Sure, we might say "only one level," but now we can't extract a sub-helper out of the function passed to `fold`. Second, we are fundamentally messing with a `goto`-like construct which is Something to Worry About. (I am aware of JavaScript `break` statements with `label`s, and see how that could help address some of these issues I'm raising. The main reason for not allowing `break` is, I think, stated in the main text.)

## There are ways to work around that problem.

There are at least two ways to have the effect of breaking out of `fold` in
languages where that's not directly possible. The `itertools`
[library](https://crates.io/crates/itertools) teaches one way, the
`fold_while` method. Then the Rust standard library provided a second
approach: `try_fold` (which actually led to `itertools`'s deprecating
`fold_while`, though I think `fold_while` is still the best approach in some
circumstances).

### `fold_while` wraps the accumulator in a `Continue` or `Done` to interact with looping.

So that we can stick with JavaScript, I will define `fold_while`, stolen from
Rust's `itertools` concept, in JS. The upshot is that we will wrap the
accumulator in `Continue` or `Done`. Before the loop each iterates each time,
it checks to see which wrapper the previous iteration returned. If it's a
`Done`, the loop breaks.

```javascript
// The three classes are basically a poor man's enum (of the Rust type where the
// variants can wrap values). We need the functionality provided in
// `FoldWhileBaseContainer` and then the two subclasses just so we can switch on
// type.
class FoldWhileBaseContainer {
  constructor(value) {
    this.value = value;
  }

  // Unwraps the `Continue` or `Done`.
  intoInner() {
    return this.value;
  }
}

// Once again, these only exist so we can switch on type.
class FoldWhileContinue extends FoldWhileBaseContainer {}
class FoldWhileDone extends FoldWhileBaseContainer {}

// Convenience factory helper thingies
const Continue = value => new FoldWhileContinue(value);
const Done = value => new FoldWhileDone(value);

// Here's the actual `fold` function we're defining. The preceding has been
// setup.
const foldWhile = (array, fn, initialAccumulator) => {
  let accumulator = Continue(
    typeof initialAccumulator === "undefined" ? [] : initialAccumulator,
  );

  for (const element of array) {
    // Are we supposed to `Continue`? If we get a `FoldWhileContinue`-wrapped
    // value from the previous iteration, we continue with the normal `fold`.
    if (accumulator.constructor.name === FoldWhileContinue.name) {
      accumulator = fn(accumulator.intoInner(), element);

      // Or are we supposed to be `Done`? This is the key to short-circuiting.
      // If we get a `FoldWhileDone`-wrapped value, we `break`.
    } else if (accumulator.constructor.name === FoldWhileDone.name) {
      break;

      // Or did somebody mess up?
    } else {
      throw new TypeError(); // Helpful message goes here.
    }
  }

  return accumulator;
};

// Now we can define a short-circuiting `any` based on `foldWhile`.
const any = (array, fn) => {
  return foldWhile(
    array,
    (truthFlag, element) => {
      console.log("Running");
      return fn(element) ? Done(true) : Continue(false);
    },
    false,
  ).intoInner();
};

any([false, false, false, false, true], x => x);
// Running
// Running
// Running
// Running
// Running
// => true

any([true, false, false, false, false], x => x);
// Running
// => true
```

The function passed to `fold` is no longer trying to alter control flow
_directly_. Instead, it is passing a message up to `foldWhile`, and
`foldWhile` is the thing actually calling `break`. It's no longer something
like a `goto`; it's now a message being passed.

We can do the same kind of thing for our "game" from the [previous
post](https://ethankent.dev/posts/i_fold#fold-is-useful-when-we-must-iterate-with-a-flag-or-intermediate-result),
in which you must collect one `tinyBoomer` or two `megaBoomers` to win, a
`nothing` does nothing and a `whammy` makes you lose, assuming you haven’t
yet won.

```javascript
const tinyBoomer = Symbol("tinyBoomer");
const megaBoomer = Symbol("megaBoomer");
const nothing = Symbol("nothing");
const whammy = Symbol("whammy");

const isWinningGame = gameTape => {
  return foldWhile(
    gameTape,
    (status, event) => {
      if (!(status.hasWon || status.hasLost)) {
        switch (event) {
          case whammy:
            status.hasLost = true;
            break;
          case megaBoomer:
            status.hasWon = true;
            break;
          case tinyBoomer:
            status.tinyBoomerCount += 1;
            if (status.tinyBoomerCount >= 2) status.hasWon = true;
        }
      }
      return gameOver(status) ? Done(status) : Continue(status);
    },
    {
      hasWon: false,
      hasLost: false,
      tinyBoomerCount: 0,
    },
  ).intoInner().hasWon;
};

const winner = [tinyBoomer, nothing, nothing, tinyBoomer, whammy];
const loser = [tinyBoomer, nothing, whammy, tinyBoomer, megaBoomer];

isWinningGame(winner);
isWinningGame(loser);

// => true
// => false
```

This kind of approach is probably the best we can do in JavaScript. But
Rust's standard library now has a built-in method for breaking out of `fold`.

### Rust's `try_fold` gives us a way to break folding using the standard library.

Rust's [standard
library](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.try_fold)
now exposes a `try_fold` method on iterators. It automatically breaks when
the closure passed to it returns a `None` value. The only problem that raises
is that `None` can't wrap a return value like the `itertools` library's
`Done` variant, and so we can't do the equivalent of Ruby's `break foo`, in
which the syntax we use to cause the looping to quit is capable of returning
a value. That's why I think `itertools` shouldn't deprecate `fold_while`.

Anyway, we _can_ use `try_fold` to define `any`, as shown below, though it's
a kind of ugly, awkward solution. As mentioned, `try_fold` keeps looping as
long as the accumulator is a `Some` type. It breaks the first time it gets a
`None`. So we could probably skip the `true`s and `false`s altogether, and
the fact that `None` means `true` is confusing. We really are abusing the
method at this point.

```rust
// We would actually make this generic over any iterable, but for simplicity's
// sake...
fn any<T>(arr: &Vec<T>, func: fn(&T) -> bool) -> bool {
    arr.iter().try_fold(false, |truth_flag, element| {
        println!("Running");
        if func(element) {
            // We need `None` to break iteration, but it actually represents a
            // `true` result, which is confusing.
            None
        } else {
            // We don't really need the accumulator at all.
            Some(truth_flag)
        }
    }).is_none()
}

fn main() {
    let not_short_circuitable = vec![false, false, false, false, true];
    let short_circuitable = vec![true, false, false, false, false];

    assert!(any(&not_short_circuitable, |x| *x));
    assert!(any(&short_circuitable, |x| *x));
}
```

## This `any` stuff was all to prove a point; don't _actually_ define `any` with `fold`.

The best way to define `any` in a language that uses iteration is something
like this:

```javascript
const any = (array, fn) => {
  for (element of array) {
    if (fn(element)) return true;
  }
  return false;
};
```

The best way to define `any` in a language that uses recursion is something
like this:

```javascript
const any = ([head, ...tail], fn) =>
  head === undefined ? false : fn(head) || any(tail, fn);
```

❧&nbsp;❧&nbsp;❧

So what was the point of all that? Well, it took me awhile to work out why
you couldn't use `break` in Rust's `fold` method and JavaScript's `reduce`
method. It helps me to consolidate my understanding of things to explain them
to others, as I have tried to do here.

Also, I do reach for `fold` a lot, but the inability to `break` tends
to be a sign that a different abstraction might be preferable.

That said, I think something like my implementation of `isWinningGame` is the
best way to solve that kind of problem, and there `break`ing is an important
part of the implementation. I would simply extract a struct or class to
represent the complex "flag" used as the accumulator. People can certainly
disagree with that approach, but I think there's a lot of value in keeping
the instance of the flag integral to the looping construct outside of which
the flag makes no sense.
