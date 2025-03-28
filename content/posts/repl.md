---
title: "REPL: Like art"
date: 2013-05-25
tags: ["Clojure", "Art"]
---


A REPL—short for *Read, Eval, Print, Loop*—is the primary development tool in dynamically compiled languages. REPLs are heavily used in languages like Ruby, Python, JavaScript, F#, Scala, Haskell, and especially in the Lisp family of languages. A REPL lets you try out expressions and forms the foundation of test-driven development in its purest form. An editor with a REPL is an IDE in its most minimal and distilled form.

Below is a primitive REPL implementation in Clojure, a Lisp dialect:

```clojure
(defmacro Loop [& body] `(loop [] ~@body (recur)))
(let [print println]
  (-> (read) (eval) (print) (Loop)))
```

Sure, it reads easily—`read`, `eval`, `print`, `loop`—spelled out in plain text. But it’s harder to grasp what’s actually happening.

The same logic can be written even shorter as:

```clojure
(->> (read) (eval) (println) (while true))
```

...but that can’t be read aloud as *REPL*.

The first line:

```clojure
(defmacro Loop [& body] `(loop [] ~@body (recur)))
```

...is a macro defining unconditional iteration, like `(while true)` but with no condition. More accurately, it recursively iterates over the expansion of the body list passed to `looop`. The built-in `loop` establishes a recursion point, and `recur` jumps back to it in a tail-recursive manner. Since `Loop` is a macro, a code pattern rather than a function, we need `loop` to define the known target for recursion. We named it `Loop` because `loop` already exists in Clojure, and redefining it would be bad hygiene.

The element `while` is also a macro, which could be written similarly but with a condition:

```clojure
(defmacro while [test & body] `(loop [] (when ~test ~@body (recur))))
```

So the `Loop` macro could be implemented as:

```clojure
(defmacro Loop [& body] `(while true ~@body))
```

This line:

```clojure
(let [print println] ...)
```

...binds the function `println` to the local name `print` within the expression body.

And finally, our main expression:

```clojure
(-> (read) (eval) (print) (Loop))
```

...is passed as the body of the `let`, where `println` is now also named `print`.

This is also a macro. It expands like this:

```clojure
(Loop (-> (read) (eval) (print)))
→ (Loop (print (-> (read) (eval))))
→ (Loop (print (eval (read))))
```

The function `read` reads a data structure from input. And since Lisp—and Clojure—code *is* data structures, `read` reads code. The function `eval` evaluates that structure. If it’s a list, surrounded by parentheses, it gets executed. The `print` function (renamed `println`) prints the result with a newline.

Then the macro `looop`, which we defined ourselves, unconditionally repeats the rest of the expression. It finally expands into:

```clojure
(loop [] (print (eval (read))) (recur))
```

The threading macro `->>` (used in `(->> (read) (eval) (println) (while true))`) behaves like `->`, but inserts the expression as the last argument instead of the second. So:

```clojure
(-> (read) (eval) (println) (while true))
```

...would expand to:

```clojure
(while (println (eval (read))) true)
```

...instead of:

```clojure
(while true (println (eval (read))))
```

We probably need some error handlign. You do make misstake in a repl. Eval will throw exception, and exit the loop  

```clojure
(defmacro Loop [& body] `(loop [] ~@body (recur)))
(let [print println]
  (-> (read) (eval) (print) (Loop)))
```
If we catch and return the exceptions to print, we will see the exception and can gracefully continue 
```clojure
(defmacro Loop [& body] `(loop [] ~@body (recur)))
(defmacro Eval [& body] `(try (eval ~@body) (catch Exception e# e#)))
(let [print println]
  (-> (read) (Eval) (print) (Loop)))
```

### Instructions

Save the source code into a file called `repl.clj`. Download `clojure.jar` from clojure.org, then let the hacking begin:

```bash
> java -jar clojure.jar repl.clj
(+ 4 5)
9
(defmacro Loop [& body] `(loop [] ~@body (recur)))
#'user/Loop
(defmacro Eval [& body] `(try (eval ~@body) (catch Exception e# e#)))
#'user/Eval
(let [print println]
  (-> (read) (Eval) (print) (Loop)))
(+ 4 3)
7
(defmacro Loop [& body] `(loop [] ~@body (recur)))
#'user/Loop
```

