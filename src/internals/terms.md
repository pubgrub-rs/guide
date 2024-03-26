# Terms

Dependency solving is a complex task where intuition may not be correct or scale to complex situations.
We therefore anchor this task in sound mathematics and logic primitive.
In turn, this demands precise notation and translations between
the primitives of an application layer (`Cargo.toml`, `requirements.txt`, ...)
and those of the underlying mathematics (CDCL, CDNL, ...).
We introduce some of that notation here.

## Definition

For any given package, we describe a set of possible versions as a **range**.
For example, a range may correspond to a single version, the set of versions higher than 2 and lower than 5,
the set containing every possible version or even the set containing no version at all.
And once we have ranges, we can build **terms**, defined as either positive or negative ranges.

```rust
pub enum Term<V> {
    Positive(Range<V>),
    Negative(Range<V>),
}
```

A term is a [literal][literal] evaluated either true or false depending on the evaluation context.
A positive term is evaluated true if and only if
a version contained in its range was selected.
A negative term is evaluated true if and only if
a version not contained in its range was selected,
or no version was selected.

> **Remark on negative terms:**
>
> A negative term is different than the positive one with its complement range.
> Indeed, saying "a version was picked in the complement of this range",
> is different than saying "**no version was picked or** a version was picked in the complement of this range".

In this guide, for any given range \\(r\\),
we will note \\([r]\\) its associated positive term,
and \\(\neg [r]\\) its associated negative term.
And for any term \\(T\\), whether positive or negative range, we will note \\(\overline{T}\\) the negation of that term,
transforming a positive range into an negative one and vice versa.
Therefore we have the following notation rules,

\\[\begin{eqnarray}
\overline{[r]} &=& \neg [r], \nonumber \\\\
\overline{\neg [r]} &=& [r]. \nonumber \\\\
\end{eqnarray}\\]

> **Remark on notation:**
>
> To keep the notation precise, we only use the \\(\neg\\) symbol in front of ranges, such as \\(\neg[r]\\).
> So we know when seeing it that we are talking about a negative term.
> In contrast, we use the overline symbol generically, so we can't make the assumption
> that the presence, or absence, of overline describes a positive or negative term.
> So for a generic term \\(T\\), we will not write \\(\neg T\\), but \\(\overline{T}\\),
> and for a range \\(r\\), we will not write \\(\neg \neg [r]\\), but \\(\overline{\neg [r]}\\)
> which is also directly \\([r]\\).

[literal]: https://en.wikipedia.org/wiki/Literal_(mathematical_logic)


## Special terms

Provided that we have defined an empty range of versions \\(\varnothing\\),
there are two terms with special behavior which are \\([\varnothing]\\) and \\(\neg [\varnothing]\\).
By definition, \\([\varnothing]\\) is evaluated true if and only if
a version contained in the empty range is selected.
This is impossible and as such the term \\([\varnothing]\\) is always evaluated false.
And by negation, the term \\(\neg [\varnothing]\\) is always evaluated true.


## Intersection of terms

We define the "intersection" of two terms
as the conjunction of those two terms (a logical AND).
Therefore, if any of the terms is positive, the intersection also is a positive term.
Given any two ranges of versions \\(r_1\\) and \\(r_2\\), the intersection of terms
based on those ranges is defined as follows,

\\[\begin{eqnarray}
[r_1] \land [r_2] &=& [r_1 \cap r_2],                 \nonumber \\\\
[r_1] \land \neg [r_2] &=& [r_1 \cap complement(r_2)], \nonumber \\\\
\neg [r_1] \land \neg [r_2] &=& \neg [r_1 \cup r_2].  \nonumber \\\\
\end{eqnarray}\\]

Where \\(\cap\\) and \\(\cup\\) are the notations for set intersections and unions of version ranges.

For any two terms \\(T_1\\) and \\(T_2\\), their union and intersection are related by De Morgan's rule:

\\[ \overline{T_1 \lor T_2} = \overline{T_1} \land \overline{T_1}. \\]


## Relation between terms

We say that a term \\(T_1\\) is satisfied by another term \\(T_2\\)
if \\(T_2\\) implies \\(T_1\\), i.e.
when \\(T_2\\) is evaluated true then \\(T_1\\) must also be true.
This is equivalent to saying that \\(T_2\\) is a subset of \\(T_1\\),
which is verified if \\( T_2 \land T_1 = T_2 \\).

> **Note on comparability of terms:**
>
> Checking if a term is satisfied by another term is accomplished
> in the code by verifying if the intersection of the two terms
> equals the second term.
> It is thus very important that terms have unique representations,
> and by consequence also that **ranges have a unique representation**.
>
> In the provided `Range` type, ranges are implemented
> as an ordered vec of non-intersecting semi-open intervals.
> It is thus important that they are always reduced to their
> canonical form.
> For example, the range `2 <= v < 2` is actually empty
> and thus should not be represented by `vec![(2, Some(2))]`
> but by the empty `vec![]`.
> **This requires special care when implementing range intersection**.
