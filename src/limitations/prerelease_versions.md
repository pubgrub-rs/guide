# Pre-release versions

Pre-releasing is a very common pattern in the world of versioning.
It is however one of the worst to take into account in a dependency system, and I highly recommend that if you can avoid introducing pre-releases in your package manager, you should.
In the context of pubgrub, pre-releases break two fondamental properties of the solver.

1. Pre-releases act similar to continuous spaces.
2. Pre-releases break the mathematical properties of subsets in a space with total order.

(1) Indeed, it is hard to answer what version comes after "1-alpha0".
Is it "1-alpha1", "1-beta0", "2"?
In practice, we could say that the version that comes after "1-alpha0" is "1-alpha0?" where the "?" character is chosen to be the lowest character in the lexicographic order, but we clearly are on a stretch here and it certainly isn't natural.

(2) Pre-releases are often semantically linked to version constraints written by humans, interpreted differently depending on context.
For example, "2.0.0-beta" is meant to exist previous to version "2.0.0".
Yet, it is not supposed to be contained in the set described by `1.0.0 <= v < 2.0.0`, and only within sets where one of the bounds contains a pre-release marker such as `2.0.0-alpha <= v < 2.0.0`.
This poses a problem to the dependency solver because of backtracking.
Indeed, the PubGrub algorithm relies on knowledge accumulated all along the propagation of the solver front.
And this knowledge is composed of facts, that are thus never removed even when backtracking happens.
Those facts are called incompatibilities and more info about those is available in the "Internals" section of the guide.
The problem is that if a fact recalls that there is no version within the `1.0.0 <= v < 2.0.0` range, backtracking to a situation where we ask for a version within `2.0.0-alpha <= v < 2.0.0` will return nothing even without checking if a pre-release exists in that range.
And this is one of the fundamental mechanisms of the algorithm, so we should not try to alter it.

Point (2) is probably the reason why some pubgrub implementations have issues dealing with pre-releases when backtracking, as can be seen in [an issue of the dart implementation][dart-prerelease-issue].

[dart-prerelease-issue]: https://github.com/dart-lang/pub/pull/3038

## Playing again with packages?

In the light of the "bucket" and "proxies" scheme we introduced in the section about allowing multiple versions per package, I'm wondering if we could do something similar for pre-releases.
Normal versions and pre-release versions would be split into two subsets, each attached to a different bucket.
This is probably not going to work because of places in the algorithm where intersections of ranges are computed.
In those places, only one of the two buckets will have the intersection applied.
So if we intersect `2.0.0-alpha <= v < 2.0.0` with `2.0.0 <= v < 3.0.0`, we want the result to be empty, but with a bucket scheme, I fear that the pre-release bucket will not be concerned by the intersection and we will still have pre-releases within `2.0.0-alpha <= v < 2.0.0` as still valid.

This is why we think pre-releases require a fundamental change in the API, with ranges of versions that can be multi-dimensional.

## Multi-dimensional ranges

We are currently exploring new APIs where `Range` is transformed into a trait, instead of a predefined struct with a single sequence of non-intersecting intervals.
For now, the new trait is called `RangeSet` and could be implemented on structs with multiple dimensions for ranges.

```rust
pub struct DoubleRange<V1: Version, V2: Version> {
    normal_range: Range<V1>,
    prerelease_range: Range<V2>,
}
```

With multi-dimensional ranges we could match the semantics of version constraints in ways that do not introduce alterations of the core of the algorithm.
For example, the constraint `2.0.0-alpha <= v < 2.0.0` could be matched to:

```rust
DoubleRange {
    normal_range: Range::none,
    prerelease_range: Range::between("2.0.0-alpha", "2.0.0"),
}
```

And the constraint `2.0.0-alpha <= v < 2.1.0` would have the same `prerelease_range` but would have `2.0.0 <= v < 2.1.0` for the normal range.
Those constraints could also be intrepreted differently since not all pre-release systems work the same.
But the important property is that this enable a separation of the dimensions that do not behave consistently with regard to the mathematical properties of the sets manipulated.

All this is under ongoing experimentations, to try reaching a sweet spot API-wise and performance-wise.
If you are eager to experiment with all the extensions and limitations mentionned in this section of the guide for your dependency provider, don't hesitate to reach out to us in our [zulip stream][zulip] or in [GitHub issues][issues] to let us know how it went!

[zulip]: https://rust-lang.zulipchat.com/#narrow/stream/260232-t-cargo.2FPubGrub
[issues]: https://github.com/pubgrub-rs/pubgrub/issues
