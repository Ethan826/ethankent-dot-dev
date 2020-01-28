---
title: "Carried on: bags"
date: 2020-01-28 12:38:22-0600
author: Ethan Kent
draft: false
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

[^birthday]: Wikipedia, Birthday problem, https://en.wikipedia.org/wiki/Birthday_problem (last visited Jan. 10, 2020).

## Buckets based on modulus

My naïve implementation added unneeded complexity by basing what bucket
chosen on calculating the percentile of the hash in the range of possible
hash values and multiplying that by the number of buckets. That's more
operations than are needed. Using `hashResult % numBuckets` should distribute
the result evenly across the hash buckets. This also lets us simplify our
`Hashable` implementation.

## Linked lists

My naïve implementation uses an array of buckets, with each bucket itself an
array.[^pointer] The usual implementation is to use a linked list. It is a
bit unclear from my reading whether that's desirable per se, or if it is
simply because Donald Knuth said to do it that way in an example.[^knuth] I
think it's because linked lists don't do spooky things like reallocate
unexpectedly, and by the time a bucket's contents would need to reallocate,
we should be re-hashing.

[^pointer]: When I say the array is "in" the bucket, I actually mean that the bucket holds a pointer to the array, which is on the heap. So it's really a pointer that is "in" the array.
[^knuth]: Roman Leventov, Answer, _Why are we using linked list to address collisions in hash tables?_, Stack Overflow, https://stackoverflow.com/a/30238046/3396324 (May 14, 2015).

## Open addressing

### Why it's good

Another way of building a hashtable is to use open addressing.

Remember how our buckets hold linked lists or arrays rather than the values
themselves? That's called "chaining." Doing that means you have to go to some
other place in the computer's memory, and so can't take advantage of
"locality of reference." That term means that computers are clever about
caching things and that accessing something close in memory to something else
you recently accessed can be more efficient if the second thing happened to
be cached when you looked up the first thing.

So in practice, it is often faster to store the data directly in the array of
buckets. That is, you can have the buckets that make up the array actually
hold the values rather than holding pointers to linked lists or other arrays.

### Dealing with collisions

But this raises its own problem. Remember that with an array or linked list,
collisions are easy to deal with: we just add a new value to the collection
stored in each bucket. But when there's just that one array, we have to find
a different place to stick the value that had a collision, and we have to
think about how we'll delete that value.

#### Insertion

```text
Before:

We want to insert Z, which hashes to 20.

Current Value     | A  | Q  | M  | empty |
------------------------------------------
Bucket            | 20 | 21 | 22 | 23    |
```

##### Standard insertion

The usual approach is to put the value in the next empty slot. Then when it's
time to go looking for a value, you hash it, go to the appropriate bucket,
and if it's not in that bucket, keep going along the array till you find _the
value_ or _an empty bucket_. In other words, go to the bucket where you'd
expect the value to be _if there wasn't a collision_, then check the buckets
following that one, where you might have put it _if there was a collision_.
Finding an empty slot means you didn't stick the value in the next available
slot after a collision—because you wouldn't have left a gap in that case.

```text
After:

Current Value     | A  | Q  | M  | Z  |
---------------------------------------
Bucket            | 20 | 21 | 22 | 23 |
```

##### Robin Hood hashing

You can also do what's called Robin Hood hashing. In short, it means that if
you have a collision, as you look for an empty spot for your value, you keep
checking which value is _more out of place_—the one you're inserting or the
one that's already there. We rob from the rich to give to the poor, which is
where the name comes from.

Here's the same example, but with some extra information:

```text
Before:

We want to insert Z, which hashes to 20.

Current Value            | A  | Q  | M  | empty |
Correct bucket           | 19 | 19 | 21 | N/A   |
How out of place?        | 1  | 2  | 1  | N/A   |
-------------------------------------------------
How far off would Z be?  | 0  | 1  | 2  | 3     |
-------------------------------------------------
Bucket                   | 20 | 21 | 22 | 23    |
```

We are trying to insert the value _Z_, which hashes to 20. Using Robin Hood
hashing would have us try to insert the value in Bucket 20. But that bucket
is occupied. And its current value _A_ really should be in Bucket 19, so it's
already _one bucket out of place_. Meanwhile, _Z_ is _zero buckets out of
place_ here at Bucket 20, so we aren't going to exchange _Z_ and _A_. _A_ is
"poorer" because it is more out of place than _Z_ is. So we don't rob from
the poor to pay the rich here.

Now we come to Bucket 21, which contains _Q_, which is _two buckets out of
place_. _Z_ is now _one bucket out of place_. No exchange there: again, it
would be robbing from the poor to pay the rich.

We come next to Bucket 22, which contains _M_, which is one bucket out of
place. But now _Z_ is two buckets out of place. Let's put _Z_ in Bucket 22
and carry _M_ forward. We rob from the richer _M_, which is only one bucket
out of place, to pay _Z_, which is two buckets out of place.

Finally, we get to Bucket 23, which is empty, so _M_ goes there. The final
result is this:

```text
After:

Current Value     | A  | Q  | Z  | M  |
Correct bucket    | 19 | 19 | 20 | 21 |
How out of place? | 1  | 2  | 2  | 2  |
---------------------------------------
Bucket            | 20 | 21 | 22 | 23 |
```

By following this algorithm, values will not wind up wildly out of place,
thereby reducing the search cost following a collision, at little extra cost
(either way you must probe to the first empty spot; but by switching the
values, you make searches faster on average).

#### Deletion

There's still one hitch (and bear with me till the example in the next
paragraph): what if you insert a value, then have a collision and insert
another value in the next available bucket, _but then delete the first value_
(the one in its proper bucket)? Or, for that matter, delete _any_ value that
displaced another value because of a collision?

Imagine _B_ hashes to Bucket 22. Then _H_ also hashes to Bucket 22, so it
goes in Bucket 23.

```text
Before:

We want to delete B, which hashes to 22.

Current Value     | B  | H  |
-----------------------------
Bucket            | 22 | 23 |
```

Now you delete _B_. Finally, you check to see if _H_ is in the hashtable. It
hashes to bucket 22, but that bucket is empty. So you incorreclty conclude
that _H_ isn't in the hashtable, and report that back to the user.

```text
                    ↓ Problem!
Current Value     | empty  | H  |
---------------------------------
Bucket            | 22     | 23 |
```

There are two approaches often used in this situation.

##### Tombstones

First, you might replace the deleted entry with a _tombstone_, a special flag
that means "I deleted something here." So you treat that bucket exactly as
you would a bucket that was filled with a value that didn't match the one we
are searching for: it's not empty, and so you look at the next one. This is
easy, but it means load factor doesn't go down when you delete values: you
might have a hashtable with lots of tombstones just wasting space.

```text
After:

Current Value     | TOMBSTONE  | H  |
-------------------------------------
Bucket            | 22         | 23 |
```

##### Left shift

Alternatively, you might shift values backward to fill in the gap. This
avoids using tombstones, but adds to the time complexity of deletion.

```text
After:

Current Value     | H  | empty  |
---------------------------------
Bucket            | 22 | 23     |
```

## Conclusion

So we have discussed some fixes here that will improve our implementation:

- Improve the hash function.
- Rehash above a specified load factor.
- Use modulus in bucketing rather than a more complex algorithm.
- Use open addressing with Robin Hood hashing and left shift on deletion.

I will implement those changes and describe updating the code in my next
post.

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
