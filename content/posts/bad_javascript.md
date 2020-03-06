---
title: Bad javascript language design choices
date: 2020-03-05T13:34:49-06:00
draft: false
author: Ethan Kent
---

The category _discussions about weird JavaScript things_ is a large and
amusing one, probably best exemplified by Gary Bernhardt's
[Wat](https://www.destroyallsoftware.com/talks/wat) lightning talk.

I'd like to focus on two I haven't seen discussed as much (though I may
simply have overlooked the canonical article). First, `const` doesn't work
the way it ought to, and second, `indexOf`'s magic number thing is godawful.

## `const` doesn't solve the problem it ought to solve.

Consider this situation:

```javascript
let myArray = [3, null, 5, 7, null, 10];
printArrayWithNoNulls(myArray);

// => null
// => null
// => null
// => null
// => null
// => null

console.log(myArray);
// => [ null, null, null, null, null, null ]
```

Wait, what happened!? Well, it turns out the method has a bug:

```javascript
const printArrayWithNoNulls = (arr) => {
    for(let i = 0; i < arr.length; ++i) {
        if (arr[i] = null) {
            continue;
        }
        console.log(arr[i]);
    }
}
```

Do you see it? We used `=` instead of `==` or `===`. Not only does the method
not work, we've also lost our data. 

In JavaScript, arrays are pass-by-reference, with the pointer allowing
mutation. Let's redefine our method so it requires a const reference:

```javascript
const printArrayWithNoNulls = (const *int arr) => { //...
```

Haha, just kidding. There's no such thing. Okay, but here's the real problem:
we should change our first line of code:

```javascript
const myBulletproofArray = [3, null, 5, 7, null, 10];
```

Let's do it again. Now at least we'll have a runtime error showing that we're
trying to mutate our `const` value:

```javascript
printArrayWithNoNulls(myBulletproofArray);
// => same output, no runtime error. Uh oh...

console.log(myBulletproofArray);
// => [ null, null, null, null, null, null ]
```

So, uhh, `const`, not helping very much, are you?

Okay, I'm going to make a protective copy, then go back to that if things go
south.

```javascript
const myArray = [3, null, 5, 7, null, 10];
const backupArray = [...myArray];

printArrayWithNoNulls(myArray);

if (myArray !== backupArray) {
    const myArray = backupArray;

    // => Thrown:
    // =>   TypeError: Assignment to constant variable.

    // Same eror without `const`
}

console.log(myArray);
```

Jeez JavaScript, _now_ you pipe up?

So that's my problem with `const`: I want it to be usable as a tool to
protect against the (mandatory) pass-by-reference semantics for non-scalar
values in JavaScript. The dangers in sharing a pointer to a mutable value are
well known: race conditions and inadvertent mutation. In Rust (and C++,
etc.), you must opt in to both pass-by-reference and mutability. In Rust you
have to mark both things in the function definition: the parameter is mutable
_and_ pass-by-reference. You must also mark the variable to be passed to the
function as mutable when you declare _it_, and _also_ mark that the argument
is mutable _and_ pass-by-reference at the call site:

```rust
// `&` means pass-by-reference, `mut` means mutable
fn print_array(arr: &mut Vec<i32>) {
    for val in arr {
        println!("{}", val);
    }
}

fn main() {
    // `mut` means mutable
    let mut my_array = vec![1, 2, 3];
    // `&` means pass-by-reference, `mut` means mutable
    print_array(&mut my_array);
}
```

Indeed, `const` in JavaScript affects what may be the _least_ important
aspect of mutability: it simply prevents you from rebinding a different value
to the same variable name. And indeed, in Rust, as long as you use `let`
again to show that you're declaring a new variable (albeit one with the same
name), you're fine:

```rust
let x = 7;
let x = 10; // No problem

let y = 15;
y = 20; // error: cannot assign twice to immutable variable `y`

let mut z = 20;
z = 30; // No problem
```

## `indexOf`'s magic number behavior is really, really atrocious.

Consider this:

```javascript
const myArray = ["foo", "bar", "baz"];

const containsValue = (arr, value) => !!arr.indexOf(value);

containsValue(myArray, "bar"); // => true
containsValue(myArray, "baz"); // => true
```

Excellent, ship it! Well, let's try to see if we get any false positives:

```javascript
containsValue(myArray, "quux"); // => true
```

Oh, shoot, looks like our code _always_ returns `true`.

But wait, we overlooked one test case:

```javascript
containsValue(myArray, "foo"); // => false
```

We have written a function that returns `true` in all cases except when we
look for the first element of the array.

Why? It has to do with `indexOf` returning a number—the index—not a boolean.

What happens if a value is missing from the array? Given the foreseeable use
cases, together with the fact that all JavaScript values can be implicitly
coerced to booleans, surely care has been taken to ensure that the value
returned is falsey. So does `indexOf` return `null` to indicate that there is
no index that has our number? Or does it return `NaN` to signify that there
is no number that can index us to where we want to go? Or `false` itself, to
signify that it is `false` that the array has the value? All of those would
coerce to false.

No, none of those. It's a number.

Okay, well I've got it then: of the 18,437,736,874,454,810,623 distinct
numbers that can be expressed in JavaScript,[^numbers] precisely one is
falsey: `0`. That has to be it, right?

Nope, can't be: the item could be at the zeroth position of the array. It's
starting to sound like this _shouldn't_ be a number. What number is it,
though? The answer is `-1`. As in, a truthy value. As in, an essentially
random magic number. As in, a number whose only benefit is that it is not a
valid index into an array. But of course that is also the case for _all_
negative numbers, all floating-point numbers, and for `null`, `undefined`,
`NaN`, and `false`.

Now that magic constant, `-1`, has to leak into _our_ code, which we
otherwise endeavor to keep free of magic constants. And if we assume that
since number coerce to booleans, we can rely on getting the right behavior
for the _value not found_ case, we're going to introduce a bug. Grotesque.

[^numbers]: Jeffrey Sax, Answer, _How many distinct values can be stored in floating-point formats?_, Stack Overflow, https://stackoverflow.com/a/7744178/3396324 (Oct. 12, 2011).