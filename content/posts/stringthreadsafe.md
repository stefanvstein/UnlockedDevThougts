---
title: "Strings: Thread Safe at Last"
date: 2005-05-22
tags: ["Java", "Threads", "JMM"]
---

Yes! The class `java.lang.String` wasn’t officially thread-safe—according to the Java specification—until Java 1.5.

Now, `String` is designed to be immutable, which inherently *should* make it thread-safe. Its internal state is set during construction and can't be changed afterward. Internally, a `String` consists of a character array and two integers that define the range of characters representing the string. All these fields are declared `final`.

The issue? Version 2 of the *Java Language Specification*, specifically chapter 17, didn’t mention anything about `final` fields. This meant we couldn’t guarantee that another thread would see a properly initialized object, unless additional memory barriers (like `synchronized`) were in place. Without those, visibility of final fields across threads wasn’t ensured.

Take the `substring()` method as an example: it returns a new `String` object that shares the same underlying character array but with different range integers. Neither the constructor nor `substring()` uses synchronization. So if one thread constructs a substring and passes it to another, there's a risk that the receiving thread might see an inconsistent state—a string that doesn’t represent the expected value. Immutability alone wasn’t enough.

But with Java 5 (and the third edition of the spec), that changed. The language now guarantees that `final` fields are visible to other threads *after* the constructor finishes. This means that any other thread accessing the newly constructed object is guaranteed to see the fully initialized values.

So finally—yes, *finally*—`String` is thread-safe, even by the book.


