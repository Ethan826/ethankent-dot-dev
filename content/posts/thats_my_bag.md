---
title: "That's my bag, baby"
date: 2020-01-01T12:13:12-06:00
author: Ethan Kent
draft: false
---

(This is part of a series of articles. The next one is [here]({{<relref "/implementing_a_bag">}}).)

When we want a collection with elements that may repeat, we generally reach
for an array, list, vector, etc. When we want a collection with elements that
don't or whose repeats we don't care about, we generally reach for a set.

Efficient set implementations often don't track insertion order. For
hashsets, that's because they hash their input and organize based on that
hashed value. For binary tree sets, it's because inputs are sorted in a
binary tree. A good hashset can insert, retrieve, and search for elements in
something close to `O(1)` time.[^collisions] A balanced binary tree set can
insert and retrieve elements in `O(log n)` time. In essence, by not caring
about insertion order, we can get faster performance.

[^collisions]: I say "something close to" because if you had very bad luck, a bad hashing algorithm, too few buckets, etc., you can wind up with all the elements hashing to the same bucket, meaning that the retrieval and search time will be linear. If you're unfamiliar with how hashmaps and hashsets work, read up on that because it's pretty easy to understand and super cool. The Wikipedia [article](https://en.wikipedia.org/wiki/Hash_table) is a good starting point. Stay tuned for a similar implementation of a related data structure below.

In short, when we care about neither insertion order nor repeats, we use a
set. But what if we don't care about insertion order but _do_ care about
repeats? Say, for example, we wanted to know how many cars a dealership has,
but not in what order they were acquired?[^weird] In that case, there is a
data structure we could use: a bag, also called a multiset.

[^weird]: It's unclear in retrospect why I chose this example, as I don't have any special interest in cars and this seems like a wildly naïve view of how car dealership software works. Car dealership inventory systems might even be so fancy as to use a database with the ability to `COUNT` entries. Let's just go with it.

## Why a bag seems useful

I am puzzled this kind of data structure is uncommon, as its characteristics
would seem to be a good match for a common use case. Probably the reason is
that a hashmap with keys as entries and tallies as values does a pretty good
job of satisfying that use case:

```json
{
  "Ford F150": 0,
  "Honda Fit": 3,
  "Honda Pilot": 1,
  "Toyota Camry": 5
}
```

But it's not quite perfect as we may have an element be absent or have a
tally of zero. Do we mean to signify something different about Ford F150s and
Toyota Tacomas? Are we saying this dealership sometimes has F150s but never
has Tacomas? It always starts to get dodgy when we embed semantic meaning in
differences like that. But if there isn't a difference, what's the deal? We
don't want to have to write code like this:

```typescript
type Inventory = { [key: string]: number };

class Dealership {
  constructor(private inventory: Inventory) {}

  /**
   *  Return whether the dealership has a particular vehicle.
   */
  hasVehicle(vehicleType: string): boolean {
    // This kind of stuff should be part of the bag / multiset's API
    return (
      this.inventory.hasOwnProperty(vehicleType) &&
      this.inventory[vehicleType] > 0
    );
  }
}

const myDealership = new Dealership({
  "Ford F150": 0,
  "Honda Fit": 3,
  "Honda Pilot": 1,
  "Toyota Camry": 5,
});

myDealership.hasVehicle("Honda Fit"); // => true
myDealership.hasVehicle("Ford F150"); // => false
myDealership.hasVehicle("Yugo"); // => false
```

C++'s Standard Template Library has a multiset with a `contains` and `count`
method.[^contains] This does what we want.

[^contains]: cppreference.com, std::multiset, https://en.cppreference.com/w/cpp/container/multiset (last visited Dec. 24, 2019). The `contains` method is part of the C++20 spec. _Id._ There is also a `find` method, which is a little confusing in that you might expect it to return the `key` you used to do the finding, which doesn't seem terribly useful. But it actually can return an iterator—but only over that element you used to do the finding, _and not repeats_; for that you use `equal_range`. So it actually is weird, maybe; I haven't researched it enough to understand the exact rationale.

## How you might implement a bag

I am going to dedicate a few articles to running through what's going on
here, but I have a working implementation of a bag
[here](https://www.github.com/ethan826/bag). For now, you can clone it and
run `yarn` and then `yarn test`.
