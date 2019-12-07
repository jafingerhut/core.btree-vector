# core.btree-vector

An implementation of Clojure vectors that supports efficient versions
of all of the operations that Clojure's built-in vectors support, but
also O(log N) worst case run time concatenation of any two vectors.

It was inspired by, and copies some of the implementation code from,
the [core.rrb-vector](https://github.com/clojure/core.rrb-vector)
library.  However, it uses
[B-Trees](https://en.wikipedia.org/wiki/B%2B_tree) as the data
structure to represent vectors, rather than the RRB Tree data
structure that the core.rrb-vector library uses.  This is because I
(Andy Fingerhut) know how to implement the concatenation operation in
worst case O(log N) time with B-Trees, but haven't yet seen a precise
enough description of how to do this for RRB Trees that I trust it is
correct.  Every proposed implementation of RRB Trees that I have
tested so far has bugs -- this might be unrelated to my lack of
understanding to how to implement RRB Trees efficiently, but I am
beginning to wonder if there is a way to do so.

Why would anyone want to use this library?  The two primary answers
are:

+ You want faster concatenation of vectors, which core.btree-vector's
  `catvec` function provides for both Clojure and ClojureScript.
+ You use vectors of Java primitive types like long, double, etc.,
  returned by Clojure's `vector-of` function, e.g. to reduce memory
  usage to about 1/3 of the memory required by vectors of arbitrary
  objects, and
  + You want the speed enabled by using the transient versions of such
    vectors.  Clojure does not implement transients for primitive
    vectors created via `vector-of` -- core.btree-vector does.

Vectors are one of the most commonly used data structures within
Clojure.  Likely you already know that creating a vector equal to `v`
plus a new element `e` appended to the end using the expression `(conj
v e)` has a run time that is "effectively constant", i.e. it takes
O(log N) time in the size N of `v`, where the base of the logarithm is
32, so it is a constant at most 4 for all vector sizes up to a
million, and at most 7 for all vector sizes that Clojure supports.

The fastest way to concatenate two vectors `v1` and `v2` into a single
new vector is using an expression like `(into v1 v2)`.  This is
implemented by repeatedly appending a single element from the second
vector to the first, so it takes linear time in the size of `v2`
(multiplied by the effectively constant time mentioned above).

Aside: There might be another expression that has a better _constant
factor_ for its run time than `(into v1 v2)` does, and is thus faster.
However, any other such expression will still take at least linear
time in the size of the second vector.

The core.btree-vector library uses a tree structure similar to the one
that Clojure uses internally for vectors, but generalizes it in such a
way that producing a new tree that represents the concatenation of two
input vectors using the `catvec` function can be done in O(log N)
time, where N is the size of the result.

You can give `catvec` vectors created in all of the ways you already
normally do, and while it will return a new type of object, this new
type behaves in all of the ways you expect a Clojure vector to behave.
This new type of vector is indistinguishable from a normal Clojure
vector unless you examine the value of `(type v)` or `(class v)`.  In
particular, `(vector? v)` is true for this new type, you can call all
of the usual sequence-based functions on it to examine or process its
elements, you can call `conj` on it, `nth`, etc.

Thus if you have a program where frequently concatenating large
vectors to produce new vectors is useful, core.btree-vector may help
you write a much faster program in a more natural way.

B-Trees build upon Clojure's internal `PersistentVector` class used to
implement its built in vectors, adding logarithmic time concatenation
and slicing (i.e. create sub-vectors from input vectors).
ClojureScript is supported with the same API, except for the absence
of the `vector-of` function.

The main functions provided are `clojure.core.btree-vector/catvec`,
performing vector concatenation, and
`clojure.core.btree-vector/subvec`, which produces a new vector
containing the appropriate subrange of the input vector (in contrast
to `clojure.core/subvec`, which returns a view on the input vector).

TBD: Double check the paragraph below for details like precise class
names, after implementation is done.

Like Clojure vectors, core.btree-vector vectors can store arbitrary
values, or using `vector-of` you can create vectors restricted to one
primitive type, e.g. long, double, etc.  The core.btree-vector
implementation provides seamless interoperability with the built in
Clojure vectors of class `clojure.lang.PersistentVector`,
`clojure.core.Vec` (vectors of primitive values) and
`clojure.lang.APersistentVector$SubVector` instances:
`clojure.core.btree-vector/catvec` and
`clojure.core.btree-vector/subvec` convert their inputs to
`clojure.core.btree_vector.btree.Vector` instances whenever necessary
(this is a very fast constant time operation for PersistentVector and
primitive vectors; for SubVector it is O(log N), where N is the size
of the underlying vector).

`clojure.core.btree-vector` also provides its own versions of `vector`,
`vector-of`, and `vec` that always produce
`clojure.core.btree_vector.btree.Vector` instances.  Note that
`vector-of` accepts `:object` as one of the possible type arguments,
in addition to keywords naming primitive types.


## Usage

None yet.  Will write it up when the implementation is working.  It
should be very similar to core.rrb-vector except for the namespace
names.


## License

Copyright © 2012-2019 Michał Marczyk, Andy Fingerhut, Rich Hickey and contributors

This program and the accompanying materials are made available under the
terms of the Eclipse Public License 2.0 which is available at
http://www.eclipse.org/legal/epl-2.0.

This Source Code may also be made available under the following Secondary
Licenses when the conditions for such availability set forth in the Eclipse
Public License, v. 2.0 are satisfied: GNU General Public License as published by
the Free Software Foundation, either version 2 of the License, or (at your
option) any later version, with the GNU Classpath Exception which is available
at https://www.gnu.org/software/classpath/license.html.
