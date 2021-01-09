# Immutable vectors of values with restricted types

Clojure and clj means Clojure that runs on the Java VM.  ClojureScript
and cljs means Clojure that runs on a JavaScript runtime.

Clojure has a function `vector` that returns an immutable vector where
every element is an arbitrary object, and every element can be a
different type of value than the others.  If you use it to hold double
precision floating point numbers, for example, they will be references
to Java objects with type `Double`, sometimes called "boxed" numbers.
Each such number has all of the memory overhead of a full Java object,
which on many systems makes them consume 24 bytes each, rather than
only the 8 bytes required for the double value itself.

Clojure also has a function `vector-of` that returns an immutable
vector where very element is restricted to be the same type of Java
primitive value.  `(vector-of :double 1.0 2.0 3.0)` returns an
immutable vector where every element is restricted to be a double
precision floating point number.  Such vectors consume significantly
less memory than those returned by `vector`, because every double
value is stored as a primitive 8-byte double number, with no object
overhead.


# ClojureScript

ClojureScript has a function `vector`, but no `vector-of`.

I have recently learned that at least some JavaScript runtimes have
the capability of creating arrays of primitive/unboxed numbers.  Here
are some references to more details on how to create them using
JavaScript.

+ https://developer.mozilla.org/en-US/docs/Web/JavaScript/Typed_arrays
+ https://liveoverflow.com/revisiting-javascriptcore-internals-boxed-vs-unboxed-browser-0x06/

It seems possible that these capabilities might be used to implement
`vector-of` in ClojureScript, with similar lower memory requirements,
and perhaps also performance improvements when manipulating such
vectors.


# Supporting vectors with elements that are not Java primitive types

For example, an immutable vector of individual bits might be useful in
some circumstances, but using an entire Java byte that can hold 8 bits
to hold only 1 bit is wasteful of memory.

It would be nice to provide an "array manager" interface that, if
implemented appropriately, could be used to handle the contents of all
array nodes in a B-tree.

This idea has been discussed earlier in comments on Clojure ticket
CLJ-1416 here: https://clojure.atlassian.net/browse/CLJ-1416
