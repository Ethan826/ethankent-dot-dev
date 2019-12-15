---
title: "I fold!"
date: 2019-12-08T14:14:00-06:00
author: Ethan Kent
draft: false
---

This article is a love letter to `fold`, a.k.a. `reduce`, a.k.a.
`inject`.[^1]

[^1]: It is unclear to me why Ruby thinks all of the big higher-order functions must end in _–ect_: `collect` (`map`), `select` (`filter`), `inject` (`fold`/`reduce`), and `reject` (like `filter`, but retain only values for which the closure returns `false`). I will admit that `reject` is pretty useful and well named, but those other ones seem pretty bloody-minded—particularly `collect`, a name much better suited to the way Rust [uses it](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.collect).

I love fold for three reasons. First, it is useful in a wide variety of
programming contexts. Second, it is a ladder between different levels of
abstraction: it is simple to define, but allows operating on collections more
abstractly. Third, it can have the same API but be defined based on looping
or recursion, and so can help you solve similar problems in similar ways
despite coding in different paradigms.

I'll explain what I mean in the opposite order. I will not take up the topic
of folding infinite collections, laziness, `foldl` vs. `foldr`, etc.; perhaps
I will write about these ideas another time.

## `fold` provides the same abstraction across programming paradigms.

We can define `fold` iteratively or recursively. Let's define `fold` in each
way (not necessarily the most efficient or with all the bells and whistles):

```javascript
const iterativeFold = (array, fn, initialAccumulator) => {
  let accumulator =
    typeof initialAccumulator === "undefined" ? [] : initialAccumulator;
  for (const element of array) {
    accumulator = fn(accumulator, element);
  }
  return accumulator;
};

const recursiveFold = ([head, ...tail], fn, initialAccumulator) => {
  let accumulator =
    typeof initialAccumulator === "undefined" ? [] : initialAccumulator;
  return head === undefined
    ? accumulator
    : recursiveFold(tail, fn, fn(accumulator, head));
};
```

Notice that although `recursiveFold` makes its recursive call from the tail
position, most JavaScript engines don't support tail-call
optimization.[^2] But that would be a necessary feature in order
to make the recursive definition usable with large collections.

[^2]: _See_ Juriy Zaytsev, Leon Arnott, Denis Pushkarev, [ECMAScript 6 compatibility table](http://kangax.github.io/compat-table/es6/#test-proper_tail_calls_%28tail_call_optimisation%29) (last visited Dec. 5, 2019).

## `fold` can be used to define all the other important higher-order functions.

Now that we have two definitions of `fold` with the same API and equivalent
logic, we can use that to define other important higher-order functions.

```javascript
// Assume that `fold` is now the name of whichever of the above implementations
// you want. Also notice that we're ignoring the perfectly good `reduce` method
// on JavaScript iterables and are defining one ourselves.

const map = (array, fn) => {
  return fold(array, (accumulator, element) => {
    // Rebuild our array with every element after applying `fn`
    accumulator.push(fn(element));
    return accumulator;
  });
};

const filter = (array, fn) => {
  return fold(array, (accumulator, element) => {
    // Rebuild our array with only those elements that return truthy values when
    // `fn` is called with them.
    if (fn(element)) accumulator.push(element);
    return accumulator;
  });
};

const all = (array, fn) => {
  return fold(
    array,
    // Accumulator is initialized to `true`. If we ever encounter an element
    // that causes `fn` to return a falsey value, we coerce that value to
    // Boolean and replace the accumulator with that. Then, no matter how many
    // truthy values we later encounter, the accumulator is now locked in as
    // `false`, which is the appropriate behavior for `all`.
    (accumulator, element) => accumulator && !!fn(element),
    true,
  );
};

const any = (array, fn) => {
  return fold(
    array,
    // Accumulator is initialized to `false`. If we ever encounter an element
    // that causes `fn` to return a truthy value, we coerce that value to
    // Boolean and replace the accumulator with that. Then, no matter how many
    // truthy values we later encounter, the accumulator is now locked in as
    // `true`, which is the appropriate behavior for `any`.
    (accumulator, element) => accumulator || !!fn(element),
    false,
  );
};
```

My [next article](https://ethankent.dev/breaking_fold/) will deal with the problem attentive readers may notice:
these `any` and `all` functions don't short-circuit; we can reduce the
constant factor associated with our time complexity if we allow these
functions to `break`. The problem, fundamentally, is that we can't reach from
within the closure to `fold` to cause the looping in `fold` itself to break.

## `fold` is useful in a wide variety of circumstances.

When I need to operate on a collection, I generally use this rule of thumb:

- Do I want to return the same collection type with the same number of
  elements, but with the elements themselves changed? Use `map`.
- Do I want to return the same collection type with a subset of the original
  elements? Use `filter`.
- Do I want to do something else (e.g. return a different type, return a
  collection with a subset of the original elements and with the elements
  changed, iterate with a flag or intermediate result, etc.)? Use `reduce`.

### `fold` is useful to combine multiple transformations.

In some languages, calling `filter` and then `map` requires two iterations of
the collection.[^3] While this does not change the runtime complexity from a
Big-O perspective, (`O(n)` == `O(2n)`), constant factors are often relevant
in practical applications.

[^3]: This is the power of `collect()` in Rust and `lazy` in Ruby.

Consider this code:

```javascript
/**
 * From an array of integers, return only the even integers, divided by two.
 *
 * @param {Integer[]} numbers - The integers to operate on.
 * @returns {Integer[]} The even integers only, divided by two.
 */
const halveEvens = numbers => {
  let iterationSteps = 0;
  return numbers
    .filter(number => {
      console.log(`Inside filter; on iterationStep ${++iterationSteps}`);
      return number % 2 === 0;
    })
    .map(number => {
      console.log(`Inside map; on iterationStep ${++iterationSteps}`);
      return number / 2;
    });
};

halveEvens([1, 2, 3, 4, 5, 6]);

// Inside filter; on iterationStep 1
// Inside filter; on iterationStep 2
// Inside filter; on iterationStep 3
// Inside filter; on iterationStep 4
// Inside filter; on iterationStep 5
// Inside filter; on iterationStep 6
// Inside map; on iterationStep 7
// Inside map; on iterationStep 8
// Inside map; on iterationStep 9
// => [ 1, 2, 3 ]
```

This code logs 9 iteration steps.

```javascript
/**
 * From an array of integers, return only the even integers, divided by two.
 *
 * @param {Integer[]} numbers - The integers to operate on.
 * @returns {Integer[]} The even integers only, divided by two.
 */
const halveEvens = numbers => {
  let iterationStep = 0;
  return fold(numbers, (accumulator, element) => {
    console.log(`Inside fold; on iteration step ${++iterationStep}`);
    if (element % 2 === 0) {
      accumulator.push(element / 2);
    }
    return accumulator;
  });
};

halveEvens([1, 2, 3, 4, 5, 6]);

// Inside fold; on iteration step 1
// Inside fold; on iteration step 2
// Inside fold; on iteration step 3
// Inside fold; on iteration step 4
// Inside fold; on iteration step 5
// Inside fold; on iteration step 6
// => [ 1, 2, 3 ]
```

This code logs 6 iteration steps, the minimum possible.

### `fold` is useful when we want to operate on a collection to return a different type.

To sum the elements of an array, we can use `fold`:

```javascript
const sum = numbers => {
  return fold(numbers, (total, number) => total + number, 0);
};

sum([1, 2, 3, 4, 5, 6, 7, 8, 9, 10]);

// => 55
```

### `fold` is useful when we must iterate with a flag or intermediate result.

We can implement `max` and `min` with `fold` by making the max or min so far
encountered take the place of the accumulator.

```javascript
const max = numbers => {
  return fold(
    numbers,
    (bestSoFar, current) => (current > bestSoFar ? current : bestSoFar),
    -Infinity,
  );
};

max([3, 0, 10, 2, 1001, -1000, 1000]);

// => 1001
```

Or, imagine a game in which you must collect one `tinyBooomer` or two
`megaBoomer`s to win.^[To make this efficient we'd want to be able to
short-circuit, which will be the subject of the next article.] A `nothing`
does nothing and a `whammy` makes you lose, assuming you haven't yet won.

There are refactors to be done by extracting some classes, etc., but here's
an illustration of what I mean:

```javascript
const tinyBoomer = Symbol("tinyBoomer");
const megaBoomer = Symbol("megaBoomer");
const nothing = Symbol("nothing");
const whammy = Symbol("whammy");

const isWinningGame = gameTape => {
  return fold(
    gameTape,
    (status, event) => {
      const gameOver = status.hasWon || status.hasLost;
      if (!gameOver) {
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
      return status;
    },
    {
      hasWon: false,
      hasLost: false,
      tinyBoomerCount: 0,
    },
  ).hasWon;
};

const winner = [tinyBoomer, nothing, nothing, tinyBoomer, whammy];
const loser = [tinyBoomer, nothing, whammy, tinyBoomer, megaBoomer];

isWinningGame(winner);
isWinningGame(loser);

// => true
// => false
```

### Attach a mutable scope

You can use `fold` to help attach a scope for mutating an object in a scope
that ends when you're done mutating it. That was the topic of [another
article](https://ethankent.dev/posts/mutation_scopes/#better-yet-put-mutation-in-a-scope-attached-to-the-declaration).

❧&nbsp;❧&nbsp;❧

I will admit to overusing `fold` a bit. But I hope you can see why its
versatility is appealing, both from a pragmatic and an aesthetic perspective.
