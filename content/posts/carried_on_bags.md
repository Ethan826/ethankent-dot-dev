---
title: "Carried on: bags"
date: 2020-01-04T11:16:53-08:00
author: Ethan Kent
draft: true
---

(This is part 3 of a series on the bag/multiset data structure. Check out
parts [1]({{<relref "/thats_my_bag">}}) and [2]({{<relref
"/implementing_a_bag">}}).)

## Improving the hash function

So our bag works per our tests. Let's take a look at the naïve implementation
of hashing, at least for strings:

```typescript
// This is the equivalent of `impl`ing a trait in Rust.
type String = Hashable;

// https://stackoverflow.com/a/8076436/3396324
String.prototype.hashCode = function(): number {
  return [...this].reduce((hash, character) => {
    hash = (hash << 5) - hash + character.charCodeAt(0);
    return hash & hash;
  }, 0);
};
String.prototype.minimumHashValue = -0x80000000;
String.prototype.maximumHashValue = 0x7fffffff;

export default String;
```

One bit of good news is this: this is not baked in to the bag's
implementation. We already discussed how it was desirable from an Open–Closed
Principle point of view to make this the responsibility of each type that
would like to be stored in a bag.

I don't know enough about hashing algorithms to eyeball it in order to
evaluate whether this one is collision-resistant in common applications for
strings. But we can test it out empirically using a large [SCOWL
dictionary](http://app.aspell.net/create).

With our `hashCode()` ported from Java per Stack Overflow, let's bucket our
657,769 dictionary entries in 100 buckets and count how many values are in
each bucket. The mean is, of course, about 6,578. But the median is 5,830 and
the standard deviation is about 3,090. The smallest bucket has 4,677 entries
while the largest has 26,835 entries.

Not great. Let's try this with the MurmurHash3 algorithm. I will
simply steal the minified JavaScript from
[here](https://github.com/garycourt/murmurhash-js/blob/master/murmurhash3_gc.js).

Now we have a median of about 6,571, a standard deviation of about 72, and a
minimum and maximum of about 6,424 and 6,781, respectively. That seems
better.

And this reinforces our decision to have the data we want to put in our bag
extend `Hashable`: We could test this particular hashing algorithm against a
dictionary of English words without worrying about the behavior for other
kinds of data. For other data types, some other means of hashing might be
better.

## Improving load factor

If you have 100 buckets and 75 entries, the _load factor_ is the ratio of
those two, 0.75. Different languages use slightly different values, but 0.75
to at most about 0.90 appear to be the maximum load factors allowable for
best performance.

One of those surprising statistical factoids is that if you're hosting a
party and want to ensure a 50% chance that two guests share the same
birthday, you need only invite 23 people; with 70 people, there's a 99.9%
chance of a shared birthday (in each case assuming birthdays are uniformly
distributed).[^birthday] And so it is with hash tables, where collisions
become a problem sooner than you'd think.

To make sure load factor is in good shape, we need to pick a good starting
value, then resize our hashtable when the load factor gets too high.

This commit implements re-hasing based on load factor.

I changed the default size to 16, as in Java's `HashMap`; they use
power-of-two sizes because there are optimizations available from bit masking
and similar fanciness. I have not gone that far with my implementation, but
it seems like a reasonable initial heuristic to choose the size in a common
implementation and double it every time we must re-hash.

[^birthday]: Wikipedia, Birthday problem, https://en.wikipedia.org/wiki/Birthday_problem (last visited Jan. 10, 2020).

## Linked lists

My implementation uses an array of buckets, with each bucket itself an array.
The usual implementation is to use a linked list. It is a bit unclear from my
reading whether that's desirable per se, or if it is simply because Donald
Knuth said to do it that way in an example.[^knuth] I think it's because
linked lists don't do spooky things like reallocate unexpectedly, and by the
time a bucket's contents would need to reallocate, we should be re-hashing.

[^knuth]: Roman Leventov, Answer, _Why are we using linked list to address collisions in hash tables?_, Stack Overflow, https://stackoverflow.com/a/30238046/3396324 (May 14, 2015).

This commit implements a linked list instead of an array for bucket contents.

## Open addressing

## Bucketing algorithm

<!--
```javascript
function murmurhash(e,c){var h,r,t,a,o,d,A,C;for(h=3&e.length,r=e.length-h,t=c,o=3432918353,d=461845907,C=0;C<r;)A=255&e.charCodeAt(C)|(255&e.charCodeAt(++C))<<8|(255&e.charCodeAt(++C))<<16|(255&e.charCodeAt(++C))<<24,++C,t=27492+(65535&(a=5*(65535&(t=(t^=A=(65535&(A=(A=(65535&A)*o+(((A>>>16)*o&65535)<<16)&4294967295)<<15|A>>>17))*d+(((A>>>16)*d&65535)<<16)&4294967295)<<13|t>>>19))+((5*(t>>>16)&65535)<<16)&4294967295))+((58964+(a>>>16)&65535)<<16);switch(A=0,h){case 3:A^=(255&e.charCodeAt(C+2))<<16;case 2:A^=(255&e.charCodeAt(C+1))<<8;case 1:t^=A=(65535&(A=(A=(65535&(A^=255&e.charCodeAt(C)))*o+(((A>>>16)*o&65535)<<16)&4294967295)<<15|A>>>17))*d+(((A>>>16)*d&65535)<<16)&4294967295}return t^=e.length,t=2246822507*(65535&(t^=t>>>16))+((2246822507*(t>>>16)&65535)<<16)&4294967295,t=3266489909*(65535&(t^=t>>>13))+((3266489909*(t>>>16)&65535)<<16)&4294967295,(t^=t>>>16)>>>0}
String.prototype.hashCode = function() {
  return murmurhash(this);
};
String.prototype.minimumHashValue = 0;
String.prototype.maximumHashValue = 2**32;
function computeBucket(value) {
  const sizeOfRange = value.maximumHashValue - value.minimumHashValue;
  const distanceFromFloor =
    (value.hashCode() - value.minimumHashValue) / sizeOfRange;
  return Math.floor(distanceFromFloor * 100);
}
const frequencies = arr =>
  arr.reduce((result, element) => {
    result.hasOwnProperty(element) ? result[element]++ : (result[element] = 1);
    return result;
  }, {});
```
I created a frequencies helper function like
[Clojure's](https://clojuredocs.org/clojure.core/frequencies) (and now Ruby's
[tally](https://docs.ruby-lang.org/en/master/Enumerable.html#method-i-tally)).

```javascript
> String.prototype.hashCode = function () { return [...this].reduce((hash, character) => { hash = (hash << 5) - hash + character.charCodeAt(0); return hash & hash; }, 0); }; String.prototype.minimumHashValue = -0x80000000; String.prototype.maximumHashValue = 0x7fffffff; function computeBucket(value) { const sizeOfRange = value.maximumHashValue - value.minimumHashValue; const distanceFromFloor = (value.hashCode() - value.minimumHashValue) / sizeOfRange; return Math.floor(distanceFromFloor * 100); }; const frequencies = (arr) => arr.reduce((result, element) => { result.hasOwnProperty(element) ? result[element]++ : result[element] = 1; return result; }, {});
// => 2147483647
> const dict = fs.readFileSync("dict.txt").toString().split("\n");
// => undefined
> frequencies(dict.map(computeBucket));
``` -->
