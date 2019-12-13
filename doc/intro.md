# Introduction to core.btree-vector

My original motivation for writing this library was to have an
implementation of an immutable vector that supports O(log N) time
concatenation operations, worst case, which is significantly faster
than linear time behavior you get if you evaluate `(into v1 v2)` for
two Clojure vectors.

These new vectors should still support all existing operations on
Clojure vectors as efficienctly as Clojure vectors do, or at worst be
a small constant times slower.

The research papers about the RRB Tree data structure claim that data
structure provides that, but everything I have read so far on RRB
Trees leave out some details of implementation that are not clear to
me how they should be performed, and [all of the RRB Tree
implementations I have found so
far](https://github.com/clojure/core.rrb-vector/blob/master/doc/rrb-tree-notes.md)
have bugs in their operations.  Without clearly stated invariants, it
can be difficult to approach such an existing implementation and see
what changes ought to be made that will be correct, or indeed if that
is even possible while preserving the performance guarantees.

So while this article may be extended to give details on how all
operations are implemented efficiently, the focus here is on ensuring
that creating a sub-vector of an existing vector, and creating a
concatenation of two existing vectors, can be done in worst case O(log
N) time.


## Using invariants to develop programs

One good way to reason carefully about all possible behaviors of a
program is by stating, and mathematically proving, conditions that
must be true at all times during the program's execution.  Such a
condition is called an _invariant_, meaning "unchanging", in the sense
that the condition remains true always.

When implementing a data structure, such invariants are often stated
based upon the "shape" and contents of the data structure.  Then, show
how every operation on that data structure "maintains the invariant",
i.e. if the initial data structure given as input(s) to the operations
satisfy the invariant, then the final data structure does, too.

Aside: There are likely better introductions to invariants for simple
data structures to learn about this topic for the first time, but one
source I found after a brief Internet search is [Lecture Notes on Data
Structure
Invariants](https://www.cs.cmu.edu/~fp/courses/15122-f10/lectures/12-dsinvs.pdf)
from a course given at Carnegie Mellon University.


## Rooted ordered trees

If you are already familiar with trees, as used in many data
structures, feel free to skip this section.

A rooted ordered tree is a tree where there is a distinguished _root
node_.  The root node has a (possibly empty) ordered sequence of child
nodes, where all of those child nodes are distinct from the root node.
If there are any nodes besides the root, each of them also has a
possibly empty ordered sequence of child nodes.  There are no cycles
in the parent-child relationships, i.e. no node can be its own parent,
grandparent, or ancestor of any kind.

The figure below gives one example of a rooted ordered tree.  Rooted
trees are typically drawn with edges (i.e. lines) indicating
parent-child relationships.  In this document, all such trees will be
drawn with the root node at the top, with child nodes always appearing
lower than their parents.

<img src="images/rooted-ordered-tree.png" alt="Rooted ordered tree" width="600" align="middle">

Among child nodes that have the same parent, their relative order will
be indicated by their left-to-right ordering in the drawing.

A node with no children is called a _leaf_ node.  A node with at least
one child is called an _internal_ node.  There is a unique path from
the root to every other node in the tree.

We define the _depth_ of the root node to be 0.  All of the root's
children have depth 1.  The depth of a non-root node is equal to the
depth of its parent, plus 1.

<img src="images/rooted-ordered-tree-showing-depths.png" alt="Rooted ordered tree showing depths" width="600" align="middle">

Most kinds of trees described later in this document will have
additional restrictions on their structure.


## B-trees


### Invariants for B-trees

[B-trees](https://en.wikipedia.org/wiki/B-tree) and [B+
trees](https://en.wikipedia.org/wiki/B%2B_tree) were developed to
implement data structures like a Clojure sorted map.  That is, they
can represent a set of key/value pairs where all keys are distinct,
and can be sorted by a [total
order](https://en.wikipedia.org/wiki/Total_order), e.g. integers by
the order `<=`, or strings sorted lexicographically.  The values can
be arbitrary, i.e. they need not be distinct, and need not have any
sorted order relative to each other.  B-trees support efficient lookup
of the value associated with a given key, and many other operations.

We will begin our discussion of B-trees this way, and then later
specialize them for use in representing vectors, where the keys are
restricted to be integer indices in the range 0 up to n-1, where n is
the number of elements in the vector.

Aside: I do not know all differences between B-trees and B+ trees.  I
will refer to what is being described here as B-trees, but mentally
substitute "B+ trees" if that is more precise.  The values in the
trees described here are stored only in the leaf nodes, and the keys
are stored in non-leaf nodes, or optimized away in some special cases
where they need not be stored at all.

The following conditions for a B-tree's structure apply for any
maximum branch factor B that is an integer value at least 3.

b=ceiling(B/2) is B divided by 2, then rounded up to the next integer
if the result is a fraction.  Since B >= 3, b is always at least 2.

The invariants that a B-tree must satisfy are:

(I1) It is a rooted ordered tree.

(I2) All values are stored in leaf nodes.  The key associated with
     each value is also stored with the value.

(I3) All leaf nodes are at the same depth as each other.

(I4) The order of all key/value pairs in a tree is the same as the
     order that the leaf nodes are visited in a [preorder depth-first
     traversal](https://en.wikipedia.org/wiki/Depth-first_search),
     where all children of a node are visited in the same order that
     they are children of their parent.  The order of keys must be
     consistent with the total order specified for the keys.

(I5) All nodes have at most B children.

(I6) All non-root internal nodes have at least b children.  The root
     node has at least 2 children, unless there is only one value in
     the entire tree, in which case the root has 1 child.

(I7) All internal nodes with A >= 2 children contain A-1 keys.  If we
     number the children, in order, from c[0] up to c[A-1], and the
     keys from k[1] up to k[A-1], then for all j in the range [1,
     A-1], key k[j] must be strictly greater than all keys in the
     subtree of child c[j-1], and less than or equal to all keys in
     the subtree of child c[j].

For now we will leave the value of B arbitrary, and only pick a
particular value for use in examples, and for the core.btree-vector
implementation.

Aside: A special case not mentioned very often is a B-tree with no
key/value pairs, i.e. an empty set of keys.  The exact representation
of an empty B-tree is not a big deal -- e.g.  storing a count field of
0 in the object is a good way.  There might be a root node with no
children in the implementation, but we will not usually consider that
case explicitly everywhere in this document.  If we did, we would
frequently be mentioning that special case, and I would prefer not to
keep mentioning it.


### Definitions of some terms related to B-trees

These definitions need not be understood immediately.  Feel free to
skip over them and come back as and when you need them.  I wanted to
keep them together in this document.

Examples of all of these definitions are in the next section.

For trees satisfying (I3), we can define the _height_ of a node as the
number of edges in any path from the node down to a leaf node.  For
any node in a tree satisfying (I3), there is only one such number.
The height of all leaf nodes is 0, and for any internal node, its
height is equal to the height of any of its children (which all have
the same height as each other), plus 1.

For any rooted ordered tree T, define _dfs(T)_ to be the ordered list
of nodes of T, in the order they are traversed in a [preorder depth
first traversal](https://en.wikipedia.org/wiki/Depth-first_search), of
T.  Since T is ordered, there is only one possible order for the nodes
in dfs(T).

Define _depth(T, d)_ to be the ordered list of nodes obtained by
starting with dfs(T), and removing all nodes that have a depth not
equal to d, keeping the relative order of the nodes with depth d.

For any node N with depth d in a rooted ordered tree T, define N's
_right neighbor_ to be the first node after N in the list depth(T, d).
If there is no node after N in that list, N has no right neighbor.
Similarly the _left neighbor_ of N is the last node before N in the
list depth(T, d), if there is such a node.

Define the _left fringe_ of a rooted ordered tree as the following set
of nodes.

+ The root node is in the left fringe.
+ If any node is in the left fringe and has children, then its
  left-most (i.e. first) child is also in the left fringe.

The _right fringe_ is defined the same way, replacing all occurrences
of "left" with "right", and "first" with "last".


### B-tree examples, and consequences of the invariants

These example B-trees have max branching B=5, and minimum branch
factor b=3.  Each represents a subset (or all of) this collection of
key/value pairs:

| Key | Value     |
| --- | --------- |
|   2 | "Ellis"   |
|   3 | "John"    |
|   5 | "Hank"    |
|   7 | "Dagny"   |
|  11 | "Hugh"    |
|  13 | "Midas"   |
|  17 | "Eddie"   |
|  19 | "Ragnar"  |
|  23 | "Quentin" |

<img src="images/b-tree-kvs-order-5-few-elements.png" alt="Small B-trees with order 5" width="800" align="middle">

For B=5 and b=3, those are the _only_ tree structures that are legal
for the given number of elements.  In general, if there are less than
2b vector elements, all elements must be direct children of the root,
at depth 1.  There are no other tree structures that satisfy all of
the invariants for so few elements.

There are at least two tree structures that satisfy the invariants for
2b+1 or more elements.  As the number of elements grows, the number of
tree structures satisfying the invariants grows exponentially.  It is
reasonable for a B-tree implementation to allow any tree structure to
be used, as long as it satisfies all of the invariants.

The three B-trees below are all legal for 9 elements, with B=5 and
b=3.

<img src="images/b-tree-kvs-order-5-with-9-elements.png" alt="B-trees with order 5 and 9 elements" width="800" align="middle">

The height of a B-tree is O(log N), where the base of the logarithm is
b.  It can be slightly less than that if enough nodes have branching
factors near B.  Chapter 3 "RRB-Trees" of Jean Niklas L'orange's
thesis contains detailed proofs of this and many other results.

* Jean Niklas L'orange, "Improving RRB-Tree Performance through
  Transience", Master Thesis, 2014,
  [[PDF]](https://hypirion.com/thesis.pdf)

The tree below demonstrates several definitions from the previous
section.

<img src="images/rooted-ordered-tree-fringes.png" alt="Left and right fringes of a rooted ordered tree" width="600" align="middle">

+ dfs(T) = [A, B, C, D, E, F, G, H, I, J, K, L]
+ depth(T, 0) = [A]
+ depth(T, 1) = [B, F, H, L]
+ depth(T, 2) = [C, D, E, G, I, K]
+ depth(T, 3) = [J]
+ depth(T, 4) = [] (i.e. the empty list -- there are no nodes at depth
  4 in this tree)
+ The left neighbor of H is F, and the right neighbor of H is L.
+ B has no left neighbor.  The right neighbor of B is F.


### Efficient B-tree split operations

By a split operation, I mean one where you are given a B-tree T and a
key K that exists in T, and the goal is to return a new B-tree T2 that
contains only the key/value pairs of T where the key is K or larger,
in the total order on keys.

By efficient, I mean it runs in time equal to some constant factor
times the height of the tree T, and that the returned data structure
satisfies all of the B-tree invariants.

Note: The corresponding operation that returns a tree with all keys K
or smaller is just the "mirror image" of the split operation above.
If we can make one of them work efficiently, the other will work, too.
And if both of those operations work efficiently, one followed by the
other can be used to achieve the effect of a "sub range" operation,
which given a tree T and a minimum key K1 and a maximum key K2,
returns a tree T2, that contains only those keys K of T such that K1
<= K <= K2.

It turns out to be a fruitful approach to implement a split operation
by first simply removing all leaves for key/value pairs with keys less
than K, and then removing all internal nodes that have no remaining
children after those leaves are removed, as shown in this example:

<img src="images/b-tree-kvs-split-example-1.png" alt="Example 1 of first step of B-tree split operation" width="800" align="middle">

The problem with this intermediate step is that the resulting tree T'
can violate some of the B-tree invariants, in particular invariant
(I6) that restricts the minimum number of children a node must have.

So, we then "fix up" the tree, such that the final tree satisfies all
invariants.  The trick is to do so in worst case O(log N) time, and
prove that the result will always satisfy all of the invariants,
regardless of what tree T and key K were given for the split
operation.

First, observe that in tree T', all internal nodes have the same
number of children they had in T, except perhaps for some of the nodes
in the left fringe of T'.  If we can find a way to correct those
nodes, and preferably a small number of others "near" them, that would
be good.

Let us start with the parent node F of the leaf node in the left
fringe of T', and try "fixing up" its number of children, then work
our way upwards along the left fringe towards the root.

First, we introduce a few more definitions that will help us be
precise.

There are several cases to consider:

Case (split1): F has a "neighbor to the right".  in the tree there is a node N with the same depth as F.  In a depth first traversal of T', F would be the next node after N that has 


### Efficient B-tree concatenate operations


## Invariants for Clojure's PersistentVector implementation

Clojure's `PersistentVector` implementation satisfies some invariants
that are similar to the ones described in the previous section.

Every vector element is stored either as a leaf in a tree, or in an
array of up to 32 elements called the "tail".  The tail array always
contains at least one element if the vector is not empty, which
guarantees that the `peek` operation, which returns the last element
of the vector, always takes O(1) time.

Similar to B-trees, all vector elements are at the same depth as each
other from the root node.

Unlike B-trees, all array nodes always contain exactly B children
(which are vector elements).  "Most" tree nodes are restricted to
contain exactly B children, with only a few nodes described below
allowed to be an exception to that condition.


### Why store relative index values instead of absolute ones?

TBD: Not sure where is best to introduce this text.

Typically when B-trees are used, removing one key/value pair leaves
all of the remaining key/value pairs unchanged.  With the
core.btree-vector, if start with a vector with 10 elements, where the
keys range from 0 through 9, and take a sub-vector starting from index
3 up through 9, inclusive, the returned vector should have 7 elements
with keys 0 through 6.

Similarly, when concatenating two vectors, the keys of the first
vector remain the same, but the keys of the second vector are all
larger in the returned vector than they were in the second input
vector (unless the first vector was empty).

Thus it is important that if we do not want to require all sub-vector
and concatenation operations to take linear time, the key values
should not be explicitly stored as absolute values.  Instead they are
stored as relative values in each sub-tree, relative to the number of
elements that exist in earlier sub-trees.


# Details to double check later

[TBD: Where to introduce this?]  We will call the parent nodes of leaf
nodes _array nodes_ (TBD: maybe a phrase to motivate this choice
here).  If the tree has height 1, then the only non-leaf node, the
root node, is an array node.

TBD: Compare this terminology with that used in Clojure source code
for PersistentVector and Vector.

TBD: Try to minimize the number of differences in terminology used for
different tree data structures that this library deals with, and make
it clear up front and on first definition what the terminology
differences are as used here, versus in some popular reference on tree
terminology, perhaps the Wikipedia article on tree data structures, or
CLRS.  If I give Niklas L'orange's articles on PersistentVector as a
reference, reread it for terminology and any differences to what is
used here.

TBD: Is the invariant that all leaf nodes are at the same depth as
each other important?  Why?  What would go wrong if we tried to allow
trees with leaf nodes at different depths?
