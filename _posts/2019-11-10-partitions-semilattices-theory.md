---
layout: post
title: "Partitions and Lattices, Oh my!"
category: theory
math: true
---
In "Regular Expression Derivatives Reexamined" [[1]][ref-one] the authors
reintroduce the work of Janusz Brzozowski and his elegant approach to directly
constructing a recognizer (i.e.  a lexer) from a regular expression or a vector
of regular expressions. The novel addition that [[1]][ref-one] brings to this
approach is the introduction of derivative classes which are a means of
partitioning the alphabet for a regular expression to reduce the number of
states needed in the deterministic finite automaton constructed by Brzozowski's
modified algorithm.

I implemented the deterministic finite automaton construction algorithm from
[[1]][ref-one] in Rust through a fairly straight forward (i.e. naive)
translation of the ideas in the paper into Rust code[^1]. The naive
implementation, though, had _terible_ performance and profiling it showed the
problem was my particularly naive implementation of derivative classes. My
search for a less naive implementation has led me down the rabbit hole of
partitions and lattice theory. This post is my attempt to climb back out of
that rabbit hole by recording what I have learned in a way that will
(hopefully) lead to an improved implementation.

This post is fairly dense in mathematical notation. This is in part because the
excuse ["this margin is to narrow"][fermat-conjecture] no longer cuts it in the
age of GitHub Pages and MathJax. If you are just looking for the bottom line it
is this:

> The derivative class approximation function from [[1]][ref-one] is equivalent
> to applying a [partition refinement] algorithm to some of the sets that make
> up the atomic terms of a given regular expression. The sets to which to apply
> the [partition refinement] algorithm are all of the sets from the atomic
> terms except those that are reachable from the right hand side of a
> concatenation term whose left hand side does not match the empty string.

[^1]: As of the time of the writing of this post the implementation is in the
    `luther-redfa` crate in the [rewrite-dfa-implementation] branch of my
    [luther] project, but it will likely be merged into the main branch once the
    problems in this post have been worked out.

[rewrite-dfa-implementation]: https://github.com/sbosnick/luther/tree/rewrite-redfa-implementation
[luther]: https://github.com/sbosnick/luther
[fermat-conjecture]: https://en.wikipedia.org/wiki/Fermat%27s_Last_Theorem

## Regular Expressions and Derivative Classes

The following definition of a regular expressions is from [[1]][ref-one]. A regular
expression over a finite alphabet $$ Σ $$ is given by the following grammar: 

$$
\begin{equation}
\begin{alignedat}{2}
   r,s ≔&\; ε &\quad\text{empty string} \\
   &|\; S &\quad\text{where}\, S ⊆ Σ \\
   &|\; r⋅s &\quad\text{concatenation} \\
   &|\; r^\ast &\quad\text{Kleene-closure} \\
   &|\; r + s &\quad\text{logical or (alternation)} \\
   &|\; r \& s &\quad\text{logical and} \\
   &|\; \neg r &\quad\text{complement}
\end{alignedat} \label{regex}
\end{equation}
$$

[[1]][ref-one] later uses this definition of a regular expression to define a function $$
C: \text{RE} → 2^{2^Σ} $$ (where $$ 2^X $$ is notation for the [power set] of
$$ X $$) that computes an approximation of the ideal derivative classes. $$ C
$$ is defined as follows:

$$
\begin{align}
    C(ε) &= \{ Σ \} \label{c_empty} \\
    C(S) &= \{S, Σ\smallsetminus S\} \label{c_subset} \\
    C(r⋅s) &= \begin{cases}
        C(r) &r \text{ is not nullable} \\
        C(r) ∧ C(s) &\text{ otherwise} \\
    \end{cases} \label{c_concat} \\
    C(r^\ast) &= C(r) \label{c_kleene} \\
    C(r+s) &= C(r) ∧ C(s) \label{c_alt} \\
    C(r\&s) &= C(r) ∧ C(s) \label{c_and} \\
    C(\neg r) &= C(r) \label{c_neg}
\end{align}
$$

where

$$
\begin{equation}
C(r) ∧ C(s) ≔ \{S_r ∩ S_s | S_r ∈ C(r), S_s ∈ C(s)\} \label{c_meet}
\end{equation}
$$

and "$$ r \text{ is not nullable} $$" is well defined in [[1]][ref-one] but
roughly means that $$ r $$ does not match the empty string. My naive
implementation of $$ C $$ is the source of the *terrible* performance I
encountered. A greater understanding of $$ C $$ led me to investigate the
algebraic properties of partitions of a set.

[power set]: https://en.wikipedia.org/wiki/Power_set

## Partitions and the Partition Semilattice
**Definition 1:** A partition of a finite set $$ Σ $$ is a set $$ Π $$ of
non-empty, pairwise disjoint subsets of $$ Σ $$ whose union is $$ Σ $$ (see
[[2]][ref-two]). That is, $$ Π ⊂ 2^Σ $$ is a partition of $$ Σ $$ if it
satisfies:
{: #defn-one}

[Definition 1]: #defn-one

$$
\begin{gather}
    ∅ \,\notin\, Π \label{part_no_null}\\
    A_i ∩ A_j = ∅ \quad ∀ A_i, A_j ∈ Π \text{ if } i \ne j \label{part_disjoint}\\
    \bigcup_{A ∈ Π} A = Σ \label{part_whole}
\end{gather}
\def\Part#1{\textbf{Part}(#1)}
$$

The members of $$ Π $$ are called its "blocks". 

**Definition 2:** Let $$ \Part{Σ} $$ be the set of all partitions of $$ Σ $$. 
Then 

$$ 
\begin{equation}
    \Part{Σ} ⊂ 2^{2^Σ}
\end{equation}
$$

**Definition 3:** We can define a [partial order] on $$ \Part{Σ} $$ by
refinement setting
{: #defn-three}

[Definition 3]: #defn-three

$$
\begin{align}
    Π_i ≤ Π_j \iff ∀ A ∈ Π_i \; ∃ B ∈ Π_j \text{ such that } A ⊆ B
\end{align}
$$

where $$ Π_i, Π_j ∈ \Part{Σ} $$.

[[2]][ref-two] and [[3]][ref-three] provide a proof that $$ \Part{Σ} $$ is a
[complete lattice] but we will rely on the more limited result that $$ \Part{Σ}
$$ with an appropriate binary operation is a bounded [semilattice].

**Definition 4:** We define a function $$ ∧ $$ (pronounced "meet") as
{: #defn-four}

$$
\begin{equation}
\begin{gathered}
    ∧ : \Part{Σ} × \Part{Σ} \mapsto \, 2^{2^Σ} \text{ with} \\
    Π_i ∧ Π_j = \{A_i ∩ A_j | A_i ∈ Π_i,\, A_j ∈ Π_j,\, A_i ∩ A_j \ne ∅\} \\
    ∀ Π_i, Π_j ∈ \Part{Σ}
\end{gathered}
\end{equation}
$$

[Definition 4]: #defn-four

**Lemma 1:** The co-domain of $$ ∧ $$ is $$ \Part{Σ} $$ and not the broader $$
2^{2^Σ} $$.
{: #lemma-one}

[Lemma 1]: #lemma-one

**Proof:** We show this by demonstrating the three parts of [Definition 1]. 

$$ Π_i ∧ Π_j $$ satisfies $$ \eqref{part_no_null} $$ because the definition of
$$ ∧ $$ specifically excludes $$ ∅ $$. 

$$ Π_i ∧ Π_j $$ satisfies $$ \eqref{part_disjoint} $$ as follows:

$$
\begin{align}
    ∀ A, B ∈ (Π_i ∧ Π_j) \quad A ∩ B &= (A_i ∩ A_j) ∩ (B_i ∩ B_j) \\
                                     &= (A_i ∩ B_i) ∩ (A_j ∩ B_j) \\
\end{align}
$$

which is $$ ∅ $$ if either $$ A_i \ne B_i $$ or $$ A_j \ne B_j $$ since both $$
Π_i $$ and $$ Π_j $$ satisfy $$ \eqref{part_disjoint} $$. If $$ A_i = B_i $$
and $$ A_j = B_j $$ then $$ (A_i ∩ A_j) = (B_i ∩ B_j) $$ which means $$ A = B
$$.

$$ Π_i ∧ Π_j $$ satisfies $$ \eqref{part_whole} $$ as follows:

$$
\begin{align}
    \bigcup_{A ∈ (Π_i ∧ Π_j)} A &= \bigcup_{\substack{A_i ∈ Π_i \\ A_j ∈ Π_j}} A_i ∩ A_j \\
                                &= (\bigcup_{A_i ∈ Π_i} A_i) ∩ 
                                    (\bigcup_{A_j ∈ Π_j} A_j) \\
                                &= Σ ∩ Σ \label{meet_covers_sigma_step}\\
                                &= Σ
\end{align}
$$

Step $$ \eqref{meet_covers_sigma_step} $$ follows because each of $$ Π_i $$ and
$$ Π_j $$ satisfy $$ \eqref{part_whole} $$. □

**Corollary:** $$ ∧ $$ is a [binary operation]. (The proof follows from the
definition of binary operation.)

**Lemma 2:** $$ Π_i ∧ Π_j $$ is a lower bound for $$ (Π_i, Π_j) $$ in $$
\langle \Part{Σ}, ≤ \rangle $$.
{: #lemma-two}

[Lemma 2]: #lemma-two

**Proof:** 

$$
\begin{eqnarray}
    Π_i ∧ Π_j   &=& \{A_i ∩ A_j| A_i ∈ Π_i, A_j ∈ Π_j, A_i ∩ A_j \ne ∅\} \\
                &\implies& (A_i ∩ A_j) ⊆ A_i \; ∀ A_i ∈ Π_i, A_j ∈ Π_j \\
                &\implies& ∀ A_i ∈ Π_i, A_j ∈ Π_j \; ∃ B ∈ Π_i \nonumber \\
                &&\quad\qquad\text{such that } (A_i ∩ A_j) ⊆ B \\
                &\implies& ∀ A ∈ (Π_i ∧ Π_j) \; ∃ B ∈ Π_i \text{ such that } A ⊆ B \\
                &\implies& (Π_i ∧ Π_j) ≤ Π_i
\end{eqnarray}
$$

The symmetry of $$ ∩ $$ means that the same result follows for $$ Π_j $$ as for
$$ Π_i $$. □

**Lemma 3:** $$ Π_i ∧ Π_j $$ is the greatest lower bound for $$ (Π_i, Π_j) $$
in $$ \langle \Part{Σ}, ≤ \rangle $$.
{: #lemma-three}

[Lemma 3]: #lemma-three

**Proof**: From [Lemma 2] we know that $$ Π_i ∧ Π_j $$ is a lower bound. The
proof that it is the _greatest_ lower bound proceed by contradiction. Assume
that $$ Π_i ∧ Π_j $$ is not the greatest lower bound for $$ (Π_i, Π_j) $$.
Then for some $$ A_i ∈ Π_i \text{ and } A_j ∈ Π_j $$:

$$
\begin{align}
    &∃ Γ ∈ \Part{Σ} \text{ such that } (Π_i ∧ Π_j) < Γ, Γ < Π_i, Γ < Π_j \\
    \implies& ∃ G ∈ Γ \text{ such that } (A_i ∩ A_j) ⊂ G, G ⊂ A_i, G ⊂ A_j \\
    \implies& ∃ g ∈ G \text{ such that } g \notin (A_i ∩ A_j), g ∈ A_i, g ∈ A_j
\end{align}
$$

which contradicts the definition of $$ ∩ $$. □

**Lemma 4:** $$ \{Σ\} $$ is the greatest element of $$ \langle \Part{Σ}, ≤
\rangle $$.
{: #lemma-four}

[Lemma 4]: #lemma-four

**Proof:** Let $$ Π $$ be an arbitrary element of $$ \Part{Σ} $$. Then by $$
\eqref{part_whole}$$

$$
\begin{align}
            &\bigcup_{A∈Π}A = Σ \\
    \implies& ∀ A ∈ Π, A ⊆ Σ \\
    \implies& ∀ A ∈ Π, ∃ B ∈ \{Σ\} \text{ such that } A ⊆ B \\
    \implies& Π ≤ \{Σ\} \label{greatest_step}
\end{align}
\def\PartSemi#1{\langle \Part{#1}, ∧, \{#1\} \rangle}
$$

$$ \eqref{greatest_step} $$ follows from [Definition 3].□

**Theorem 1:** $$ \PartSemi{Σ} $$ is a bounded meet 
[semilattice].

**Proof:** Lemmas [1][Lemma 1], [2][Lemma 2] and [3][Lemma 3] show that $$ ∧ $$
is a binary operator that gives the greatest lower bound of any two elements of
$$ \Part{Σ} $$. This shows that $$ \PartSemi{Σ} $$ is a meet-semilattice.
[Lemma 4] shows that $$ \{Σ\} $$ is the greatest element of $$ \Part{Σ} $$,
which show that $$ \PartSemi{Σ} $$ is bounded.□

**Corollary:** $$ \PartSemi{Σ} $$ is an idempotent 
commutative [monoid]. (The proof follows from the algebraic definition of a
[semilattice].)

[partial order]: https://en.wikipedia.org/wiki/Partially_ordered_set
[complete lattice]: https://en.wikipedia.org/wiki/Complete_lattice
[semilattice]: https://en.wikipedia.org/wiki/Semilattice
[binary operation]: https://en.wikipedia.org/wiki/Binary_operation
[monoid]: https://en.wikipedia.org/wiki/Monoid

## Derivative Classes and Partitions

The link between the partition semilattice described in the previous section
and the regular expressions and derivative classes defined earlier becomes
apparent when we look closer at the definition of the function $$ C $$ given in
[[1]][ref-one]. Although the definition of $$ C $$ in [[1]][ref-one] gives its
co-domain as $$ 2^{2^Σ} $$, a slightly modified function $$ C^* $$ has the
co-domain $$ \Part{Σ} $$. 

**Definiton 5:** We define $$ C^* $$ as being equal to $$ C $$ except for the
$$ S ⊆ Σ \text{ case in equation } \eqref{c_subset} $$. For that case we define

$$
\begin{equation}
    C^*(S) =  \begin{cases}
                \{Σ\}   & \text{if } S = ∅  \text{ or } S = Σ \\
                \{S, Σ\smallsetminus S \} & \text{if } S \ne ∅, S ⊂ Σ
            \end{cases}
            \label{cprime_subset}
\end{equation} 
$$

**Lemma 5:** The co-domain of $$ C^* \text{ is } \Part{Σ} $$.

**Proof:** The proof proceeds by induction.

The atomic cases, $$ \eqref{c_empty} $$ and $$ \eqref{cprime_subset} $$ hold
because $$ \{Σ\} ∈ \Part{Σ} $$ and $$ \{S, Σ\smallsetminus S \} ∈ \Part{Σ} $$.
(We had to introduce $$ \eqref{cprime_subset} $$ to avoid having $$ \{∅, Σ \}
$$ which is not an element of $$ \Part{Σ} $$ because of $$ \eqref{part_no_null}
$$.)

The "$$ r $$ is not nullable" case of $$ \eqref{c_concat} $$ and cases $$
\eqref{c_kleene} $$ and $$ \eqref{c_neg} $$ are defined in terms of $$ C^*(r)
$$ so they hold if $$ C^*(r) $$ does.The remaining cases are all defined in
terms of the definition of $$ ∧ $$ give in $$ \eqref{c_meet} $$, but this
definition is the same as the one give in [Definition 4] if $$ C^*(r) $$ and $$
C^*(s) $$ are elements of $$ \Part{Σ} $$. From [Lemma 1] we know that the
co-domain of $$ ∧ $$ is $$ \Part{Σ} \text{.} $$ Thus all the compound cases of
$$ C^* $$ also map to $$ \Part{Σ} $$. □

The compound cases of $$ C^* $$ are all defined in terms of either the identity
function on one of their terms (i.e. $$ \eqref{c_kleene} $$, and $$
\eqref{c_neg} $$ and the "$$ r $$ is not nullable" case of $$ \eqref{c_concat}
$$) or the application of $$ ∧ $$ to both of their terms (i.e. $$ \eqref{c_alt}
$$, $$ \eqref{c_and} $$, and the "otherwise" case of $$ \eqref{c_concat} $$).

If we were to "expand out" the application of $$ C^* $$ to a particular regular
expression we would end up with a series of application of $$ ∧ $$ to the
atomic terms of the regular expression. Since $$ \PartSemi{Σ} $$ is a
commutative [monoid] it won't matter what order we apply $$ ∧ $$. Also, since
$$ \{Σ\} $$ is the identity element of $$ \PartSemi{Σ} $$ the $$ C^*(ε) $$ case
(see $$ \eqref{c_empty} $$), the $$ C^*(∅) $$ case, and the $$ C^*(Σ) $$ case
(see $$ \eqref{cprime_subset} $$) can only have an effect on the final result
if they are the only terms. If there are any $$ C^*(S) $$ cases where $$ S $$
is a non-empty proper subset of $$ Σ $$ then the $$ C^*(ε), \, C^*(∅), \text{
and } C^*(Σ)$$ cases will effectively drop out. That is, 

$$
\begin{equation}
    C^*(r) = C^*(S_1) ∧ C^*(S_2) ∧ ... ∧ C^*(S_n) \text{ where } S_i \ne ∅, S_i ⊂ Σ
\end{equation}
$$

for appropriately chosen $$ S_1 ... S_n $$.

Call $$ C^*(ε) \, , C^*(∅), \text{ and } C^*(Σ) $$ the trivial cases. Then we
have the following theorem which gives a method of computing the non-trivial
cases of $$ C^* $$.

**Theorem 2:** $$ C^*(r) $$ for the non-trivial cases is equivalant to a
[partition refinement] algorithm for $$ Σ $$ for appropriately chosen $$ S_1
... S_n $$.

**Proof:** We can compute $$ C^*(r) $$ by folding the $$ C^*(S_i) $$ using $$ ∧
$$. 

Let

$$
\begin{equation}
    Π_i = C^*(S_1) ∧ ... ∧ C^*(S_i)
\end{equation}
$$

Then

$$
\begin{align}
    Π_{i+1} &= Π_i ∧ C^*(S_{i+1}) \label{fold_step_1} \\
            &= Π_i ∧ \{ S, Σ \smallsetminus S \} \\
            &=  \{ A_i ∩ S, A_i ∩ (Σ \smallsetminus S) \; | \, A_i ∈ Π_i \} 
                    \label{fold_step_3}
\end{align}
$$

where we interpret equation $$ \eqref{fold_step_3} $$ to exclude blocks where

$$
A_i ∩ S = ∅  \text{ or }  A_i ∩ (Σ \smallsetminus S) = ∅
$$ 

If we set

$$
\begin{equation}
    Π_0 = \{Σ\} \label{fold_start}
\end{equation}
$$

then equations $$ \eqref{fold_step_1} \text{ to } \eqref{fold_start} $$
describe an implementation of [partition refinement].□

Theorem 2 and the description above it are phrased in terms of "appropriately
chosen $$ S_1 ... S_n $$. The recursive definition of regular expressions given
at $$ \eqref{regex} $$ and the definition of $$ C^* \text{ given at }
\eqref{c_empty}, \, \eqref{cprime_subset}, \text{ and } \eqref{c_concat} \text{
to } \eqref{c_neg} $$ give us a method of choosing the appropriate $$ S_1 ...
S_n $$. The insight is that $$ \eqref{regex} $$ descirbes a tree with a maximum
arity of 2. The leaves of this tree are $$ ε $$ and $$ S ⊆ Σ $$. The other
parts of the defintion give rise to either binary or unary interior nodes of
the tree. 

The application of $$ C^* $$ to this tree results in visiting all nodes of the
tree except those reachable through the $$ s $$ branch of a concatenation where
the $$ r $$ branch is not nullable. $$ \text{(} \eqref{c_concat} \text{ where }
r \text{ is not nullable } $$ is the only part of the definition of the
compound cases of $$ C^* $$ that does not reference all of the terms from the
left hand side on the right hand side.)

A traversal of the tree for a given regular expression $$ t $$ that prunes the
$$ s $$ branch of any concatenation nodes with a non-nullable $$ r $$ branch
and then filters out everything besides the non-trivial leaves will give us the
appropriate collection of sets $$ S_i $$. Running the [partition refinement]
algorithm over these sets will give us the partition that is the result of $$
C^*(t) $$.

[partition refinement]: https://en.wikipedia.org/wiki/Partition_refinement#Data_structure

## Conclusion

Restating the derivative class approximation function $$ C $$ from
[[1]][ref-one] in as the function $$ C^* $$ allows us to show that it maps to
partitions of the alphabet $$ Σ $$ using the meet binary operation from the
partition semilattice over $$ Σ $$. This in turn gives us a method of
calculating $$ C^*(t) $$ for some regular expression $$ t $$ by running a
[partition refinement] algorithm over the non-trivial leaves of $$ t $$ once we
prune those that are reachable from the $$ s $$ branch of a concatenation whose
$$ r $$ branch is not nullable.

There are well known implementations of [partition refinement] that would
calculate each step of this algorithm in $$ O(|S_i|) $$ time, where $$ S_i
$$ is one of the non-trivial leaves of $$ t $$. Unfortunately such
implementations take $$ O(|Σ|) $$ space. If $$ Σ $$ is the set of all [Unicode
scalar values] (spelled `char` in Rust), then $$ O(|Σ|) $$ is more than a
million.

I am working on an implementation of [partition refinement] that is based on a
representation of the sets $$ S_i $$ as an ordered collection of ranges. This
implementation will (hopefully) use much less than $$ O(|Σ|) $$ space, though at
a cost of taking more then $$ O(|S_i|) $$ time.  If this is successful then it
will (again hopefully) move the performance bar of my deterministic finite
automaton construction algorithm from *terrible* to simply bad.

[Unicode scalar values]: https://unicode.org/glossary/#unicode_scalar_value

### References
[ref-one]: #ref-one
[ref-two]: #ref-two
[ref-three]: #ref-three

[1] S. Owens, J. Reppy, and A. Turon, "Regular expression derivatives reexamined", 
    *J. Funct. Program.*, vol. 19, no. 2, pp. 173-190, 2009. 
{: #ref-one}

[2] G. Grätzer, *General Lattice Theory*, 2nd ed., Boston: Birkhäuser Verlag, 1998,
    sec. IV.4.
{: #ref-two}

[3] C. Woo, "Partitions form a lattice", planetmath.org, March 3, 2013. [Online].
    Available: <https://planetmath.org/partitionsformalattice>. [Accessed: October 19,
    2019].
{: #ref-three}

### Notes
