---
title: "Consuming SOAP Services with Clojure Macros"
date: 2015-10-30
tags: ["clojure", "macros", "soap", "interop"]
---


Two colleagues were consuming a SOAP service in Clojure and ran into some trouble. Being fairly new to Clojure, they struggled with mapping Java class hierarchies into Clojure keyword maps. They didn’t use the built-in `bean` function—partly because they didn’t know about it, and partly because they preferred a more controlled transformation.

They generated the classes from WSDL using `wsimport`.

### The Problem

Let’s consider a simplified SOAP service implemented in Java:

```java
public class Foo {
  public final String name;
  public final String lastName;
  public Foo(String name, String lastName) {
    this.name = name;
    this.lastName = lastName;
  }
  public String getFullName() {
    return name + " " + lastName;
  }
}

public class Bar {
  private List<Foo> foos;
  public final int noBaz;
  private final String quote;
  public Bar(List<Foo> foos, int noBaz, String quoteOfTheDay) {
    this.foos = foos;
    this.noBaz = noBaz;
    this.quote = quoteOfTheDay;
  }
  public Iterable<Foo> foos() {
    return new ArrayList<>(foos);
  }
  public String getQuoteOfTheDay() {
    return quote;
  }
}
```

Manually translating an instance of `Bar` into a Clojure map would look like this:

```clojure
{:no-baz (.-noBaz bar)
 :quote-of-the-day (.getQuoteOfTheDay bar)
 :foos (map (fn [foo]
              {:name (.name foo)
               :last-name (.-lastName foo)
               :full-name (.getFullName foo)})
            (.foos bar))}
```

It’s verbose and error-prone. So I wrote a few macros to simplify it.

### Introducing `auto-key-map`

This macro extracts fields and methods and automatically creates kebab-case keywords:

```clojure
(auto-key-map (Foo. "Arne" "Weise")
              .-name
              .-lastName
              .getFullName)
;; => {:name "Arne" :last-name "Weise" :full-name "Arne Weise"}
```

We also define macros for handling collections:

```clojure
(auto-key-map-each bar .foos
                        .-name
                        .-lastName
                        .getFullName)
```

This produces a map with a key derived from the accessor (`:foos`), and a sequence of extracted maps.

### Full Example

```clojure
(merge (auto-key-map bar
                     .-noBaz
                     .getQuoteOfTheDay)
       (auto-key-map-each bar .foos
                               .-name
                               .-lastName
                               .getFullName))
```

Result:

```clojure
{:no-baz 32
 :quote-of-the-day "Try simplicity"
 :foos ({:name "Alan" :last-name "Boris" :full-name "Alan Boris"}
        {:name "Ceasar" :last-name "Davis" :full-name "Ceasar Davis"}
        {:name "Elvis" :last-name "Foley" :full-name "Elvis Foley"})}
```

If you don’t want the collection as a map entry, use `auto-key-map-seq`:

```clojure
(assoc (auto-key-map bar
                     .-noBaz
                     .getQuoteOfTheDay)
       :foos-vec (vec (auto-key-map-seq (.foos bar)
                                        .-name
                                        .-lastName
                                        .getFullName)))
```

### Custom Naming with `key-map`

Sometimes, you want manual control:

```clojure
(key-map (Foo. "Arne" "Weise")
         :family .-lastName
         :all .getFullName)
;; => {:family "Weise" :all "Arne Weise"}
```

Supports `some->` threading:

```clojure
(key-map (Foo. "Arne" "Weise")
         :len [.-name .length])
;; => {:len 4}
```

### Nested Structures

```clojure
(merge (key-map-each-fn bar .foos :foos
                                :first-len [.-name .length]
                                :last .-lastName
                                :full-name .getFullName)
       (auto-key-map bar .-noBaz .getQuoteOfTheDay))
```

Result:

```clojure
{:no-baz 32, :quote-of-the-day "Try simplicity",
 :foos ({:first-len 4, :last "Boris", :full-name "Alan Boris"}
        {:first-len 6, :last "Davis", :full-name "Ceasar Davis"}
        {:first-len 5, :last "Foley", :full-name "Elvis Foley"})}
```

These macros help create readable Clojure maps from Java interop, reduce boilerplate, and are useful well beyond SOAP parsing.

