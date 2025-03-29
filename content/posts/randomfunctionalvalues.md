---
title: "Random Functional Values"
date: 2016-01-20
tags: ["clojure", "randomness", "functional"]
---

Functional random values, or a sequence of well-spread values.

Did some evening Clojuring with a bunch of my padawans and friends. Everything looked good—free of side effects, single state, mastery—until randomness suddenly appeared as a hidden input.

I quickly recommended passing randomness explicitly as input, perhaps stored in the single state and continuously transformed. This way, randomness becomes side-effect free and repeatable. The next random value is simply a deterministic transformation of the previous one.

We could use a `java.util.Random` sequence with an explicit seed like:

```clojure
(defn randoms [seed]
  (let [r (java.util.Random. seed)]
    (repeatedly #(.nextLong r))))
```

But architectural principles suggest that state should contain simple, serializable data. A serialized `java.util.Random` includes more than just a seed—hidden internal state complicates things.

Instead, we just need a sequence that is repeatable and appears random. If each new value is the seed for the next, we only need to store a single value.

George Marsaglia's fast random algorithm (http://www.jstatsoft.org/article/view/v008i14) fits perfectly:

```clojure
(defn reasonable-spread [^long i]
  (let [xor-right (fn [x n] (bit-xor x (bit-shift-right x n)))
        xor-left (fn [x n] (bit-xor x (bit-shift-left x n)))]
    (if (= 0 i)
      1
      (-> (xor-left i 21)
          (xor-right 35)
          (xor-left 4)))))
```

By feeding the result of `reasonable-spread` back into itself, we generate a deterministic, well-spread pseudo-random sequence. A `long` value is easy to pass around in most serialization formats.

For convenience, here's how to produce an infinite sequence:

```clojure
(defn reasonable-spreads [^long i]
  (iterate reasonable-spread (reasonable-spread i)))
```

This approach gives you functional-style randomness—repeatable, side-effect-free, and architecture-friendly.

