---
title: "Types Are for People"
date: 2019-11-06T12:59:01-06:00
draft: true
---

Types—in the static-typing sense—are for people. Oh sure, we use them, in
part, to subdue the compiler or meet some need peculiarly arising from our
computer. But types are valuable because they are a way of communicating.

## Type systems are a way of communicating.

Type systems are a way of announcing what you understand, expect, or intend.
Good type systems let you do so at the level of abstraction you choose. And
they let you describe these things based on what things _are_ or what they
_do_.

So if types are a way of communicating, who or what are you talking _to_?
First, you are talking to yourself—leaving notes. Next, you are talking to
the computer. You are asking it to hold you accountable to what you told it
you understood, expected, or intended. Finally, you are talking to other
developers (including future you).

## Good type systems remind us to think about things we might otherwise forget.

Consider this example, in which we deserialize some JSON into a struct and
then cast one of the values from a string to an integer:

````rust
use serde::Deserialize;

// Define a struct that has the shape of our JSON. By deriving the `Deserialize`
// trait from the `serde` library, Rust now knows how to take JSON and put it
// into this struct.
#[derive(Deserialize)]
#[serde(rename_all = "camelCase")] // JSON's camelcase -> Rust's snake case.
struct MyJsonFormat {
    /// A string that should be coercible to a number.
    stringy_number: String,
}

// In Rust, we have a `Result` type, defined:
//
// ```
// enum Result<T, E> { // `T` is the type for `Ok`; `E` is the type for `Err`
//   Ok(T), // Rust enum variants can hold values.
//   Err(E),
// }
// ```
//
// So a `Result` is either the `Ok` variant that wraps the value on success, or
// the `Err` variant that wraps the value on error.

/// Given a `json_string` parsable into `MyJsonFormat`, parse the
/// `stringy_number` field into an integer. Return an error if `json_string` is
/// not parsable into `MyJsonFormat` or if `string_number` is not parsable into a
/// an integer.
fn extract_number(json_string: &str) -> Result<i32, Box<dyn std::error::Error>> {
    // The `?` operator is a shorthand similar to a `try` block. If `serde_json`
    // returns an error, we short-circuit the function's evaluation and
    // immediately return the Error. Otherwise, we get the success value,
    // unwrapped from its `Ok()`.
    let deserialized: MyJsonFormat = serde_json::from_str(json_string)?;

    // This operation is *not* fallible. If we successfully deserialized the
    // JSON (the line above), we're *guaranteed* to have a field called
    // `stringy_number`. That's why there's no `?` or the like.
    let my_number = deserialized.stringy_number;

    // Once again, we have a fallible operation: parsing a string to an integer
    // can fail, so we have to use the `?` operator to either short-circuit with
    // an error or to unwrap the `Ok` if we succeed.
    let parsed: i32 = my_number.parse()?;

    // If we made it this far without returning an error, it's time to return
    // the integer wrapped in an `Ok` to signify success.
    Ok(parsed)
}

fn main() {
    // Try replacing "7" with "foo".
    // Try removing the "stringNumber" key.
    let json_string = "{\"stringyNumber\": \"7\"}";
    match extract_number(json_string) {
        Ok(num) => println!("Your number times 2 = {}", num * 2),
        Err(e) => println!("I had an error: {}", e),
    }
}
````

Notice that _every time_ something is fallible, we have to handle it. The
fallibility is built into the type: the success and failure cases are wrapped
in the enum variants `Ok` or `Err`. If you don't extract the value from it's
`Ok` box, you get a type error. If you only handle the `Ok` variant, the
compiler will catch you in one of several ways.

You can opt out of that behavior by using `unwrap()` on the `Result`
type. That panics if you call it on the `Err` variant. But `unwrap()`ing is
considered a faux pas in Rust most of the time.

## Static-typing detractors attack several strawmen.

### Types are not simply an anemic test suite.

A coworker argued that

> the extra one or two tests to confirm the inputs and outputs `respond_to`
> the correct duck typing are so easy to write, why not do it? We need tests
> for the business logic anyway so the "type" tests are about 2% of the effort.

### Type are not primarily about helping your computer.

There are times when type systems get in the way.

## The advantages of static typing are real because programmers are fallible and because better is better.

In an [Ask Me
Anything](https://dev.to/sandimetz/im-sandi-metz-ask-me-anything-4ff9), the
brilliant Sandi Metz wrote,

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

Metz also implicitly rejects the propositions that static typing is necessary
to prevent runtime errors, that programmers "cannot infer an object's type
from its context," and that static typing is necessary for optimal
performance. _Practical Object-oriented Design in Ruby: An Agile Primer_
101–02 (2013).

## Static type systems don't ruin duck typing or metaprogramming.

<!-- Fans of dynamic typing often make two arguments:

1. Static typing is an anemic test suite, and
2. Duck typing allows greater flexibility, because it focuses on behavior
   rather than type.

For example, the brilliant Sandi Metz writes:

> Applications may define many public interfaces that are not related to one specific class

Type systems conflate two things, in my view: what the computer needs versus what helps you write correct code.
In Rust, if you want to return a generic error, you sometimes have to return a type like Box<std::error::Error>. That’s a unmitigated pain in the ass: you have put something in a Box, which is a way of returning a pointer, which is necessary because functions have to have a size known at compile time, which would be impossible if the function returned any type that implemented the Error interface. You’re doing that because the computer needs it.
But static typing used to help the programmer is a huge boon. It improves support from tooling. It turns runtime errors into compile-time errors. It lets you ask the compiler or tooling to help you commit to things—for example, that you’re sure to have an object with a particular method on it. And a well-designed type system lets you make those commitments at the level of specificity that you need.

Really good type systems also require you to handle errors and null values, otherwise refusing to compile.

Types aren’t a poor man’s test suite. Types are a way of communicating your expectations of the typed thing. Communicating it to the compiler, which will tell you if you’re wrong. Communicating it to your tooling, which will help you and others find where it’s defined and how it’s expected to be used. Communicating it to other developers. Is it panacea? No. But the ability to define types and to use generics in order to describe expectations is richer than “just write a test for types or duck typing” suggests to me. -->

```

```
