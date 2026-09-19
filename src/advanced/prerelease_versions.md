# Pre-release versions

Pre-releasing is a very common pattern in the world of versioning. It is however
one of the worst to take into account in a dependency system, and I highly
recommend that if you can avoid introducing pre-releases in your package
manager, you should.

In the context of pubgrub, pre-releases break the fundamental properties of the
solver that there is or isn't a version between two versions "x" and "x+1", that
there cannot be a version "(x+1).alpha.1" depending on whether an input version
had a pre-release specifier.

Pre-releases are often semantically linked to version constraints written by
humans, interpreted differently depending on context. For example, "2.0.0-beta"
is meant to exist previous to version "2.0.0". Yet, in many versioning schemes
it is not supposed to be contained in the set described by `1.0.0 <= v < 2.0.0`,
and only within sets where one of the bounds contains a pre-release marker such
as `2.0.0-alpha <= v < 2.0.0`. This poses a problem to the dependency solver
because of backtracking. Indeed, the PubGrub algorithm relies on knowledge
accumulated all along the propagation of the solver front. And this knowledge is
composed of facts, that are thus never removed even when backtracking happens.
Those facts are called incompatibilities and more info about those is available
in the "Internals" section of the guide. The problem is that if a fact recalls
that there is no version within the `1.0.0 <= v < 2.0.0` range, backtracking to
a situation where we ask for a version within `2.0.0-alpha <= v < 2.0.0` will
return nothing even without checking if a pre-release exists in that range. And
this is one of the fundamental mechanisms of the algorithm, so we should not try
to alter it.

## Playing again with packages?

In the light of the "bucket" and "proxies" scheme we introduced in the section
about allowing multiple versions per package, I'm wondering if we could do
something similar for pre-releases. Normal versions and pre-release versions
would be split into two subsets, each attached to a different bucket. In order
to make this work, we would need a way to express negative dependencies. For
example, we would want to say: "a" depends on "b" within the (2.0, 3.0) range
and is incompatible with any pre-release version of "b". The tool to express
such dependencies is already available in the form of `Term` which can be a
`Positive` range or a `Negative` one. We would have to adjust the API for the
`get_dependencies` method to return terms instead of a ranges. This may have
consequences on other parts of the algorithm and should be thoroughly tested.

One issue is that the proxy and bucket scheme would allow for having both a
normal and a pre-release version of the same package in dependencies. We do not
want that, so instead of proxy packages, we might have "frontend" packages. The
difference being that a proxy links a source to a target, while a frontend does
not care about the source, only the target. As such, only one frontend version
can be selected, thus followed by either a normal version or a pre-release
version but not both.

Another issue would be that the proxy and bucket scheme breaks strategies
depending on ordering of versions. Since we have two proxy versions, one
targeting the normal bucket, and one targeting the pre-release bucket, a
strategy aiming at the newest versions will lean towards normal or pre-release
depending if the newest proxy version is the one for the normal or pre-release
bucket. Mitigating this issue seems complicated, but hopefully, we are also
exploring alternative API changes that could enable pre-releases.

## Multi-dimensional ranges

Building on top of the `Ranges` API, we can implement a custom `VersionSet` of
multi-dimensional ranges:

```rust
pub struct DoubleRange<V1: Version, V2: Version> {
    normal_range: Range<V1>,
    prerelease_range: Range<V2>,
}
```

With multi-dimensional ranges we can match the semantics of version constraints
in ways that do not introduce alterations of the core of the algorithm. For
example, the constraint `2.0.0-alpha <= v < 2.0.0` can be matched to:

```rust
DoubleRange {
    normal_range: Ranges::empty(),
    prerelease_range: Ranges::between("2.0.0-alpha", "2.0.0"),
}
```

And the constraint `2.0.0-alpha <= v < 2.1.0` has the same `prerelease_range`
but has `2.0.0 <= v < 2.1.0` for the normal range. Those constraints could also
be interpreted differently since not all pre-release systems work the same. But
the important property is that this enables a separation of the dimensions that
do not behave consistently with regard to the mathematical properties of the
sets manipulated.

This strategy is successfully used by
[semver-pubgrub](https://github.com/pubgrub-rs/semver-pubgrub) to model rust
dependencies.
