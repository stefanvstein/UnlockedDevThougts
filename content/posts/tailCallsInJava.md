---
title: "Tail Call Recursion in Java"
date: 2015-10-08
tags: ["Java", "Functional"]
---


### Tail Call Recursion in Java

A colleague of mine faced a problem that he really wanted to solve using recursion. The catch? He was using Java, which doesn’t support tail call optimization. I suggested he use a trampoline to simulate recursive calls. After a few examples online fell short, we sat down and came up with the following solution:

```java
package trampoline;
import java.util.function.Function;

public class Trampoline {
  public static class RecursiveVal<V> {
    public final Supplier<RecursiveVal<V>> fun;
    public final V v;

    private RecursiveVal(Supplier<RecursiveVal<V>> fun, V v) {
      this.fun = fun;
      this.v = v;
    }

    private static <V> RecursiveVal<V> nil() {
      return new RecursiveVal<V>(null, null);
    }
  }

  static <V> RecursiveVal<V> recur(Supplier<RecursiveVal<V>> f) {
    return new RecursiveVal<V>(f, null);
  }

  static <V> RecursiveVal<V> done(V v) {
    return new RecursiveVal<V>(null, v);
  }

  public static <V> V trampoline(Supplier<RecursiveVal<V>> fun) {
    RecursiveVal<V> ret = RecursiveVal.nil();
    while (fun != null) {
      ret = fun.get();
      fun = ret.fun;
    }
    return ret.v;
  }
}
```

### Rewriting a Recursive Function

Let’s take a traditional stack-blowing recursive function:

```java
static String foo(int a) {
  return a > 200000 ? "Foo" : foo(a + 1);
}
```

We can rewrite it using the trampoline pattern like this:

```java
static Supplier<RecursiveVal<String>> foo(int a) {
  return () -> a > 200000 ? done("Foo") : recur(foo(a + 1));
}
```

This uses a slightly more complex signature and explicitly states when we recur.
To run it:

```java
trampoline(foo(0));
```

### Mutual Recursion Example

Mutual recursion also works cleanly:

```java
static Supplier<RecursiveVal<Boolean>> isEven(int a) {
  return () -> a == 0 ? done(true) : recur(isOdd(a - 1));
}

static Supplier<RecursiveVal<Boolean>> isOdd(int a) {
  return () -> a == 0 ? done(false) : recur(isEven(a - 1));
}

trampoline(isEven(200000));
```

### Why Use Recursive Loops?

So what’s the benefit of recursive looping instead of traditional loops? It reduces complexity by minimizing side effects. With trampolines, recursion becomes stack-safe and can be clearer in expressing intent—especially when modeling computations that naturally unfold recursively.

