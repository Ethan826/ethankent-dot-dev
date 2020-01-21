---
title: "Carried on: bags"
date: 2020-01-04T11:16:53-08:00
author: Ethan Kent
draft: true
---

(This is part 3 of a series on the bag/multiset data structure. Check out
parts [1]({{<relref "/thats_my_bag">}}) and [2]({{<relref
"/implementing_a_bag">}}).)

## An extremely long analogy

Consider this the director's cut of a strange story that includes a variety
of alternative endings. The scene is set in a school with many floors and
hallways, all lined with lockers. On the first day of class, you are trying
to find a locker in which to store your backpack.

You say good morning to the teacher, and ask where you should put your
backpack. The teacher returns your greeting, and tells you something unusual:
"I don't assign lockers. For that, we use a device called the BackpackHasher.
You tell it a bit about your backpack—the brand, color, and it gives you a
locker assignment."

"Shouldn't I just put my backpack in the first empty locker I find," you ask.

"No, there are a lot of students in this class, so finding an empty locker can
take a long time. It's a lot quicker to use the BackpackHasher, because it
will tell you directly where to go."

"What if I forget where I put my backpack, or what if somebody else has to
get it for me," you ask.

"They can just describe your backpack to the BackpackHasher and it will tell
them the same locker number it told you. The locker assignments seem random,
but as long as a backpack is described the same way, the BackpackHasher comes
up with the same locker number."

You wait in line for the teacher's strange machine, then you describe your
backpack, and the BackpackHasher tells you, "go to locker 39."

"Weird," you think, "didn't the BackpackHasher tell that other student to go to
locker 39 a few minutes ago? In fact, I think I heard a _few_ students get locker
39."

### Alternate ending 1 (hashtable with pointers to arrays)

You walk directly to locker 39—it seems too small for a backpack,
actually—and open it. The locker is indeed too small to hold your backpack,
but it contains a note: "Go to the third floor hallway south of Mr. Jones's
room. Put your backpack in the first open locker in that hallway." You see
another student opening his locker, and it contains a similar note, but
his note refers to the sixth floor hallway north of Ms. Smith's classroom.

You're a pretty fast walker, and you get to the third floor pretty quickly.
As you scan the hallway, you can tell that lockers A–E already have backpacks
in them, but locker F is sitting open. You put your backpack there and head
to class.

The schoolday ends, and you realize you forgot where you put your backpack.
But you describe your backpack to the BackpackHasher, which tells you to go
to locker 39, which has the note telling you to go to the third floor hallway
south of Mr. Jones's room.

As you arrive, you see another student take her backpack out of locker C. A
worker in a uniform marked "Facilities" is watching, and quickly opens all
the lockers after C. He moves the backpack in locker H into locker G, the
backpack from locker G into locker F, and so on, until he closes up the gap
caused by the student removing her backpack. He leaves the first empty locker
door open and closes the rest. (You couldn't see the backpacks from where you
were standing when the Facilities worker moved them around.)

You realize you are going to have to find your backpack by starting at the
first locker and looking in each locker in turn: the backpacks aren't
arranged in lockers in any particular order. But this is still easier than
searching the huge number of lockers in the hallway outside homeroom.

"Pardon me, what happens if I go looking for my backpack but it isn't here,"
you ask.

"Well, you get to an empty locker and don't find your backpack, that's how
you know," responds the Facilities person. "This works pretty well, except
that sometimes all the lockers get filled up. Then we have to get all the
backpacks out of the lockers and carry them to a hallway with more lockers,
and then go back and update the instructions in the locker that sent you over
here."

### Alternate ending 2 (hashtable with pointers to linked lists)

Just like in ending 1, you are directed to the third-floor hallway. But this
time, you are explicitly told to open Locker J rather than find the first
empty locker in the hallway. Locker J already has a backpack, but it has a
laminated note hanging from a ring: "If I'm full, go to the second floor,
Locker N, and put your backpack there." (The floor and locker number are
handwritten onto the laminated card. There are some additional instructions,
but they relate to retrieving your backpack rather than putting it away.
You'll check those out later.)

You walk to Locker N on the second floor, which directs you to Locker Z on
basement level 1. But Locker Z's instructions don't tell you where to go
next. You see a person in a "Facilities" uniform. "Pardon me. This locker is
full, but the instructions don't tell me where to go next."

"Go, to, umm, how about Locker V in the cafeteria? Yeah, that one." You can
see he is looking at a list on his clipboard marked "Available Lockers." He
crosses out "Locker V" on his list, and updates the instruction card in
Locker Z so that it refers to Locker V throughout. Locker V is empty, and you
put away your backpack

At the end of the schoolday, you've forgotten your locker's location, but you
describe your backpack to the BackpackHasher, which sends you to Locker 39,
which points you to Locker J, then Locker N, then Locker Z. Locker Z's
instructions tell you to go to Locker V.

As you're about to leave for Locker V, you notice the rest of the
instructions on the laminated card: "Take me with you." You realize the ring
attached to the instruction card is on a cord, and the cord retracts
automatically (like a seatbelt). You pull it out a little bit and it zips
back.

So you take the Locker Z instruction card with you to Locker V. It's a long
cord! You open Locker Z and discover your backpack. Before you close the
locker, you read the rest of the instructions on the card you took with you:
"Call a Facilities worker for help."

You see a Facilities worker and ask for help. She takes the instruction card
tethered to Locker Z. She erases all references to Locker V on the
instruction card. She looks at the instructions in your locker, Locker V, and
sees that they refer to Locker Q (which you've never seen before).

On the instruction card you handed her—the one for Locker Z—she writes "Q"
everywhere "V" used to be. Now it has exactly the same instructions as the
card for your locker, Locker V. She lets go of the instructions on the
lanyard, and it weaves its way through the school, making a satisfying "zip"
sound as it goes. Finally, the Facilities worker erases the instruction card
inside your locker, Locker V, then adds "Locker V" back to the list of "Empty
Lockers." In short, the chain of instructions now skip your locker, and it is
back in the pool of empty lockers.

"What was that all about," you ask.

"Well, whoever came after you this morning saw your bag hanging in this
locker, Locker V, and we found that student a new locker, just like we did for
you. That student got Locker Q, as it happens. But now your old locker," she
points to Locker V, "isn't in use. So there's no need for the student who
came after you to bother stopping here. That's why we rigged up the
retractable cords. Now we can just rewrite the instructions about where to go
rather than actually moving any bags around. And we can make the chain of
instructions however long it needs to be because we aren't restricted to
lockers next to each other that fit in a hallway. But it does make you
students have to do a lot of walking around sometimes."

### Alternate ending 3 (hashtable with linear probing and tombstones)

As before, the BackpackHasher tells you to go to Locker 39 right next your
homeroom classroom. This time, it's large enough to hold a backpack—but
it's already full. There's a note: "Put your backpack in the first empty
higher-numbered locker." (You ignore some instructions about retrieving your
backpack.) Locker 40 and 41 are full, but Locker 42 is empty.

At the end of the schoolday, you describe your backpack to the BackpackHasher
and are told to go to Locker 39. The retrieval instructions say, "If this
isn't your backpack, check the next higher-numbered locker. If you reach an
empty locker, your backpack isn't here."

You find your

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

This commit implements re-hashing based on load factor.

I changed the default size to 16, as in Java's `HashMap`; they use
power-of-two sizes because there are optimizations available from bit masking
and similar fanciness. I have not gone that far with my implementation, but
it seems like a reasonable initial heuristic to choose the size in a common
implementation and double it every time we must re-hash.

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

[^pointer]: When I say the array is "in" the bucket, I actually mean that the bucket holds a pointer to the array, which is on the heap. So it's really a pointer that is "in" the array. I'll be using the construction throughout, however.
[^knuth]: Roman Leventov, Answer, _Why are we using linked list to address collisions in hash tables?_, Stack Overflow, https://stackoverflow.com/a/30238046/3396324 (May 14, 2015).

## Open addressing

### Why it's good

Another way of building a hashtable is to use open addressing instead of a
linked list.

Remember how our buckets hold linked lists or arrays rather than the values
themselves? Well, doing that means you have to go to some other place in the
computer's memory, and so can't take advantage of "locality of reference."
That term means that computers are clever about caching things and that
accessing something close in memory to something else you recently accessed
can be more efficient if the second thing happened to be cached when you
looked up the first thing.

So in practice, it is often faster to store the data directly in the array of
buckets. That is, you can have the buckets that make up the array actually
hold the values rather than holding pointers to linked lists or other arrays.

### Dealing with collisions

But this raises its own problem. Remember that with an array or linked list,
collisions are easy to deal with: we just add a new value to the collection
stored in each bucket. But when there's just that one array, we have to find
a different place to stick the value that had a collision.

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
