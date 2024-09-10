+++
title = 'Obligatory Monad explainer'
date = 2024-09-10T02:38:18.818Z
draft = false
+++

> In addition to its being good and useful, it’s also cursed and the curse of
> the monad is that once you get the epiphany, once you understand—"oh that's
> what it is"—you lose the ability to explain it to anybody else.

– [Douglas Crockford](https://youtu.be/dkZFtimgAcM&t=42)

> It is a truth universally acknowledged, that a programmer in possession of a
> rudimentary knowledge of monads, must publish a dev blog post on that topic.

– Jane Austen, more or less.

# The idea of a "thing you can `map()` over"

The superpower of the FP community is spotting abstracter abstractions than
everybody else. This post's examples are in TypeScript, and I describe the FP
concepts as they exist in Haskell.

Let's start this monad explainer using `map()`:

```ts
[1, 2].map((x) => 2 * x);
// => [2, 4]
```

Array's `map()` lets us perform a function on each element of the array. Notice
that the `map()` method knows things about arrays. We don't have to do stuff
with `for` loops or any of that. By contrast, the function we pass to `map()`,
namely—

```ts
(x) => 2 * x;
```

doesn't know stuff about arrays.

Keep that in mind as we look at promises:

```ts
const myPromise = Promise.resolve(3);
const result = await myPromise.then((x) => 2 * x);
console.log(result);
// => 6
```

Same concepts: the `then()` method knows stuff about promises. We don't have to
do stuff with the microtasks queue or any of that. By contrast, the function we
pass to `then()`, namely—

```ts
(x) => 2 * x;
```

doesn't know stuff about promises.

Remember how I said that the FP smarties' superpower is abstracter abstractions?
They noticed the usefulness of this general concept of _a thing that you can
`map()` over_: that's called a functor.

Let's just make up a type that has this functor-y-ness:

```ts
class Box<A> {
  constructor(private value: A) {}

  public map<B>(theFunction: (a: A) => B) {
    return new Box(theFunction(this.value));
  }
}

const box = new Box(10);

console.log(box.map((x) => 2 * x));
// => Box: { "value": 20 }
```

# The idea of a "thing you can `flatMap()` over"

Now let's consider mapping a function over an array, but that function also
returns an array. Let's say we have a function that assigns roles to a user (in
an admittedly contrived way):

```ts
const assignRoles = (user: string) =>
  ["admin", "editor", "viewer"].map((role) => ({ user, role }));
```

This works fine for a single user:

```ts
assignRoles("Ethan");
// => [
//      { user: "Ethan", role: "admin" },
//      { user: "Ethan", role: "editor" },
//      { user: "Ethan", role: "viewer" },
//    ];
```

But let's say we want to create an array with all permissions for several users
for later processing:

```ts
assignRoles(["Anita", "Brad", "Cindy"]);
// => [
//      [
//        { user: "Anita", role: "admin" },
//        { user: "Anita", role: "editor" },
//        { user: "Anita", role: "viewer" },
//      ],
//      [
//        { user: "Brad", role: "admin" },
//        { user: "Brad", role: "editor" },
//        { user: "Brad", role: "viewer" },
//      ],
//      [
//        { user: "Cindy", role: "admin" },
//        { user: "Cindy", role: "editor" },
//        { user: "Cindy", role: "viewer",
//        },
//      ],
//    ];
```

Whoops, we didn't account for the nested arrays. Well, don't worry: recent JS
runtimes give us a `flatMap()` method:

```ts
["Anita", "Brad", "Cindy"].flatMap(assignRoles);
// => [
//      { user: "Anita", role: "admin" },
//      { user: "Anita", role: "editor" },
//      { user: "Anita", role: "viewer" },
//      { user: "Brad", role: "admin" },
//      { user: "Brad", role: "editor" },
//      { user: "Brad", role: "viewer" },
//      { user: "Cindy", role: "admin" },
//      { user: "Cindy", role: "editor" },
//      { user: "Cindy", role: "viewer" },
//    ];
```

Pretty straightforward. This function just makes sure that the outer and inner
arrays get smashed together.

Now let's implement `flatMap()` for our `Box` type:

```ts
class Box<A> {
  constructor(private value: A) {}

  public map<B>(theFunction: (a: A) => B) {
    return new Box(theFunction(this.value));
  }

  public flatMap<B>(theFunction: (a: A) => Box<B>) {
    return theFunction(this.value);
  }
}

const boxedDoubler = (x: number) => new Box(2 * x);

const box = new Box(10);

// Whoops, inadvertently nested
console.log(box.map(boxedDoubler));
// => Box: { "value": { "value": 20 } }

// Ahh, that's better
console.log(box.flatMap(boxedDoubler));
// => Box: { "value": 20 }
```

Remember how I said that the FP smarties' superpower is abstracter abstractions?
They noticed the usefulness of this general concept of _a thing that you can
`flatMap()` over_: that's called a monad.

Mic drop.

# Some loose ends

You've now gotten the minimum viable monad explainer. The rest is optional
reading.

## So what?

If you're not a math or FP theory person, this may seem bloody-minded. Why in
the world do I care? I can give two reasons in brief. I may write a follow-up
post on this.

- Spotting abstractions is useful. If your type has a functor instance, I know I
  can give you a function to work with the data inside your type and you'll deal
  with the plumbing of calling my function on the data inside your type. You'll
  figure out how to iterate your array, or await your async code, or whatever
  your type is all about.
- Monads in particular allow sequencing computations, with each subsequent
  computation able to use the results of the previous computation. This provides
  the necessary framework to abstract over side effects and impurity in a way
  that, as if by magic, lets you write pure code to work with impure data. This
  is a deep subject.

## A little bit to assuage the FP nerds

This is the simplest version of the main idea of a monad: a type that you can
`flatMap()` over. But I'm of course leaving out some details.

Also, `flatMap()` is called `bind` and looks like `>>=`, and its arguments are
in a different order, and it looks like a function call.

```haskell
-- Double a number and wrap the result in a list
doubleInsideList x = [2 * x]

-- In Haskell, `>>=` (bind) applies the doubleInsideList function and flattens
-- the result, similar to `flatMap()` in other languages.
[1, 2] >>= doubleInsideList
-- [2, 4]
```

Here are a few more nuances.

### Type classes

What is the sort of thing that a functor and a monad are? In Haskell, they're
called _type classes_. The terminology here is, "array has an instance of the
functor type class," "my type `Box` has an instance of the functor type class,"
etc.

A strong instinct for type classes comes from an interface in an OOP language.
But if you experiment with trying to actually do an interface for functors in
TypeScript, you'll realize you don't have a spot to put the two "layers" of
generic parameters: a functor needs to be generic over the type (array, `Box`)
and also the type _in the type_ (an array of numbers, a `Box` with a string).
You just can't do that with off-the-shelf TypeScript. The feature that's missing
is called _higher-kinded types_, which Haskell has.

### Hierarchies

Monads also have to be functors. In OOP we'd say that monad _extends_ functor,
but in FP we'd say functor is a superclass of monad. We're also leaving out a
thing called applicative: all monads are applicatives, all applicatives are
functors.

### Additional methods

Monad also has a method for putting something in the type, which in Haskell can
be called `pure`. You can think of it as a constructor for the type.

```ts
class Box<A> {
  private constructor(private value: A) {}

  // Technically `pure` comes from applicative, and monad, for historical
  // reasons, has a method called `return` that does the same thing. Okay, now
  // forget I said anything.
  public static pure<A>(value: A): Box<A> {
    return new Box(value);
  }

  public map<B>(theFunction: (a: A) => B) {
    return new Box(theFunction(this.value));
  }

  public flatMap<B>(theFunction: (a: A) => Box<B>) {
    return theFunction(this.value);
  }
}
```

### Laws

In addition to what I've already said, type classes also have "laws." These
express invariants that the language can't directly enforce.

For example, a functor isn't lawful if, when you `map()` the `id` function (`a
=> a`), you get something different back. That one is called the _identity law_
(they also have a _composition law_).

When implementing a type class for a type, implementers must make sure they
follow the appropriate laws, or code using that type class may do weird stuff.

## For promises, `then()` is both `map()` and `flatMap()`

What about promises—are they monads or functors? It turns out promises don't let
themselves get nested. The JS runtime just sands off this rough edge.

```ts
const asyncDoubler = (x: number) => Promise.resolve(2 * x);
const myPromise = Promise.resolve(3);

const result = await myPromise.then(asyncDoubler);
console.log(result);
// => 6
```

So actually, `then()` is like `map()` and `flatMap()`. It is not a lawful monad,
though it is akin to a lawful functor.
