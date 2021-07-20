# Versions in a continuous space

The current design of pubgrub exposes a `Version` trait demanding two properties, (1) that there exists a lowest version, and (2) that versions live in a discrete space where the successor of each version is known.
So versions are basically isomorph with N^n, where N is the set of natural numbers.

## The successor design

There is a good reason why we started with the successor design for the `Version` trait.
When building knowledge about the dependency system, pubgrub needs to compare sets of versions, and to perform common set operations such as intersection, union, inclusion, comparison for equality etc.
In particular, given two sets of versions S1 and S2, it needs to be able to answer if S1 is a subset of S2 (S1 ⊂ S2).
And we know that S1 ⊂ S2 if and only if S1 ∩ S2 == S1.
So checking for subsets can be done by checking for the equality between two sets of versions.
Therefore, **sets of versions need to have unique canonical representations to be comparable**.

We have the interesting property that we require `Version` to have a total order.
As a consequence, the most adequate way to represent sets of versions with a total order, is to use a sequence of non intersecting segments, such as `[0, 3] ∪ ]5, 9[ ∪ [42, +∞[`.

> Notation: we note segments with close or open brackets depending on if the value at the frontier is included or excluded of the interval.
> It is also common to use a parenthesis for open brackets.
> So `[0, 14[` is equivalent to `[0, 14)` in that other notation.

The previous set is for example composed of three segments,
- the closed segment [0, 3] containing versions 0, 1, 2 and 3,
- the open segment ]5, 9[ containing versions 6, 7 and 8,
- the semi-open segment [42, +∞[ containing all numbers above 42.

For the initial design, we did not want to have to deal with close or open brackets on both interval bounds.
Since we have a lowest version, the left bracket of segments must be closed to be able to contain that lowest version.
And since `Version` does not impose any upper bound, we need to use open brackets on the right side of segments.
So our previous set thus becomes: `[0, ?[ ∪ [?, 9[ ∪ [42, +∞[`.
But the question now is what do we use in place of the 3 in the first segment and in place of the 5 in the second segment.
This is the reason why we require the `bump()` method on the `Version` trait.
If we know the next version, we can replace 3 by bump(3) == 4, and 5 by bump(5) == 6.
We finally get the following representation `[0, 4[ ∪ [6, 9[ ∪ [42, +∞[`.
And so the `Range` type is defined as follows.

```rust
pub struct Range<V: Version> {
    segments: Vec<Interval<V>>,
}
type Interval<V> = (V, Option<V>);
// set = [0, 4[ ∪ [6, 9[ ∪ [42, +∞[
let set = vec![(0, Some(4)), (6, Some(9)), (42, None)];
```

## The bounded interval design

We may want however to have versions live in a continuous space.
For example, if we want to use fractions, we can always build a new fraction between two others.
As such it is impossible to define the successor of a fraction version.

We are currently investigating the use of bounded intervals to enable continuous spaces for versions.
If it happens, this will only be in the next major release of pubgrub, probably 0.3.
The current experiments look like follows.

```rust
/// New trait for versions.
/// Bound is core::ops::Bound.
pub trait Version: Clone + Ord + Debug + Display {
    /// Returns the minimum version.
    fn minimum() -> Bound<Self>;
    /// Returns the maximum version.
    fn maximum() -> Bound<Self>;
}

/// An interval is a bounded domain containing all values
/// between its starting and ending bounds.
/// RangeBounds is core::ops::RangeBounds.
pub trait Interval<V>: RangeBounds<V> + Debug + Clone + Eq + PartialEq {
    /// Create an interval from its starting and ending bounds.
    /// It's the caller responsability to order them correctly.
    fn new(start_bound: Bound<V>, end_bound: Bound<V>) -> Self;
}

/// The new Range type is composed of bounded intervals.
pub struct Range<I> {
    segments: Vec<I>,
}
```

It is certain though that the flexibility of enabling usage of continuous spaces will come at a performance price.
We just have to evaluate how much it costs and if it is worth sharing a single implementation, or having both a discrete and a continuous implementation.
