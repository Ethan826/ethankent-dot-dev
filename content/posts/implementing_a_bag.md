---
title: "Implementing a bag"
date: 2020-01-03T11:39:14-08:00
author: Ethan Kent
draft: false
---

In my previous [post]({{< relref "/thats_my_bag" >}}), I talked about a bag
data structure, which is like a set that allows repeated occurrences of the
same value.

In this post I will discuss my naïve implementation that has the
goal of `O(1)` best case insertion, search, and deletion. You can follow
along by viewing the code at https://www.github.com/ethan826/bag.

In the next post I'll discuss improvements to consider, mostly around
improving the hash function and considering load factor. For now we are using
techniques you wouldn't want to use in production: among other things, we set
a `numBuckets` value to 100 and never adjust that; and our hashing method may
not be robust against collisions. I will also discuss testing in a future
post.

One other preliminary note: the easiest way to do this would be to wrap a
hashmap/object.

```typescript
class Bag<T> {
  private bag: Object<T, number>;

  constructor() {
    this.bag = {};
  }

  insert(value: T) {
    if (this.bag.hasOwnProperty(value)) {
      this.bag[value] += 1;
    } else {
      this.bag[value] = 1;
    }
  }

  // And so on...
}
```

I wanted to learn a bit about how hashtable-based data structures worked, so
I've rolled my own using arrays.

## Generic bag

We want the bag to accept generic types, but we want to make sure that any
type we use can be hashed. For the time being, it makes the most sense to
require types to satisfy this requirement themselves: I don't care what kind
of type you are so long as you implement the behavior we ask for. So
throughout this project, the type signature is `T extends Hashable`, where
`Hashable` is this:

```typescript
// src/Hashable/index.ts

/**
 * Represents a type that can be hashed to an integer value.
 */
export default interface Hashable {
  hashCode: () => number;
  maximumHashValue: number;
  minimumHashValue: number;
}
```

So a type that extends `Hashable` is one that (1) can hash itself, and (2)
can tell you where that hashed value fits in with the range of possible
outputs from the hashing function. This is an example of asking "who needs to
know this?" Should `Bag` and its collaborators know how to hash every
possible type? Should a person creating a new type have to write code in
`Bag` or one of its collaborators? No. This is also an example of the _O_ in
_SOLID_.

That allows the `HashTable` class to work out what bucket the value belongs
in:

```typescript
// src/Bag/HashTable.ts

/**
  * Determine what bucket—meaning the index of `hashTable`—the `value` belongs
  * in. This is the result of taking the hashcode for `value` and determining
  * proprotionately how far between the minimum and maximum `hashCode` values
  * that `value` falls, then finding what integer that falls into from zero to
  * `numBuckets`.
  *
  * @param value - The value to determine the bucket for.
  * @returns The bucket number, meaning the index for `hashTable`.
  */
private computeBucket(value: T): number {
  const sizeOfRange = value.maximumHashValue - value.minimumHashValue;
  const distanceFromFloor =
    (value.hashCode() - value.minimumHashValue) / sizeOfRange;
  return Math.floor(distanceFromFloor * this.numBuckets);
}
```

This is also an example of "describing what kinds of arguments are
appropriate in _precisely_ the terms that make sense" that I referred to in
an earlier [post]({{< relref "/types_are_for_people" >}}).

## Design

Recall that hash tables work a bit like libraries. You look up a book by call
number, which lets you end up standing in front of the right bookshelf. Then
you look at the books that are present to find the one you want. For our
implementation,

- `Bag` is the library: it contains the bookshelves and the card catalog, and
  keeps track of the overall status of things (for example, it might track
  how many total books are present).
- `HashTable` is the card catalog: it helps you go from knowing what book
  title you want to standing in front of the correct shelf.
- `Bucket` is the shelf: it's what actually holds a small subset of the
  library's books, and the thing from which books are actually inserted and
  removed.

### `Bag`

`Bag` exposes methods like `insert(value: T)`, `delete(value: T)`,
`deleteAll(value: T)`, and `count(value: T)`. `Bag`—after doing its own
bookkeeping (for example increasing the value returned by
`length()`—delegates all these methods to the same methods on `Bucket`. `Bag`
also directly delegates `toArray()` to `HashTable`.

### `HashTable`

`HashTable` maintains the array of buckets and knows how to fetch a `Bucket`
based on a `value`. It simply organizes buckets and passes them up to `Bag`.
It calls `hashCode()` on values, then figures out proportionately where that
fits given the number of buckets (if the value hashes to 9,999 in a range of
10,000 possible hash codes, and there are ten buckets, `HashTable` figures
out that we want the tenth bucket).

### `Bucket`

In an ideal world a `Bucket` would contain only a few values. If there were
only one bucket or if the hashing algorithm always returned the same value,
all entries in the `HashTable` (and therefore the `Bag`) would be in a single
bucket. That's why the worst-case performance for common operations is
`O(n)`.

`Bucket` exposes a similar public API as `Bag` does, and that's because `Bag`
uses `HashTable` to find the right `Bucket`, then asks `Bucket` to do the
work of inserting, deleting, counting, etc.

`Bucket` also hides the detail mentioned in the previous
[post]({{< relref "/thats_my_bag" >}}), namely avoiding the presumption that something is intended by the difference between a value that's missing from the data structure versus a value that reports zero occurrences.[^priors]

[^priors]: See my earlier [post]({{< relref "/priors" >}}) about avoiding differences that don't mean anything.

## Injectability

`Bag` allows us to inject the `HashTable` implementation. A good refactor
would be to define `HashTable` as an interface and make the parameter generic
over `HashTable` interfaces. Similarly, `HashTable` allows us to inject
`Bucket`'s constructor. The same refactor applies here.

The goal with this is to satisfy the _D_ of _SOLID_, which brings along
benefits when testing, as I shall take up in a future post.

I arrived at this overall architecture by applying the _extract class_
refactor two times. What drove me to that was that I realized it was
difficult but important to test aspects that would have been part of the
private API of `Bag` if it had been one giant class. The fact that something
important to test is inside a private API is a sign that you should extract
that logic into collaborator and probably inject that dependency.

## Equality

One other thing to notice here is that we are not preserving object identity.
We are treating objects that are equal and that hash to the same bucket as
being indistinguishable. In other words, as long as two objects have
structural equality, we will allow all copies to be garbage collected. This
is the same thing that happens in a set, but not the same thing that happens
in an array. Using DDD terminology, a bag of this kind would be an
inappropriate way to store entities but would be suitable for value objects.
