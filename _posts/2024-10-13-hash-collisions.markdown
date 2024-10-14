---
layout: post
title:  "CPython hash table collisions"
date:   2024-10-14 13:20:44 +0000
categories: jekyll update
katex: true
---

I was overanalyzing two sum (as one does), trying to figure out its *actual* time complexity (\$O(n log m)\$ where `m` is the largest number in `n`, as opposed to the liars who say its \$O(n)\$), when I stumbled upon [this comment](https://github.com/python/cpython/blob/ce740d46246b28bb675ba9d62214b59be9b8411e/Python/pyhash.c#L31-L32) in the CPython source code:
{% highlight c %}
/* For numeric types, the hash of a number x is based on the reduction
   of x modulo the prime P = 2**_PyHASH_BITS - 1.
{% endhighlight %}
(reduction of x modulo the prime `P = 2**_PyHASH_BITS - 1` is just a fancy way of saying \$x \mod 2^{61} - 1\$, aka the `%` operator)

The first thing I tried to do was make a hash table filled with collisions by filling it with multiples (+ 1) of `P`, to see how slow it was.

## A primer on dictionaries
Firstly, a primer on dictionaries.
A dictionary, (or a hash table, mapping, whatever your language calls it), is a data structure that maps keys to values. It does this by "hash"ing the key into a shorter version, and then using the hash of the key to map to the value for \$O(1)\$ lookup time.

... most of the time.

## Hash collisions and dictionaries
Because hashes shorten keys into smaller values, there will be inevitably be multiple keys that hash to the same value. Hash tables have to somehow solve this issue by storing and comparing against the original (un-hashed key). Multiple collisions means comparing against more keys, and thus causing performance issues due to the table having to check multiple keys, which is why most hash tables use a universal hash function or another hash function designed for hash tables like [SipHash](https://en.wikipedia.org/wiki/SipHash), which, to its credit, is what CPython uses for `str` and `bytes` objects.

However, for integer hashing, CPython calculates the integer modulo the prime `P` as specified above. This can lead to trivial hash collisions like:
{% highlight python %}
>>> hash(1)
1
>>> hash(p + 1)
1
>>> hash(2 * p + 1)
1
{% endhighlight %}

## Benchmarks
{% highlight python %}
>>> import timeit
>>> p = 2 ** 61 - 1
>>> timeit.timeit("collisions = {x * p + 1: 4 for x in range(1, 1000)}", setup="from __main__ import p", number=100)
1.0314989800099283
>>> p2 = 2 ** 62 - 1
>>> timeit.timeit("no_collisions = {x * p + 1: 4 for x in range(1, 1000)}", setup="from __main__ import p2 as p", number=100)
0.019491110928356647
{% endhighlight %}

Interestingly, the hash table made *with* collisions was about 50x slower than the hash table without any collisions. Testing on different values of `p2` and machines kept the same results. The logical next step, then, was to measure how slow lookup times were in the collisions hash table compared to the collision-less hash table (choosing 420 as an arbitrary number here, your mileage may vary for reasons specified below).

{% highlight python %}
>>> collisions = {x * p + 1: 4 for x in range(1, 1000)}
>>> no_collisions = {x * p2 + 1: 4 for x in range(1, 1000)}
>>> timeit.timeit("collisions[420 * p + 1]", setup = "from __main__ import collisions, p")
8.073610159917735
>>> timeit.timeit("no_collisions[420 * p2 + 1]", setup = "from __main__ import no_collisions, p2")
0.1297627940075472
{% endhighlight %}

The lookup time difference was even worse, by a factor of 62x between the collisions and collision-less hash tables. Why is it this much slower even with only 1,000 elements? To figure this out, we need to check the CPython source code.

## Explanation

### Lookups
Firstly, why is the lookup time so much slower? How does CPython lookup colliding hash values? Looking in `Objects/dictobject.c` gives us the [`do_lookup` function](https://github.com/python/cpython/blob/ce740d46246b28bb675ba9d62214b59be9b8411e/Objects/dictobject.c#L995), defined as
{% highlight c %}
static inline Py_ALWAYS_INLINE Py_ssize_t
do_lookup(PyDictObject *mp, PyDictKeysObject *dk, PyObject *key, Py_hash_t hash,
          int (*check_lookup)(PyDictObject *, PyDictKeysObject *, void *, Py_ssize_t ix, PyObject *key, Py_hash_t))
{
    void *ep0 = _DK_ENTRIES(dk);
    size_t mask = DK_MASK(dk);
    size_t perturb = hash;
    size_t i = (size_t)hash & mask;
    Py_ssize_t ix;
    for (;;) {
        ix = dictkeys_get_index(dk, i);
        if (ix >= 0) {
            int cmp = check_lookup(mp, dk, ep0, ix, key, hash);
            if (cmp < 0) {
                return cmp;
            } else if (cmp) {
                return ix;
            }
        }
        else if (ix == DKIX_EMPTY) {
            return DKIX_EMPTY;
        }
        perturb >>= PERTURB_SHIFT;
        i = mask & (i*5 + perturb + 1);

        // Manual loop unrolling
        ix = dictkeys_get_index(dk, i);
        if (ix >= 0) {
            int cmp = check_lookup(mp, dk, ep0, ix, key, hash);
            if (cmp < 0) {
                return cmp;
            } else if (cmp) {
                return ix;
            }
        }
        else if (ix == DKIX_EMPTY) {
            return DKIX_EMPTY;
        }
        perturb >>= PERTURB_SHIFT;
        i = mask & (i*5 + perturb + 1);
    }
    Py_UNREACHABLE();
}
{% endhighlight %}

The main part of this code is inside of the for loop. The code firstly uses `dictkeys_get_index`, which should have little to no collisions in most cases (except ours). After that, if a collision is found, it repeatedly "perturbs" the index `i` with the equation \$i * 5 + perturb + 1 \pmod {mask + 1}\$, where `mask` is defined as \$2^{\lceil\log_2(length)\rceil}\$ until it reaches the correct key, which it checks by using the function `check_lookup`. In our case, `i` starts at 1 (as `hash(i) = 1`), which after shifted by `PERTURB_SHIFT`, turns into 0, where it then iterates through all the members of the dict due to modular arithmetic (which only works because 5 is coprime to the modulus `mask`, which is defined as the smallest power of 2 larger than the size of the dictionary).

The perturbation goes through the hash table until it reaches the correct element, which takes a while until it reaches the correct element. This causes major slowdowns in tables with lots of collisions (like ours), causing a huge slowdown. Due to this perturbation strategy, the insertion order of the key also strongly affects the performance (\$p + 1\$ is approximately equal to a collision-less lookup, while \$840p + 1\$ takes about double the time \$420p + 1\$ takes).

### Initialization
Why, then, is initialization faster compared to its collision-less counterpart, then, compared to lookup, but slower than a collision-less initialization?
Initialization (seems) to use `build_indices_generic`, which is defined as
{% highlight c %}
static void
build_indices_generic(PyDictKeysObject *keys, PyDictKeyEntry *ep, Py_ssize_t n)
{
    size_t mask = DK_MASK(keys);
    for (Py_ssize_t ix = 0; ix != n; ix++, ep++) {
        Py_hash_t hash = ep->me_hash;
        size_t i = hash & mask;
        for (size_t perturb = hash; dictkeys_get_index(keys, i) != DKIX_EMPTY;) {
            perturb >>= PERTURB_SHIFT;
            i = mask & (i*5 + perturb + 1);
        }
        dictkeys_set_index(keys, i, ix);
    }
}
{% endhighlight %}
This repeatedly perturbs `i` until it reaches an empty memory address. When `hash & mask` does not collide with any other elements, it never goes into the `for` loop, thus saving a lot of computation, and why it is slower than an initialization without any collisions. It is also faster than looking up a key as it just needs to reach a region of memory without any keys "occupying" it, thus being faster than a lookup relatively to its collision-less counterparts.
