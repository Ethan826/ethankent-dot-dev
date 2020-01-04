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

<!--
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

Bucketing our 657,769 dictionary entries in 100 buckets and counting how many
values are in each bucket, the mean is, of course, about 6,578. But the
median is 5,830 and the standard deviation is about 3,090. The smallest
bucket has 4,677 entries while the largest has 26,835 entries.

Not great. Let's try this with the algorithm Rust uses, Siphash1-3. I will
simply steal the minified JavaScript from
[here](https://raw.githubusercontent.com/jedisct1/siphash-js/master/lib/siphash13.js.min).

```javascript
var SipHash13 = (function() {
  "use strict";
  function r(r, n) {
    var t = r.l + n.l,
      h = { h: (r.h + n.h + ((t / 2) >>> 31)) >>> 0, l: t >>> 0 };
    (r.h = h.h), (r.l = h.l);
  }
  function n(r, n) {
    (r.h ^= n.h), (r.h >>>= 0), (r.l ^= n.l), (r.l >>>= 0);
  }
  function t(r, n) {
    var t = {
      h: (r.h << n) | (r.l >>> (32 - n)),
      l: (r.l << n) | (r.h >>> (32 - n)),
    };
    (r.h = t.h), (r.l = t.l);
  }
  function h(r) {
    var n = r.l;
    (r.l = r.h), (r.h = n);
  }
  function e(e, l, o, u) {
    r(e, l),
      r(o, u),
      t(l, 13),
      t(u, 16),
      n(l, e),
      n(u, o),
      h(e),
      r(o, l),
      r(e, u),
      t(l, 17),
      t(u, 21),
      n(l, o),
      n(u, e),
      h(o);
  }
  function l(r, n) {
    return (r[n + 3] << 24) | (r[n + 2] << 16) | (r[n + 1] << 8) | r[n];
  }
  function o(r, t) {
    "string" == typeof t && (t = u(t));
    var h = { h: r[1] >>> 0, l: r[0] >>> 0 },
      o = { h: r[3] >>> 0, l: r[2] >>> 0 },
      i = { h: h.h, l: h.l },
      a = h,
      f = { h: o.h, l: o.l },
      c = o,
      s = t.length,
      v = s - 7,
      g = new Uint8Array(new ArrayBuffer(8));
    n(i, { h: 1936682341, l: 1886610805 }),
      n(f, { h: 1685025377, l: 1852075885 }),
      n(a, { h: 1819895653, l: 1852142177 }),
      n(c, { h: 1952801890, l: 2037671283 });
    for (var y = 0; y < v; ) {
      var d = { h: l(t, y + 4), l: l(t, y) };
      n(c, d), e(i, f, a, c), n(i, d), (y += 8);
    }
    g[7] = s;
    for (var p = 0; y < s; ) g[p++] = t[y++];
    for (; p < 7; ) g[p++] = 0;
    var w = {
      h: (g[7] << 24) | (g[6] << 16) | (g[5] << 8) | g[4],
      l: (g[3] << 24) | (g[2] << 16) | (g[1] << 8) | g[0],
    };
    n(c, w),
      e(i, f, a, c),
      n(i, w),
      n(a, { h: 0, l: 255 }),
      e(i, f, a, c),
      e(i, f, a, c),
      e(i, f, a, c);
    var _ = i;
    return n(_, f), n(_, a), n(_, c), _;
  }
  function u(r) {
    if ("function" == typeof TextEncoder) return new TextEncoder().encode(r);
    r = unescape(encodeURIComponent(r));
    for (var n = new Uint8Array(r.length), t = 0, h = r.length; t < h; t++)
      n[t] = r.charCodeAt(t);
    return n;
  }
  return {
    hash: o,
    hash_hex: function(r, n) {
      var t = o(r, n);
      return (
        ("0000000" + t.h.toString(16)).substr(-8) +
        ("0000000" + t.l.toString(16)).substr(-8)
      );
    },
    hash_uint: function(r, n) {
      var t = o(r, n);
      return 4294967296 * (2097151 & t.h) + t.l;
    },
    string16_to_key: function(r) {
      var n = u(r);
      if (16 !== n.length) throw Error("Key length must be 16 bytes");
      var t = new Uint32Array(4);
      return (
        (t[0] = l(n, 0)),
        (t[1] = l(n, 4)),
        (t[2] = l(n, 8)),
        (t[3] = l(n, 12)),
        t
      );
    },
    string_to_u8: u,
  };
})();
String.prototype.hashCode = function() {
  const key = SipHash13.string16_to_key("0123456789ABCDEF");
  return SipHash13.hash_uint(key, this);
};
String.prototype.minimumHashValue = 0;
String.prototype.maximumHashValue = Number.MAX_SAFE_INTEGER;
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

## Improving load factor
