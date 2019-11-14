---
title: "Nulls"
date: 2019-11-13T12:54:37-06:00
draft: true
---

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
