---
title: "Limiting the scope of mutation"
date: 2019-10-25T11:57:11-05:00
draft: true
---

Functional programming teaches us about the dangers of mutability. Some
object-oriented code mutates variables freely, in different scopes, along
different code paths, and in many places.

To maintain our ability to reason about code, we can learn from functional
programming and treat mutation as a risk to be managed. Even if we short of
mandatory immutability and using constructs like monads, we can still manage
mutations and recognize them as a source of bugs and confusion.

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

    result
  end

  # ... 500 lines of spaghetti
end
```

We might not be able to fix all the spaghetti code and we're sure to have
difficulty grokking what comes from from where (the class inherits from
`GrabBag::Base` and mixes in `BizzaroHelpers`). But if we're writing
`do_something`, we can use a few rules of thumb to avoid getting too confused
and to make our code more maintainable.

## Instantiate objects with all necessary data.

Prefer instantiating objects using literals, other objects, the builder pattern, or a fluent API. Avoid declaring a basic object and repeatedly mutating it.

## Keep mutation near—or attached to—declaration.

Keep mutation as close as possible to where that object is declared—ideally,
inseparable from the declaration.

## Assign from rather than in conditional expressions.

## Do not embed logic in the interaction of flow control and mutation.

This is a general statement of the specific example offered above.

_within_ conditionals, assuming your language has `if` expressions rather
than `if` statements.
