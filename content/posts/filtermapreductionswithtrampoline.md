---
title: "Filter, Map, Reductions with Trampoline"
date: 2017-02-17
slug: trampoline-iterables
---


Use recursion with a trampoline to implement lazy versions of map, filter and reduce.

In Java 8 we got the `Stream` type, allowing functional transformations on stream elements. Streams are not classic collections—we don’t know the traversal order. But since pure functions don’t rely on order, this enables parallelism. 

However, sometimes we want control. We may wish to transform elements in sequence, filter, take some, or fold them. Java Streams are classes with a fixed API. Adding new kinds of iteration becomes awkward—especially compared to Clojure’s delightful sequence library.

Let’s see how we could use a **trampoline** to mimic lazy sequences in Java, avoiding verbose boilerplate and keeping our code functional and expressive.

---

## `map` using boilerplate

```java
public static <A, V> Iterable<V> map(
  Function<? super A, ? extends V> f,
  Iterable<? extends A> data) {
  if(f == null || data == null)
    return null;
  return new LazyMapIterable<>(f, data);
}

private static final class LazyMapIterable<V, A> implements Iterable<V> {
  private final Function<? super A, ? extends V> f;
  private final Iterable<? extends A> data;

  public LazyMapIterable(Function<? super A, ? extends V> f, Iterable<? extends A> data) {
    this.f = f;
    this.data = data;
  }

  @Override public Iterator<V> iterator() {
    return new LazyMapIterator<>(data.iterator(), f);
  }
}

private static final class LazyMapIterator<V, A> extends ImmutableIterator<V> {
  private final Iterator<? extends A> i;
  private final Function<? super A, ? extends V> f;

  public LazyMapIterator(Iterator<? extends A> i, Function<? super A, ? extends V> f) {
    this.i = i;
    this.f = f;
  }

  @Override public boolean hasNext() {
    return i.hasNext();
  }

  @Override public V next() {
    if (i.hasNext())
      return f.apply(i.next());
    throw new NoSuchElementException();
  }
}
```

That’s a lot of boilerplate for a single line: `return f.apply(i.next());`

Let’s simplify.

---

## Trampoline-based lazy sequences

A **trampoline** avoids stack overflows in recursive calls. Here's the idea:

```java
public class Trampoline {
  static <V> V trampoline(Supplier<RecursiveVal<V>> fun);
  static <V> RecursiveVal<V> recur(Supplier<RecursiveVal<V>> f);
  static <V> RecursiveVal<V> done(V v);
  static <V> Iterable<V> lazy(Supplier<RecursiveVal<V>> fun);
  static <V> RecursiveVal<V> seq(Supplier<RecursiveVal<V>> f, V v);
  static <V> RecursiveVal<V> stop();
}
```

- `recur` indicates continuation
- `done` indicates final value
- `seq` yields a value and continues
- `stop` ends the sequence

---

## Lazy `map`

```java
private static <A, V> Iterable<V> map(Function<? super A, V> f, Iterable<A> data) {
  if (data == null || f == null)
    return null;
  return () -> {
    Iterator<A> i = data.iterator();
    return i.hasNext() ? lazy(mapI(f, i)).iterator() : emptyIterator();
  };
}

private static <A, V> Supplier<RecursiveVal<V>> mapI(Function<? super A, ? extends V> f, Iterator<A> i) {
  return () -> {
    V v = f.apply(i.next());
    return i.hasNext() ? seq(mapI(f, i), v) : done(v);
  };
}
```

---

## Lazy `filter`

```java
private static <T> Iterable<T> filter(Predicate<? super T> pred, Iterable<T> data) {
  if (data == null || pred == null)
    return null;
  return () -> {
    Iterator<T> i = data.iterator();
    return i.hasNext() ? lazy(filterI(pred, i)).iterator() : emptyIterator();
  };
}

private static <T> Supplier<RecursiveVal<T>> filterI(Predicate<? super T> pred, Iterator<T> i) {
  return () -> {
    if (!i.hasNext()) return stop();
    T t = i.next();
    return pred.test(t) ? seq(filterI(pred, i), t) : recur(filterI(pred, i));
  };
}
```

---

## Lazy `reductions`

```java
public static <A, V> Iterable<V> reductions(BiFunction<? super V, ? super A, V> f, V init, Iterable<? extends A> data) {
  if (data == null || f == null)
    return null;
  return () -> lazy(reductionsI(f, init, data.iterator())).iterator();
}

private static <A, V> Supplier<RecursiveVal<V>> reductionsI(BiFunction<? super V, ? super A, V> f, V acc, Iterator<? extends A> i) {
  return () -> {
    if (!i.hasNext()) return stop();
    V newAcc = f.apply(acc, i.next());
    return i.hasNext() ? seq(reductionsI(f, newAcc, i), newAcc) : done(newAcc);
  };
}
```

---

## Example: Compose `filter`, `map` and `reductions`

```java
Iterable<Integer> result =
  reductions((sum, x) -> sum + x, 0,
    map(x -> x * 2,
      filter(x -> x % 2 == 0, List.of(1, 2, 3, 4, 5))));

for (int val : result) {
  System.out.println(val);
}
```

This outputs:
```
4
12
20
```

---

## Summary

Using a trampoline allows us to:

- avoid stack overflows with recursion
- implement lazy iteration without boilerplate
- emulate Clojure’s `map`, `filter`, `reductions`, and more in Java

By splitting each transformation into a **bootstrap** method and a **recursive core**, we can elegantly process data in a functional, lazy, and expressive way—even in Java.

Java doesn’t need to be verbose. It just needs better habits. 😉

