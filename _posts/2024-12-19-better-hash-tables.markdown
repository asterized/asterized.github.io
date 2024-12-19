---
layout: post
title:  "More collision-resistant hash tables"
date:   2024-12-19 10:18:49 +0000
categories: jekyll update
katex: true
---

A sequel to the [previous post](https://somehybrid.github.io/jekyll/update/2024/10/14/hash-collisions.html). If you haven't read it, read it now.

The previous post covered integer collisions in CPython's hash tables - notably that the hash algorithm for integers in CPython was done by taking the integer modulo a constant $P$, which was defined as the Mersenne prime $2^{61} - 1$. The main issue regarding this is DoS attacks on hash tables - namely that if an attacker that could add elements to a hash table is able to inexpensively manufacture hash collisions, they could fill it with items with the same hash and slow it down a lot. In the previous post, this could be done just by taking numbers congruent modulo $P$. How would you be able to combat such attacks?

## A first attempt at a better hash table
One way (and the most intuitive to me) to prevent hash collisions is instead of taking integers modulo $P$, you could bitwise XOR the ints with a secret that such an attacker wouldn't know.

An implementation is given below:
{% highlight python %}
import secrets

class XorDict(dict):
    def __init__(self, *args, **kwargs):
        self.secret = secrets.randbits(64)
        super().__init__(*args, **kwargs)

    def __getitem__(self, key: Any):
        if isinstance(key, int):
            key ^= self.secret

        try:
            return super().__getitem__(key)
        except KeyError:
            raise KeyError(key)

    def __getitem__(self, key: Any, value: Any):
        if isinstance(key, int):
            key ^= self.secret

        return super().__setitem__(key)
{% endhighlight %}

### Constructing collisions
We can see this uses CPython's integer hash again, meaning you could arrange this as $(x \oplus s) \mod {P}$, with $x$ and $s$ as the input and secret respectively, and $P$ being the modulus defined above. From there, we can reuse the collision with congruence, by doing something like:
{% highlight python %}
>>> def badhash(x: int):
...    return (x ^ 12) % 15  # selecting values of s and p as 12 and 15 as a simplified example
...
>>> badhash(0b10)
14
>>> 15 << 4 + 2
242
>>> badhash(242)
14
{% endhighlight %}

This works in essentially the same way as the other collision with more steps. You can just take the modulus multiplied by the smallest power of 2 larger than the modulus and the secret, which boils down to $[qP + ((x \mod P) \oplus s)] \equiv [(x \mod P) \oplus s] \pmod P$. The attacker just needs to know the number of bits in $P$ and $s$, which would only take a maximum of $\lceil \log_2(max(P, s)) \rceil$ hits against the hash table if not already known.

## A second attempt
As mentioned in the previous post, a good hashing algorithm for hash tables in CPython already exists - [SipHash](https://en.wikipedia.org/wiki/SipHash). One way you could construct a hash table is by converting the integers into bytes objects and then hashing them.

In code:
{% highlight python %}
from collections.abc import Iterable, Sequence
from typing import override, overload

class SipDict[K, V](dict[K | int, V]):  # pyright: ignore[reportMissingTypeArgument]
    def _calculate_index(self, key: int):
        return hash(key.to_bytes((key.bit_length() + 7) // 8))

    @overload
    def __setitem__(self, key: int, value: V) -> None: ...

    @overload
    def __setitem__(self, key: K, value: V) -> None: ...

    @override
    def __setitem__(self, key: K | int, value: V) -> None:
        index = key
        if isinstance(key, int):
            index = self._calculate_index(key)

        return super().__setitem__(index, value)

    @overload
    def __getitem__(self, key: int) -> V: ...

    @overload
    def __getitem__(self, key: K) -> V: ...

    @override
    def __getitem__(self, key: K | int) -> V:
        index = key
        if isinstance(key, int):
            index = self._calculate_index(key)

        try:
            return super().__getitem__(index)
        except KeyError:
            raise KeyError(key)

    @classmethod
    def from_pairs(cls, data: Iterable[tuple[K, V]]) -> SipDict[K, V]:
        out: SipDict[K, V] = SipDict()
        for item in data:
            out[item[0]] = item[1]
        return out
{% endhighlight %}

The attacker could trivially find 2 keys producing the same hash by noting that the algorithm finds the hash of the byte representation of an integer. If the dictionary wasn't constrained specifically to integers, the attacker could get the byte representation and create 2 collisions. However, I do not give 2 ~~shits~~ hash collisions, as it wouldn't impact performance much.

## Benchmarks

Functions used to benchmark:
{% highlight python %}
def compare_initialization(data: list[tuple[int, int]]):
    start = time.perf_counter()
    _ = SipDict.from_pairs(data)
    end = time.perf_counter()
    print("improved hash table:", end - start)

    start = time.perf_counter()
    _ = dict(data)
    end = time.perf_counter()
    print("normal hash table:", end - start)

def compare_retrieval(d1: SipDict[int, int], d2: dict, item: int):
    start = time.perf_counter()
    _ = d1[item]
    end = time.perf_counter()
    print("improved hash table:", end - start)

    start = time.perf_counter()
    _ = d2[item]
    end = time.perf_counter()
    print("normal hash table:", end - start)
{% endhighlight %}

### Constructing hash tables
Constructing hash table with collisions:

Benchmark code:
{% highlight python %}
p = 2**61 - 1
data = [(i * p + 1, i) for i in range(1000)]

compare_initialization(data)
{% endhighlight %}

Results:
{% highlight none %}
improved hash table: 0.0006913679972058162
normal hash table: 0.015330620997701772
{% endhighlight %}

Constructing hash table without collisions:

{% highlight python %}
p = 2**61 - 1
data = [(random.randint(1, 2 ** 64), i) for i in range(1000)]

compare_initialization(data)
{% endhighlight %}

Results:
{% highlight none %}
improved hash table: 0.0008386349945794791
normal hash table: 0.00017141499847639352
{% endhighlight %}

Note that the new hash table is many times faster than the normal one when initializing with lots of collisions, however is slower than the normal one without collisions. One thing to note is that both hashes are applied, CPython's modular reduction and SipHash, making it slow down more. Another reason is that the interpreter has to repeatedly add more memory, whilst CPython creates a presized dict using `dict_new_presized`.

### Retrieving from hash tables
Retrieval with collisions:

Benchmark code:
{% highlight python %}
data = [(p * i + 1, i) for i in range(1000)]
d1 = SipDict.from_pairs(data)
d2 = dict(data)

compare_retrieval(d1, d2, 874 * p + 1)
{% endhighlight %}

Results:
{% highlight none %}
improved hash table: 7.757989806123078e-06
normal hash table: 1.6758000128902495e-05
{% endhighlight %}

Retrieval without collisions:
{% highlight python %}
data = [(i * (2**53), i) for i in range(1000)]
d1 = SipDict.from_pairs(data)
d2 = dict(data)

compare_retrieval(d1, d2, 874 * (2**53))
{% endhighlight %}

Results:
{% highlight python %}
improved hash table: 4.636996891349554e-06
normal hash table: 7.43995769880712e-07
{% endhighlight %}

## Conclusion
Is the use-case contrived? Possibly. May it actually happen? Also possibly. The above implementation of a slightly (?) (citation needed) better hash table is much slower for initialization (although its probably because of the memory allocation costs) and retrieval (this could probably be fixed if I edited the CPython source code, but honestly, i don't care enough). Like before, if this for some reason is a concern at all, go ask a professional.

## Disclaimer
The info in this blog post has tried to be accurate, yet there may be some issues and mistakes. If you find any, mention me on Mastodon at somehybrid@hachyderm.io and I might have enough motivation to actually fix any of it.
