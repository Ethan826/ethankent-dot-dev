---
title: Desire paths in code
date: 2020-07-28T07:31:48-05:00
author: Ethan Kent
draft: false
---

Have you noticed those bootleg paths that shortcut between two sidewalks?
They're common in parks and on campuses. They're called [_desire
paths_](https://en.wikipedia.org/wiki/Desire_path). They show where the
sidewalk _should have been_ (sometimes the park or school will actually give
in and pave them, leading to strange results when seen on a
[map](https://www.google.com/maps/@35.912851,-79.052018,423m/data=!3m1!1e3)).

Programming languages have desire paths, too, and I'd argue they demonstrate
shortcomings in those languages. Indeed, people often claim that a language
doesn't need a particular feature, but the desire paths in their code
demonstrate otherwise. Thus the old saying, "Don't listen to what people say;
watch what they do."

I'll give two examples from Ruby and one from JavaScript.

## `NotImplementedError` as a desire path for interfaces

Ruby's `NotImplementedError` is _not_ a marker for an abstract method on a
base class. It means "a feature is not implemented _on the current
platform_."[^meaning]

[^meaning]: Class: NotImplementedError, https://ruby-doc.org/core-2.7.1/NotImplementedError.html (last visited July 28, 2020) (emphasis added).

Hardcore duck-typers will tell you that it's unnecessary to enforce
interfaces and that one of the beauties of Ruby is that you don't have to go
around writing boilerplate to do so.

And yet, numerous Ruby codebase misuse `NotImplementedError` as a way to
enforce interfaces. Don't listen to what people say; watch what they do.
People want interfaces.

## Symbols as a desire path for enums

Ruby symbols are everywhere in the language. Sometimes they are unbounded.
For example, if we `public_send` a symbol in the context of dynamic method
dispatch, we might not know the methods on an object until runtime—heck, they
may not even _exist_ until runtime, as they could be built from an impure
source such as a database, timestamp, or random value.

But at other times, symbols are part of a known universe of values. In that
situation, the Ruby way is to write tests to exercise all the variants. But
because symbols sometimes have long names, and because spelling is not every
coder's strong suit, it's easy to typo a symbol's name. Because of Ruby's
groovy, anything-goes attitude, tooling support for typo-d symbols is
difficult or impossible.

And so sometimes ad hoc double-checking occurs. If 14 enum values are
defined, it can be hard to tell whether that 273-line test file (located in
an entirely different part of the project) exercises every variant. So the
`case` statement's default branch throws an error.

Sometimes, contrivances like this occur:

```ruby
module Infielders
  Pitcher = "Pitcher"
  Catcher = "Catcher"
  FirstBaseman = "FirstBaseman"
  SecondBaseman = "SecondBaseman"
  ThirdBaseman = "ThirdBaseman"
  Shortstop = "Shortstop
end
```

Now we at least have tooling support.

And of course,
[Rails](https://api.rubyonrails.org/v5.2.3/classes/ActiveRecord/Enum.html)
implements enums on models.

## Ternary abuse as a desire path for conditional expressions or `do` expressions

In React JSX/TSX, this kind of code crops up:

```jsx
<Foo>
  {something === 3 ? (
    somethingElse === 17 ? (
      <ThreeAnd17Thing number={Math.floor(foo * Math.PI)}>
        "We got that 3 and 17 situation this time"
      </ThreeAnd17Thing>
    ) : (
      "We got that 3 and not 17 situation this time"
    )
  ) : somethingElse === 17 ? (
    "We got that not 3 and yes 17 situation this time"
  ) : (
    <NeitherThreeNor17Thing
      bar={baz(moo + 22)}
      quux={xyzzy}
      corge={grault}
      thud={wubble}
      flob={garply}
    >
      "We got that not 3 and not 17 situation this time"
    </NeitherThreeNor17Thing>
  )}
</Foo>
```

The problem is that we need the code to be an expression inside the
component, and we aren't quite ready to extract a subcomponent,[^should] so
we have a hot potato where we have to keep the code as a long expression
without any intermediate statements.

[^should]: We should do that to get rid of the nested `if`s and clean things up. The discussion that follows assumes we're not going to do that.

JavaScript lacks the ergonomics common to expression-based languages. For
example, in Clojure, you'd do something like this:[^clojure]

[^clojure]: There's no React here; I'm pretending the components in the JSX code are function calls here. Also, please forgive errors: I don't have a Clojure environment set up to lint/indent this, so I did it by hand.

```clojure
(if (= 3 something)
  (if (= 17 somethingElse)
    (let [fooTimesPi (* foo Math/PI)
          number (Math/floor fooTimesPi)]
      (ThreeAnd17Thing
        {:number number}
        "We got that 3 and 17 situation this time"))
    "We got that 3 and not 17 situation this time")
  (if (= 17 somethingElse)
    "We got that not 3 and yes 17 situation this time"
    (let [bazArg (+ 22 moo)
          bar (baz bazArg)
          args {:bar bar
                :quux xyzzy
                :corge grault
                :thud wubble
                :flob garply}]
      (NeitherThreeNor17Thing
        args
        "We got that not 3 and not 17 situation this time"))))
```

Note that we can `let`-bind locals within the expressions, so we can set the
hot potato down, as it were—while the whole form remains an expression.

In JavaScript, by contrast, the keywords `if` and `else` introduce statements, not
expressions, and we don't have a way of binding locals in an expression, so
we have to nest function calls and abuse ternaries.

What's missing is `do` expressions, which is a [Stage 1 TC39
proposal](https://github.com/tc39/proposal-do-expressions).

```jsx
<Foo>
  do {
    if (something === 3) {
      if (somethingElse === 17) {
        const fooTimesPi = foo * Math.PI;
        const number = Math.floor(fooTimesPi);

        <ThreeAnd17Thing number={number}>
          "We got that 3 and 17 situation this time"
        </ThreeAnd17Thing>
      } else {
        "We got that 3 and not 17 situation this time"
      }
    } else {
      if (somethingElse === 17) {
        "We got that not 3 and yes 17 situation this time"
      } else {
        const bazArg = moo + 22;
        const bar = baz(bazArg);
        const args = {
          bar,
          quux: xyzzy,
          corge: grault,
          thud: wubble,
          flob: garply
        };

        <NeitherThreeNor17Thing {...args}>
          "We got that not 3 and not 17 situation this time"
        </NeitherThreeNor17Thing>
      }
    }
  }
}
</Foo>
```
