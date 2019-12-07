# Introduction to core.btree-vector


## Using invariants to develop programs

One good way to reason carefully about operations on a data structure
is to explicitly state conditions that all correct examples of the
data structure satisfy, often called an invariant, meaning
"unchanging", in the sense that the condition remains true always.

Then, show how every operation on that data structure "maintains the
invariant", i.e. if the initial data structure given as input(s) to
the operations satisfy the invariant, then the final data structure
does, too.

TBD: There are likely better introductions to invariants for simple
data structures to learn about this topic for the first time, but one
source I found after a brief Internet search is [Lecture Notes on Data
Structure Invariants]
https://www.cs.cmu.edu/~fp/courses/15122-f10/lectures/12-dsinvs.pdf)
from a course given at Carnegie Mellon University.


## Rooted ordered trees

If you are already familiar with trees, as used in many data
structures, feel free to skip this section.

A rooted ordered tree is a tree where there is a distinguished _root
node_.  The root node has a (possibly empty) ordered sequence of child
nodes, all of which are distinct from the root node .  Every other
node (if there are any others) also has a possibly empty ordered
sequence of child nodes.  There are no cycles in the parent-child
relationships, i.e. no node can be its own parent, grandparent, or
ancestor of any kind.

![Rooted ordered tree](images/rooted-ordered-tree.png)

Rooted trees are often drawn with edges (i.e. lines) indicating
parent-child relationships.  In this document, all such trees will be
drawn with the root node at the top, with child nodes always appearing
lower than their parents.  The ordering of child nodes relative to
each other is indicated by their left-to-right ordering in the
drawing, with earlier children to the left of later children.

TBD: Show a figure of a rooted ordered tree with leaves at different
depths.  Mention that most kinds of trees described later will have
additional restrictions on them, including that all leaves must be at
the same depth from the root.

A node with no children is called a leaf node.  There is a unique path
from the root to every other node in the tree.

The depth of the root node is defined to be 0.  All of the root's
children have depth 1, and in general


## Invariants for B-trees

B trees and B+ trees were originally developed to implement data
structures with arbitrary key/value pairs, where the keys can be
sorted by a [total order](https://en.wikipedia.org/wiki/Total_order).

As implemented by the core.btree-vector library, all of the keys are
integer indices, ranging from 0 up to n-1, where n is the number of
elements in the vector.  This is a fairly special case for B trees,
and leads to some slight simplifications in the data structure.

Typically when B trees are used, removing one key/value pair leaves
all of the remaining key/value pairs unchanged.  With the
core.btree-vector, if we want to take a vector with 10 elements with
keys 0 through 9, and take a sub-vector starting from index 3 up
through 9, inclusive, the returned vector should have 7 elements with
keys 0 through 6.

Similarly, when concatenating two vectors, the keys of the first
vector remain the same, but the keys of the second vector are all
larger in the returned vector than they were in the second input
vector (unless the first vector was empty).

Thus it is important that if we do not want to require all sub-vector
and concatenation operations to take linear time, the key values
should not be explicitly stored as absolute values.  Instead they are
stored as relative values in each sub-tree, relative to the number of
elements that exist in earlier subtrees.

I do not know all differences between
[B-trees](https://en.wikipedia.org/wiki/B-tree) and [B+
trees](https://en.wikipedia.org/wiki/B%2B_tree), but I suspect that
what I will describe here is slightly closer to B+ trees, in that the
elements of the vector are stored only in the leaves, and the keys are
stored in non-leaf tree nodes.

Choose an "order", also called a branching factor, B, at least 3.
Every node in the tree will have at most B children.

A special case not mentioned below is an empty vector.  The exact
representation of an empty vector is not a very big deal -- e.g.
storing a count field of 0 in the object representing the vector is a
good way.  There might be a root node with no children in the
implementation, but we will not consider that case when describing the
invariants that trees representing non-empty vectors must satisfy.  If
we did, we would frequently be mentioning that special case, and I
would prefer not to keep mentioning it.

b=ceiling(B/2) is B divided by 2, then rounded up to the next integer
if the result is a fraction.  Since B >= 3, b is always at least 2.
With one exception, every node in the tree has at least b children.

The exception is:

+ The root node is allowed to have less than b children.  If there is
  only 1 element in the vector, the root has only 1 child (which is
  the vector element), otherwise the root always has at least 2
  children.

All vector elements are leaves in the tree, and all are at the same
depth in the tree.  If there are less than B vector elements, they are
all direct children of the root, at depth 1.  Such a tree has height
1.

If there are more than B vector elements, they cannot all be direct
children of the root.  Instead they are all at the same depth from the
root, at least 2.

There is always a root node, which is at depth 0.  The root and all
other non-leaf nodes are called "internal nodes".  The parent nodes of
vector elements I will call "array nodes".  If the tree has height 1,
then the the only non-leaf node, the root node, is an array node.

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


## Invariants for Clojure's PersistentVector implementation

Clojure's `PersistentVector` implementation satisfies some invariants
that are similar to the ones described in the previus section.

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

Define the "right fringe" of a tree as the following set of nodes.

+ The root node is in the right fringe.
+ If the root node has any children, then the right-most (i.e. last)
  child is also in the right fringe.

If the root node has any children, then 
