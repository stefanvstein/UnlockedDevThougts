---
title: "Java `cond` Expression"
date: 2016-03-06
slug: "java-cond"
tags: ["java", "functional", "lambdas", "clojure"]
---

## Java `cond`

### Implementing `cond` in Java

A friend of mine had to switch from writing Clojure to writing Java at his new job. Poor soul. One day, while staring at a mess of nested `if` statements, he sighed:

> “If only Java had `cond`.”

For those unfamiliar, Clojure's `cond` is like a more expressive version of `switch-case`, returning a single value based on a list of condition-result pairs.

Here's a simple Clojure example:

```clojure
(let [x 2]
  (cond
    (= x 1) "One"
    (= x 2) "Two"
    :otherwise "Many"))
```

Which evaluates to `"Two"`.

---

### The Java Equivalent

I figured it wouldn't be too hard to implement something similar using Java 8 lambdas. Here's the Java version:

```java
int t = 2;
String b = cond(asList(
  of(() -> t == 1, () -> "One"),
  of(() -> t == 2, () -> "Two"),
  of(() -> true,   () -> "Many")
));
```

Each pair consists of a condition (`Supplier<Boolean>`) and a result (`Supplier<T>`), which are lazily evaluated.

To make this work, we define a generic `Tuple` class:

```java
import static java.util.Arrays.asList;
import java.util.Objects;

public class Tuple<F, S> {
  public final F first;
  public final S second;

  private Tuple(F f, S s) {
    first = f;
    second = s;
  }

  public static <F, S> Tuple<F, S> of(F first, S second) {
    return new Tuple<>(first, second);
  }

  @Override
  public boolean equals(Object obj) {
    if (this == obj) return true;
    if (!(obj instanceof Tuple)) return false;
    Tuple<?, ?> other = (Tuple<?, ?>) obj;
    return Objects.equals(first, other.first) &&
           Objects.equals(second, other.second);
  }

  @Override
  public int hashCode() {
    return Objects.hash(first, second);
  }

  @Override
  public String toString() {
    return "[" + first + ", " + second + "]";
  }
}
```

Now for the `cond` logic itself:

```java
import static java.util.Objects.nonNull;
import static java.util.Optional.ofNullable;
import java.util.function.Supplier;

class Logic {
  public static <T> T cond(Iterable<Tuple<Supplier<Boolean>, Supplier<T>>> list) {
    if (nonNull(list)) {
      for (Tuple<Supplier<Boolean>, Supplier<T>> choice : list) {
        if (ofNullable(choice.first).map(Supplier::get).orElse(false)) {
          return ofNullable(choice.second).map(Supplier::get).orElse(null);
        }
      }
    }
    return null;
  }
}
```

---

### Usage Example

```java
import static Logic.cond;
import static Tuple.of;
import static java.util.Arrays.asList;

int t = 2;
String b = cond(asList(
  of(() -> t == 1, () -> "One"),
  of(() -> t == 2, () -> "Two"),
  of(() -> true,   () -> "Many")
));
```

The value of `b` will be `"Two"`.

This approach gives you the expressive, functional feel of Clojure's `cond` while staying within the confines of Java 8. No more nested ifs. Just elegant branching.

Happy coding!

