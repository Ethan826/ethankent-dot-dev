---
title: "Limiting the scope of mutation"
date: 2019-11-05T15:05:29-06:00
draft: false
---

Functional programming teaches us about the dangers of mutability. But some
object-oriented code mutates variables freely, in different scopes, along
different code paths, and in many places.

To maintain our ability to reason about code, we can learn from functional
programming and treat mutation as a risk to be managed. Even if we stop short
of mandatory immutability and using constructs like monads, we can still
manage mutations and recognize them as a source of bugs and confusion.

I'm always nervous when I see code like this:

```ruby
class Unwieldy < GrabBag::Base
  include BizarroHelpers
  # ... 500 lines of spaghetti

  def do_something(wibble)
    result = []
    intermediate_result = []

    # ... 20 lines of code

    intermediate_result << weird_helper(wibble)

    # ... 20 lines of code

    intermediate_result << biscuits? ? gravy : grits

    # ... 20 lines of code

    if weirder_helper(wibble)
      intermediate_result.reject!(&:magic)
    else
      result << inexplicable_helper(wibble)
    end

    # ... 20 lines of code

    if confusing_predicate_method?
      result += intermediate_result
      return result
    end

    result << :deus_ex_machina
    result
  end

  # ... 500 lines of spaghetti
end
```

We might not be able to fix all the spaghetti code and we're sure to have
difficulty grokking what comes from where (the class inherits from
`GrabBag::Base` and mixes in `BizzaroHelpers`). But if we're writing
`do_something`, we can use a few rules of thumb to avoid getting too confused
and to make our code more maintainable.

To summarize the ideas presented below: Generally, avoid declaring a basic
object and repeatedly mutating it in a large scope. Prefer instantiating
objects using literals, other objects, and the builder pattern/fluent API.
Run the code with mutations in an attached scope, or at least right next to
the declaration. And avoid building logic into the interaction of control
flow with mutation.

## Keep mutation near the declaration.

The examples here use a Rust `HashMap`. I chose that because the standard
library doesn't give us a way to construct one using a literal.

The following Rust code has a few benefits: it keeps all mutations close
together and it immediately makes `lunch` immutable once the assigning is
done.

```rust
#![allow(unused_variables)]

use std::collections::HashMap;

fn main() {
    let mut lunch = HashMap::new();
    lunch.insert("salad", "house");
    lunch.insert("entree", "sandwich");
    lunch.insert("dessert", "cookie");
    lunch.insert("drink", "milk");
    let lunch = lunch; // `let` binding without `mut` makes `lunch` immutable.
}
```

## Better yet, put mutation in a scope attached to the declaration.

This is overkill here, but it demonstrates a point. Notice that the scope in
which the `HashMap` is mutable is only in the body of the closure passed to `fold`:

```rust
let lunch = [
    ("salad", "house"),
    ("entree", "sandwich"),
    ("dessert", "cookie"),
    ("drink", "milk"),
]
.iter()
.fold(HashMap::new(), |mut result, (k, v)| {
    result.insert(k, v);
    result
});
```

Ruby has some nice patterns with `Object#tap` and `Object#yield_self`, for
example:

```ruby
immutable = MyClass.new.tap |mutable| do
  mutable.blip = 8
  mutable.blop = 10
end
```

An instance of `MyClass` is mutated after being instantiated, but before it
is bound to `immutable`. We do that with a scope in which the instance of
`MyClass` is mutated, but `mutable` goes out of scope at the end of the `tap`
block; the result is now bound to `immutable` and—we hope—not mutated any
more. This doesn't actually prevent anyone from mutating `immutable` (that's
just a name, after all), but it makes our intent clear and helps the next
developer follow the same pattern.

## Better yet, use a literal or the builder pattern to instantiate the object.

A Rust crate called [`maplit`](https://github.com/bluss/maplit) exposes a
macro for creating literal hashmaps:

```rust
let lunch = hashmap! {
    "salad" => "house",
    "entree" => "sandwich",
    "dessert" => "cookie",
    "drink" => "milk",
};
```

If you have control over objects, you can allow them to be initialized with a
literal. For example:

```rust
use std::collections::HashMap;

// The "Newtype" pattern. Lunch is basically a `HashMap`, but we're making our
// own type as a thin wrapper over it (in Rust it's a struct, but it would be a
// class in many languages) so that we control its construction and which
// methods we expose.
struct Lunch(HashMap<String, String>);

impl Lunch {
    // You can ignore the `&str` and `to_string()` stuff for this example and
    // view them all as strings. It's a Rust quirk.
    fn new(init: Vec<(&str, &str)>) -> Self {
        let mut result = HashMap::with_capacity(init.len());
        for (k, v) in init {
            result.insert(k.to_string(), v.to_string());
        }
        Self(result)
    }
}

fn main() {
    let lunch = Lunch::new(vec![
        ("salad", "house"),
        ("entree", "sandwich"),
        ("dessert", "cookie"),
        ("drink", "milk"),
    ]);
}
```

Or you can use the builder pattern with an object you define:

```rust
#![allow(unused_variables)]

use std::collections::HashMap;

// You'd do this with named struct fields; just making this example more like
// the others.
struct Lunch(HashMap<String, String>);

impl Lunch {
    fn new() -> Self {
        Lunch(HashMap::with_capacity(4))
    }

    fn with_salad(mut self, salad: &str) -> Self {
        self.0.insert("salad".to_string(), salad.to_string());
        self
    }

    fn with_entree(mut self, entree: &str) -> Self {
        self.0.insert("entree".to_string(), entree.to_string());
        self
    }

    fn with_dessert(mut self, dessert: &str) -> Self {
        self.0.insert("dessert".to_string(), dessert.to_string());
        self
    }

    fn with_drink(mut self, drink: &str) -> Self {
        self.0.insert("drink".to_string(), drink.to_string());
        self
    }
}

fn main() {
    let lunch = Lunch::new()
        .with_salad("house")
        .with_entree("sandwich")
        .with_dessert("cookie")
        .with_drink("milk");
}
```

## Assign _from_ rather than _in_ conditional _expressions_.

In many languages, `if/else` is an expression rather than a statement. When
that is so, make use of this to assign the value of the `if/else` expression
rather than assigning within the `if`. That fits in with the rest of the
recommendations because it entails mutation in one scope that affects a larger
scope.

Thus, prefer

```ruby
my_result = if predicate?
              value_a
            else
              value_b
            end
```

to

```ruby
if predicate?
  my_result = value_a
else
  my_result = value_b
end
```

Among other reasons, it assures that `my_result` is always defined, and it
makes clear that the intent of the `if/else` is to bind a value to a
variable. (And of course, don't _also_ do some side-effecting thing in the
conditional. Remember the [command–query separation
principle](https://en.wikipedia.org/wiki/Command%E2%80%93query_separation).)

In JavaScript, ternaries are expressions while `if/else` blocks are
not. So you can use ternaries in lieu of `if/else` expressions in some
cases. But ternaries are probably best confined to small amounts of code
because the `?` and `:` operators are easy to lose track of.

## Do not embed logic in the interaction of flow control and mutation.

This is a general statement of the specific example of `if/then` statements
from the previous section. When your business logic is immanent from the
interaction of control flow and mutation, it can be quite difficult to reason
about your code, the code is more difficult to maintain, and bugs are more
likely.

The tongue-in-cheek example at the beginning is a case of this. A common
pattern is for a `result` variable to be mutated throughout a large function,
with some mutation occurring inside conditionals, and `result` sometimes
short-circuit returned. The author of the code may think he or she is helping
you out because you don't wind up deep within nested conditionals. Yet the
effect is often worse: the logic is just as confusing, but now the logic is
implicit: the state of your `result` variable is determined by the
combination of not having returned at line X, having entered the `else` block
at line Y, and so forth.
