---
title: "Fibo"
date: 2012-12-18
tags: ["clojure", "haskell", "java", "C#"]
---

In a study group discussing Domain-Driven Design (DDD), someone jokingly remarked that DDD doesn't suit functional programming (FP), saying something along the lines of: "Because in FP you use a language where you implement the Fibonacci sequence using recursion." It might have been another typical calculation used to introduce FP concepts. I can’t recall exactly.

Anyway, here's an example of calculating Fibonacci in Clojure:

```clojure
(defn fib []
  ((fn fibo [a b]
       (cons a (lazy-seq (fibo b (+ a b)))))
    0 1))
```

I don't expect everyone in the study group to follow half of what I'm saying here, but for those interested:

This example goes straight to the point and implements the Fibonacci sequence at a pretty low level. It’s short and elegant, albeit cryptic if you're not familiar with Clojure. We clearly express that the sequence is lazy, which matters since Fibonacci extends infinitely and the caller may only want part of it. Being lazy means values are computed only when needed.

Below, we take 50 values from the sequence by calling `fib` and using `take`. That means we don't compute more than 50 numbers:

```clojure
(take 50 (fib))
```

Here’s a version I find more aesthetically pleasing. It still generates the sequence lazily but uses functions like `map` and `iterate`, which many FP developers are familiar with:

```clojure
(defn fib []
  (let [fibo (fn [[a b]]
               [b (+ a b)])]
    (map first (iterate fibo [0 1]))))
```

Again, we can take 50 lazy values from it the same way:

```clojure
(take 50 (fib))
```

Haskell is fun when it comes to Fibonacci. It’s often used to explore deeper FP thinking. Haskell is lazy by nature, so arguments to functions aren't evaluated until needed. This naive Fibonacci implementation in Haskell looks just like the mathematical definition:

```
fib 0 = 0
fib 1 = 1
fib n = fib (n-1) + fib (n-2)
```

And we can get the first 50 numbers using:

```
take 50 (map fib [0..])
```

This is naturally lazy and general since we define the range starting from index 0. Elegant. But this version recalculates values repeatedly, which is fine mathematically, but computationally expensive. It slows down quickly. This optimized version avoids that but isn't as aesthetically pleasing (unless you’re a devoted Haskeller):

```
fib n = fibs !! n
  where
    fibs = 0 : 1 : zipWith (+) fibs (tail fibs)

take 50 (map fib [0..])
```

In C#, it's also easy to implement Fibonacci lazily:

```java
IEnumerable<ulong> fibo()
{
  ulong a = 0, b = 1, c;
  yield return a;
  yield return b;
          
  while (true)
  {
    c = a + b;
    a = b;
    b = c;
    yield return c;
  }
}
```
It’s not functional, and it crashes eventually because C# doesn’t automatically promote primitive types to arbitrary-precision like most FP languages do. Still, we can take the first 50 values similarly:

```
Enumerable.Take(fibo(), 50);
```

You could surely improve this C# code. I don't have many years of experience in C#.

Then we have Java. Calculating Fibonacci in Java is straightforward, but doing it lazily with only the JDK is verbose due to lack of built-in laziness:

```java
class Fib implements Iterator<Long>, Iterable<Long> {
  long a = 0, b = 1, c = 0;
  int count = 0;

  @Override
  public Long next() {
    count++;
    if (count == 1)
      return a;
    else if (count == 2)
      return b;
    else {
      c = a + b;
      a = b;
      b = c;
    }
    return c;
  }

  @Override
  public boolean hasNext() {
    return true;
  }

  @Override
  public void remove() {
    throw new UnsupportedOperationException();
  }

  @Override
  public Iterator<Long> iterator() {
    return new Fib();
  }
```

We don't even have a take function, so:

```java
static <T> Iterable<T> take(final int num, final Iterable<T> iterable) {
    return new Iterable<T>() {

      @Override
      public Iterator<T> iterator() {
        return new Iterator<T>() {
          int i = 0;
          Iterator<T> it = iterable.iterator();

          @Override
          public boolean hasNext() {
            return it.hasNext() && i < num;
          }

          @Override
          public T next() {
            if (hasNext()) {
              i++;
              return it.next();
            }
            throw new NoSuchElementException();
          }

          @Override
          public void remove() {
            it.remove();
          }
        };
      }
    };
  }
```

Finally:

```
take(50, new Fib())
```
Nothing wrong with Java, but the JDK lacks general composable functions, which I noted more than once in my Java 7 talk. Hopefully closures in Java 8 help.

There are plenty of 3rd party libraries with functional tools, but not in the otherwise bloated JDK.

We also discussed a more recent interview with Eric Evans, which is worth checking out:

http://www.infoq.com/interviews/Technology-Influences-DDD

Here’s what he said about FP and DDD:

*What about functional programming. How do you see DDD mapping to this new world? 
Yes, well, again, I think that what you need in order to do DDD is a powerful abstraction tool, something that allows you to take that ubiquitous language and render it in code. So, in functional, we certainly have the abstraction power and a lot of people in functional programming tend to think in terms of basically creating their own internal DSLs, that is Domain Specific Languages. So that’s one style of programming which people in functional tend to use and say, well, build up a language basically that you can use to describe your domain. So that’s probably one of the areas I expect we’ll work out. We’ll end up with more DSL-like style. And I think this kind of internal DSLs was one of the popular styles within the OO world although it never spread so widely, but it was influential.*

*I think another is that when you start looking at problem, for example, I’ve been looking at functional programming for the last few years and I’ve done a little bit of work in Clojure, for example and partly because I really feel, I mean, you know, I want to, I get excited about new stuff too, and so I wanted to get my hands on, you know, something functional. I chose Clojure because it’s a really pure functional language and you can’t cop out and do it the OO way in Clojure, so it forces you to really think about it as a functional problem.
And one thing I noticed, and in retrospect this doesn’t seem surprising to me at all, is that the nature of the models and the way that I approach the problems starts to change. And, one of the things I noticed is that I’m much more likely to view things as sort of streams and sequences. So you look at your domain and you start looking for the things where you’ve got a stream of things happening or stream of things you can process. Because functional language is sort of structured that way and you’ve got a function and you fit it as stream and out comes another stream of something.*

*I guess my point is, that is has the abstraction power that we need. There is, if you take the mentality of how can I express the ubiquitous language here and what shape should my model take? If you’re really open-minded about those things you’ll find ways of creating nice DDD solutions with functional and then we’ll need to do patterns and variations of the old patterns. Some of them are natural like value objects which was always one of the things I used to preach about a lot. But, in OO you have to kind of say, you know, make an object like this. In functional you almost start there. That’s the default way that a data structure, at least, would be. It is not an object and you still have to say, “Well these functions and this data structure together accomplish this thing.” And in Clojure, the concept of a protocol is pretty clear. So anyway, value objects map over pretty easily, very naturally.*

*Then entity, what is an entity? The identity part of an entity is still relevant, but some of the other aspects of it may just not apply. Domain events? I think that’s going to be one of the real keys to making DDD work in functional is to think more in terms basically of event sourcing which was one of the, you know, has been one of the big trends in DDD for the last several years. And now, when I start looking at problems in the functional world I say, “Ah, event sourcing actually fits pretty well.” A sequence of events again, that stream, being processed to create basically a current view of an entity. So the entity isn’t there as such, it’s not stored state, but if you want to know what a particular entity looks like at a particular point of time, you take the stream of events you put it together, I mean that’s event sourcing. Works pretty well with functional, actually. So we’re just at the beginning of this, you know, that’s the exciting thing, it’s fun. I mean, well, we’re not really at the beginning I’m sure because functional is an old, old style of programming, but we’re at the beginning of maybe of widespread adoption.*

It’s great that he appreciates elegant FP languages :-)

In the Clojure presentation I gave earlier, we eventually arrived at the concept of protocols after a journey through functional ideas and Lisp’s unusual syntax.

A protocol is a definition of free-standing functions that can dispatch based on type—polymorphism, as we're used to from OO. In functional programming, polymorphism can be over anything, but since Clojure is designed to run on hosted platforms, dispatching on type is a good choice—both for performance and interoperability.

A protocol is similar to an interface (a purely abstract data type) but doesn’t participate in any type hierarchy. It’s free-standing. For example, a protocol for a bank account might look like:

```clojure
(defprotocol account-protocol
  (deposit [this amount])
  (withdraw [this amount]))
```
…which defines two free functions: deposit and withdraw.

Each function takes an instance of a type as its first argument. You can use various kinds of types here. The Type type is general and powerful—it can even contain (heaven forbid!) mutable state. Anonymous reifications are concise, while records are designed to serve as domain types.

 ```clojure
 (defrecord account [balance transactions])
 ```

This defines the domain data type account, containing a balance and a list of transactions—probably a number and a sequence.

Clojure uses associative types like hash maps pervasively. We often replace classes with maps of names and values (or functions). After all, a class in a dynamic language is conceptually just a named bag of data. Object hierarchies become nested maps.

Here’s a definition:

```clojure
(def olle {:name "Olle",
           :last-name "Andersson",
           :details {:age 23, :favorite-dish "Pizza"},
           :adress {:street "Strandvägen", :num 43}}
```

You can update nested structures:

```clojure

(assoc-in olle [:details :favorite-dish] "Paneng gai")
```clojure

Or:


```clojure
(update-in a [:details :age] inc)
```
A record is a type that behaves like a map but is compiled to a class on the host platform, offering Java-level performance and optional type guarantees. It still works with all the associative and sequence functions.
```clojure
(def a (account. 23.4 [])) 
(assoc a :balance 45.2)
``` 
We used (assoc map key value)—a general function for adding values to associative collections. The account data type is still a domain type and also happens to be a Java class.
```clojure
(assoc {} :foo "Bar") ;=> {:foo "Bar"}
``` 
Records can be nested and extended with new keys like regular maps. Records are immutable, so you always get new values—no surprises.

Back to the protocol: to make the account record implement the account-protocol:

```clojure
(defrecord account [balance trans]
  account-protocol
  (deposit [this value]
    (assoc this :balance (+ balance value)
                :trans (conj trans value)))
  (withdraw [this value]
    (assoc this :balance (- balance value)
                :trans (conj trans (- value)))))
```

The implementations return new account values with updated balances and transactions. You can still treat the account like a map or use the protocol functions for domain behavior.
```clojure
(deposit (account. 32 []) 43) ;=> {:balance 75 :transactions [43]} 
```

You can even extend account-protocol to Java types:

```clojure
(extend-type java.lang.String
  account-protocol
   (deposit [_ v] (str v))
   (withdraw [_ v] (str v)))
(deposit "Java" "Clojure") ;=> "Clojure"
```

We made String behave like an account—odd, but illustrative. In most OO languages, extending foreign hierarchies like this is difficult. You usually need adapters, which break identity. This is known as the expression problem: combining two hierarchies is hard.

In FP, we flip the model: functions solve this cleanly. We didn’t change the String class—we just added dispatch logic.

Modeling like this is why: what takes Coca Cola and Marlboro Lights in OO takes Cognac and a fat cigar in FP. You're no longer constrained by OO’s rough edges.

Another FP advantage: immutable data is easy to manage, making modeling simpler (not necessarily easier).

F#, Haskell, and other ML-based languages are just as (or more) elegant for domain modeling as Clojure—but they come with strong static types, which can be a hurdle before you get used to them. Sometimes they feel a bit strict, in my view.
