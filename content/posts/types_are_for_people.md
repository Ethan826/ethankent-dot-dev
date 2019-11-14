---
title: Types Are for People, not Computers
date: 2019-11-14T09:00:00-06:00
draft: false
---

Types—in the static-typing sense—are useful because they help people, not
computers. Oh sure, we use them, in part, to subdue the compiler or meet some
need peculiarly arising from our computer. But types are valuable because
they are a way of communicating.

## Type systems are a way of communicating.

Type systems are a way of announcing what you understand, expect, or intend.
Good type systems let you do so at the level of abstraction you choose. And
they let you describe these things based on what things _are_ or what they
_do_.

So if types are a way of communicating, who or what are you talking _to_?
Several audiences. You are talking to yourself—leaving notes. You are also
talking to the computer—but often for your sake rather than its. You are
asking it to hold you accountable to what you told it you understood,
expected, or intended. Finally, you are talking to other developers
(including future you).

## We can match how we use the type system with what we understand, expect, or intend of our objects.

Consider this JavaScript code:

```javascript
const mathItForMe = (a, b) => {
  return { added: a + b, subtracted: a - b, multiplied: a * b, divided: a / b };
};

mathItForMe(5, "3");
// => { added: '53', subtracted: 2, multiplied: 15, divided: 1.6666666666666667 }

mathItForMe(true, 7);
// => { added: 8, subtracted: -6, multiplied: 7, divided: 0.14285714285714285 }

mathItForMe("3", true);
// => { added: '3true', subtracted: 2, multiplied: 3, divided: 3 }
```

In what programming context is the desired behavior for using `+` with a
string and a boolean to coerce the boolean into the string `true` or `false`
and concatenate it to the string? It's madness. But then again, all implicit
coercion is the same kind of insanity; some instances are just more florid
than others. Using types imposes some discipline:

```rust
#[derive(Debug, PartialEq)]
struct GotMathed {
    added: i32,
    subtracted: i32,
    divided: i32,
    multiplied: i32,
}

fn math_it_for_me(num_1: i32, num_2: i32) -> GotMathed {
    GotMathed {
        added: num_1 + num_2,
        subtracted: num_1 - num_2,
        divided: num_1 / num_2,
        multiplied: num_1 * num_2,
    }
}

fn main() {
    let x: i32 = 7;
    let y: i32 = 5;

    assert_eq!(
        math_it_for_me(x, y),
        GotMathed {
            added: 12,
            subtracted: 2,
            divided: 1,
            multiplied: 35,
        }
    );
}
```

But notice that we aren't necessarily capturing our intent. What if, instead
of a signed 32-bit integer (an `i32`), we use an unsigned, 64-bit integer (a
`u64`)? Or a 32-bit float (an `f32`)? Our function doesn't work—but it
probably should. This function isn't _about_ doing math on 32-bit integers,
it's about doing math on numbers.

The type system has helped us figure out what our function ought to do by
requiring us to describe it explicitly. Here we've gone too narrow. This is
worlds better than a function that takes types we didn't even consider, like
the JavaScript example. The behavior there was pathological because we were
too accepting.

But now the type system is putting pressure on us to think about exactly what
this function ought to do. If it were part of a private API—a private method
in a class or a non `pub` function in an `impl` block in Rust—we might accept
that it places arbitrary limits on the types it accepts in order to
streamline implementation or avoid premature optimization. Plus, we would
have additional context and the method's restricted visibility means we
wouldn't be beholden to others who rely on the code.

```rust
use core::ops::{Add, Div, Mul, Sub};

#[derive(Debug, PartialEq)]
struct GotMathed<T> {
    added: T,
    subtracted: T,
    divided: T,
    multiplied: T,
}

fn math_it_for_me<T>(num_1: T, num_2: T) -> GotMathed<T>
where
    T: Add<Output = T>    // Specify "trait bounds": A trait is like an
        + Sub<Output = T> // interface. It specifies the behavior (method
        + Mul<Output = T> // signatures) a type must implement. Here we're
        + Div<Output = T> // saying `T` is a type that must implement addition,
        + Copy            // subtraction, multiplication, and division, in each
        + Clone           // case returning the same type. It must also
        + PartialEq,      // allow the value to be copied and cloned, and
{                         // compared for equality (used by `assert_eq!` ↓).
    GotMathed {
        added: num_1 + num_2,
        subtracted: num_1 - num_2,
        divided: num_1 / num_2,
        multiplied: num_1 * num_2,
    }
}

fn main() {
    let x: f32 = 7.0;
    let y: f32 = 5.0;

    assert_eq!(
        math_it_for_me(x, y),
        GotMathed {
            added: 12.0,
            subtracted: 2.0,
            divided: 1.4,
            multiplied: 35.0,
        }
    );
}
```

Here we have brought our intent, expectations, and understanding into
alignment. We are communicating to the computer what our function needs to be
able to do and how we intend to do it; the computer will now make sure that
we are able to do what we promised with the types our function will accept.
We have put ourselves in a position where the computer can help us. We are
also communicating to other people what our function is _for_ based on what
kinds of arguments are acceptable and what they can expect it to return.

Note that we are now describing what kinds of arguments are appropriate in
_precisely_ the terms that make sense. We're no longer talking about
arbitrary concretions: we are talking about the arguments at exactly the
level of abstraction that ought to define them.^[_Cf._ the [Dependency
Inversion
Principle](https://en.wikipedia.org/wiki/Dependency_inversion_principle).]

We have specified the arguments that `math_it_for_me` takes in terms of
_behavior_, and in so doing have aligned our intent with our understanding
and expectations. We could alter the code so that `num_1` and `num_2` could
be different numeric types, but that would force us to consider edge cases
and typecasts, and so it is reasonable to limit our code the way we have, out
to the limit of undefined or tricky behavior that we're not interested in
dealing with. And all of these positive changes were driven by the type
system's pressure to define things explicitly, together with its allowing
us to describe acceptable arguments at the right level of abstraction.

## Good type systems remind us to think about things we might otherwise forget.

Type systems with `Option`/`Maybe` and `Result`/`Either` are a good example
of how type systems can force us to handle all possible results. For example, Rust
defines the `Option` type:

```rust
enum Option<T> {
    // Enums in Rust can contain values. Here the `Some` variant contains a
    // value of the generic type `T`.
    Some(T),
    None
}
```

There is no `null` in Rust. A function that can fail to return a value of
type `T` must return an `Option<T>` (or similar) rather than a `T`. If you
write a function that is supposed to return a `T` but there is any code path
in the function that would fail to return a `T`, the compiler will yell at
you for not satisfying the function's type signature. If the caller of that
function tries to treat its return type like a `T` rather than an
`Option<T>`, you will likely get a type error.^[If you just so happened to
treat your `Option<T>` in a way that is fully compatible with `T`, the type
system would not catch it.]

So unless you follow the antipattern of calling `unwrap()` or `expect()` with
your `Option` type,^[If you call either with a `None` type, you get a panic,
which is like an unhandled exception.] the compiler will hound you until you
have dealt with the `null` case. Libertine Rubyists and Pythonistas may
object and say that they always handle their `nil`s. But that's not what the
runtime logs say for any of the applications I've worked on in any of my
programming jobs.^[[Rich Hickey](https://www.youtube.com/watch?v=YR5WdGrpoug)
doesn't like this pattern, and TypeScript handles nullability a [different
way](https://www.typescriptlang.org/docs/handbook/advanced-types.html#nullable-types),
but at bottom any alternative approach should require handling the `null`
case.]

## Specifying types in terms of interfaces, traits, or typeclasses is the same thing as duck typing, except it's harder to screw up.

In dynamically typed languages, duck types

> are abstract; this gives them strength as a design tool but this very
> abstraction makes the duck type less than obvious in code. [¶] When you
> create duck types you must document _and_ test their public interfaces.
> Fortunately, good tests are the best documentation, so you are already
> halfway done; you need only write the tests.[^implicit-duck]

[^implicit-duck]: Sandi Metz, _Practical Object-oriented Design in Ruby: An Agile Primer_ 98 (2013).

_Pace_ the brilliant Sandi Metz, types are better documentation than tests,
at least for the kinds of things types help us with. Unlike tests, types are
nearby, facilitate our tooling, and provide a consistent means of
communication. Tests and types are both better than documentation, but types
are documentation attached to our code like PostIt notes, while tests are
like books down in the stacks that we have to go looking for, hoping that the
card catalog will help us find them.

Elsewhere, Metz writes,^[[I'm Sandi Metz, Ask Me
Anything!](https://dev.to/sandimetz/im-sandi-metz-ask-me-anything-4ff9),
(Jan. 24, 2018).]

> [I]n my code, I don't get run-time type errors. Because my experience is
> that dynamic typing is perfectly safe, I find myself resenting having to
> enter type annotations when I work in statically typed languages. While I
> appreciate the fact that static typing means that I don't have to write some
> tests, I hate having to add this extra code.
>
> Having said that, I realize that many folks have had different experiences
> with dynamic typing. I write trustworthy code where objects behave like you'd
> expect. This means that I can trust that any object with which I'm
> interacting just works. This, in turn, means that I don't have to check if
> objects behave the right way. Sadly, I've seen many OO applications where
> these things were not true. Folks fall into the trap of writing code that's
> not trustworthy. Because they can't trust message sends to return objects
> that behave correctly, they have to check the type of the return of messages
> sends. This leads to a descending spiral of manually adding type checking,
> and code which ultimately breaks in confusing and painful ways.

I'm not as good a programmer as Sandi Metz is, and you probably aren't,
either. It may be like the case of the great athlete who can't coach
inferior talents because she didn't have to learn to work with the same
limitations.

But even if your code is Metzian in its brilliance, implicit interfaces are
like verbal contracts: a fine way to get confused about what has been
promised. Worse, the absence of a rigorously and explicitly defined for your
interface means your computer is powerless to help you get it right.
Remember, types are _for people_.

## Types are not a panacea.

There are some limitations to type systems, and I should acknowledge them to
show that I am not enamored of a caricature of types. The problems are real,
but they are worth it.

### Sometimes types aren't for people; sometimes they are about the programmer helping the computer.

In Rust, if you have a function that can return more than one error type, you
must in some cases put the error in a
[`Box`](https://doc.rust-lang.org/rust-by-example/error/multiple_error_types/boxing_errors.html).
That is because different errors are different sizes, and the return type of
a Rust function must be of a size known at compile-time. So the indirection
of a `Box` (which is a pointer) means that the compiler knows how big the
return value is. You have to help the compiler know what's going on—you're
helping the computer rather than the reverse.

This is annoying. But even Ruby and Python aren't written in English
sentences. Semicolons and curly braces are mostly there to help the computer,
too. Just because types are _sometimes_ for computers doesn't mean than types
aren't mostly useful for people.^[The title "Types Are Mostly for People,
Though Occasionally for Computers, Which can Be a Pain" seemed less punchy.]

### Types can't help you with all your business logic.

I once used a `u8` (an 8-bit, unsigned integer) to represent people's ages.
Because people haven't lived past 122, 255 seemed like plenty of room. Fair
enough. But I also saw it as a way of embedding business logic into the type.
It just so happens that 255 is not an insane limit for validating a person's
age. If you input `-50` or `3000`, you have probably made a mistake.

That got me thinking: What if I could somehow define a 7-bit unsigned
integer. Then I could use the type system to do a kind of validation.

I now see that this was colossally wrongheaded. This is mixing levels of
abstraction in the wrong way. It's like noticing that motor oil and coffee
are about the same color, so they should be interchangeable to keep your car
running and to perk you up in the morning.

Types are not for fine-grained validation. They're not a replacement for
tests.

But when you use them for their proper purposes, types can help you talk to
yourself, your computer, and other people; they can catch certain kinds of
errors; and they can put pressure on you to think, design, and communicate
more clearly. That's enough to make them a great tool that most programmers
should use.
