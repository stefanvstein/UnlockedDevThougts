---
title: "Functional Composition and Threading Calls in Java"
date: 2016-03-08
slug: "java-composition"
tags: ["java", "functional", "lambdas", "clojure"]
---

With the simplified functional programming capabilities that have been available in Java for a while now, many of my friends are becoming more productive—even in Java. Methods are increasingly being written as free functions, often without side effects, which boosts productivity. It's easier to compose functionality, even though defining new types is still simpler with object-oriented approaches.

Expressions are becoming clearer and easier to reason about. For example:

```java
plus(4, inc(len("Text")));
```

Most readers would recognize that this evaluates to `9`. The equivalent in Clojure would be:

```clojure
(+ 4 (inc (count "Text")))
```

However, as expressions grow, they become harder to read because they're written in reverse order relative to their evaluation. Functional languages like Clojure solve this with threading macros. For example:

```clojure
(->> (count "Text")
     (inc)
     (+ 4))
```

In Java, we could simulate this with:

```java
thread("Text",
       Ex::len,
       Ex::inc,
       partial(Ex::plus, 4));
```

Even though `thread` is not part of the standard library (and certainly not a macro), the result is an expression written in evaluation order. `partial` here performs partial application, returning a new function where the first argument has been pre-applied. For example:

```java
public static <T, U, V> Function<U, V> partial(
   BiFunction<T, U, V> f,
   T t) {
  return u -> f.apply(t, u);
}
```

Below are the function definitions with obvious implementations. The `toNil` method will be used later:

```java
public class Ex {
    public static int len(String s) {
        return s.length();
    }

    public static int inc(int x) {
        return x + 1;
    }

    public static int plus(int x, int y) {
        return x + y;
    }

    public static Integer toNil(Object o) {
        return null;
    }
}
```

### Using Function Composition

Another way to write the expression in evaluation order is through function composition. Using a `compose` function like:

```java
public static <A, B, V> Function<A, V> compose(
   Function<? super A, ? extends B> f1,
   Function<? super B, ? extends V> f2) {
  return a -> f2.apply(f1.apply(a));
}
```

We can write:

```java
Function<String, Integer> f =
  compose(Ex::len,
    compose(Ex::inc,
      partial(Ex::plus, 4)));
f.apply("Text");
```

This achieves what we want—although it's a bit nested.

With Java 8, the `Function` interface includes the `andThen` method for function composition:

```java
Function.<String>identity()
  .andThen(Ex::len)
  .andThen(Ex::inc)
  .andThen(partial(Ex::plus, 4))
  .apply("Text");
```

That removes nesting, but the repeated `andThen` calls are verbose. Let’s fix that.

### A Threading Function for Java

The `thread` function can be implemented imperatively:

```java
public static <T0, T1, T2, T3>
T3 thread(T0 t0,
          Function<T0, T1> f0,
          Function<T1, T2> f1,
          Function<T2, T3> f2) {
    T1 t1 = f0.apply(t0);
    T2 t2 = f1.apply(t1);
    return f2.apply(t2);
}
```

Each overload handles a fixed number of functions. Since every function has its own type parameters, varargs isn't a good fit. Ten overloads should cover most cases.

Similarly, for two functions:

```java
public static <T0, T1, T2>
T2 thread(T0 t0,
          Function<T0, T1> f0,
          Function<T1, T2> f1) {
    T1 t1 = f0.apply(t0);
    return f1.apply(t1);
}
```

This avoids altering the core `Function` type, preserving separation of concerns and avoiding the expression problem.

### Threading With Null-Safety: `threadMaybe`

Clojure has `some->>` for null-safe threading. We can create something similar:

```java
threadMaybe("Text",
            Ex::len,
            Ex::toNil,
            Ex::inc);
```

`Ex::inc` will not be called because `Ex::toNil` returns null.

Here's the implementation:

```java
public static <T0, T1, T2, T3>
T3 threadMaybe(T0 t0,
               Function<T0, T1> f0,
               Function<T1, T2> f1,
               Function<T2, T3> f2) {
    if (t0 == null) return null;
    T1 t1 = f0.apply(t0);
    if (t1 == null) return null;
    T2 t2 = f1.apply(t1);
    if (t2 == null) return null;
    return f2.apply(t2);
}
```

### Conditional Logic With Fewer Boilerplate

This pattern also applies to the `cond` function from the previous post. With different arities, we avoid manually creating an iterable and the catch-all guard:

```java
public static <T> T cond(
   Supplier<Boolean> t0, Supplier<? extends T> v0,
   Supplier<Boolean> t1, Supplier<? extends T> v1,
   Supplier<Boolean> t2, Supplier<? extends T> v2,
   Supplier<? extends T> v3) {
   if (t0 != null && Boolean.TRUE.equals(t0.get()))
     return v0 == null ? null : v0.get();
   else if (t1 != null && Boolean.TRUE.equals(t1.get()))
     return v1 == null ? null : v1.get();
   else if (t2 != null && Boolean.TRUE.equals(t2.get()))
     return v2 == null ? null : v2.get();
   return v3 == null ? null : v3.get();
}
```

---

Functional composition doesn't have to be reversed—even in Java. Threading-style expressions can be clearer and more declarative. It might take a few helper functions, but the result is more readable and easier to reason about.

_Side note: Let's continue making Java just a little more functional._

