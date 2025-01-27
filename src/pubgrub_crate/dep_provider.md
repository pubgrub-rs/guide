# Implementing a dependency provider

The `OfflineDependencyProvider` is very useful for testing and playing with the
API, but is not sufficient in complex setting such as Cargo. In those cases, a
dependency provider may need to retrieve package information from caches, from
the disk or from network requests.

PubGrub is generic over all its internal types. You need:

- A package type `P` that implements `Clone + Eq + Hash + Debug + Display`, for
  example `String`
- A version type `V` that implements `Debug + Display + Clone + Ord`, for
  example `SemanticVersion`
- A version set type `VS` that implements
  `VersionSet + Debug + Display + Clone + Eq`, for example
  `version_ranges::Ranges`. `VersionSet` is defined as:

  ```rust
  pub trait VersionSet: Debug + Display + Clone + Eq {
    type V: Debug + Display + Clone + Ord;

    // Constructors

    /// An empty set containing no version.
    fn empty() -> Self;

    /// A set containing only the given version.
    fn singleton(v: Self::V) -> Self;

    // Operations

    /// The set of all version that are not in this set.
    fn complement(&self) -> Self;

    /// The set of all versions that are in both sets.
    fn intersection(&self, other: &Self) -> Self;

    /// Whether the version is part of this set.
    fn contains(&self, v: &Self::V) -> bool;
  }
  ```

- A package priority `Priority that implements `Ord +
  Clone`, for example `usize`
- A type for custom incompatibilities `Incompatibility` that implements
  `Eq + Clone + Debug + Display`, for example `String`
- The error type returned from the `DependencyProvider` implements
  `Error + 'static`, for example `anyhow::Error`

While PubGrub is generic to encourage bringing your own types tailored to your
use, it also provides some convenience types. For versions, we provide the
`SemanticVersion` and the `version_ranges::Ranges` types. `SemanticVersion`
implements `Version` for versions expressed as `Major.Minor.Patch`. `u32` also
fulfills all requirements of a version. `Ranges` represents multiple intervals
of a continuous ranges with inclusive or exclusive bounds, e.g.
`>=1.3.0,<2 || >4,<=6 || >=7.1`.

`DependencyProvider` requires implementing three functions:

```rust
pub trait DependencyProvider<P: Package, V: Version> {
  fn prioritize(
    &self,
    package: &Self::P,
    range: &Self::VS,
    package_conflicts_counts: &PackageResolutionStatistics,
  ) -> Self::Priority;

  fn choose_version(
    &self,
    package: &Self::P,
    range: &Self::VS,
  ) -> Result<Option<Self::V>, Self::Err>;

  fn get_dependencies(
    &self,
    package: &Self::P,
    version: &Self::V,
  ) -> Result<Dependencies<Self::P, Self::VS, Self::Incompatibility>, Self::Err>;

}
```

`prioritize` determines the order in which versions are chosen for packages.

Decisions are always made for the highest priority package first. The order of
decisions determines which solution is chosen and can drastically change the
performances of the solver. If there is a conflict between two package versions,
the package with the higher priority is preserved and the lower priority gets
discarded. Usually, you want to decide more certain packages (e.g. those with a
single version constraint) and packages with more conflicts first. This function
has potentially the highest impact on resolver performance.

Example:

- The root package depends on A and B
- A 2 depends on C 2
- B 2 depends on C 1
- A 1 has no dependencies
- B 1 has no dependencies

If we always try the highest version first, prioritization determines the
solution: If A has higher priority, the solution will be A 2, B 1, if B has
higher priority, the solution will be A 1, B 2.

The `package_conflicts_counts` argument provides access to some other heuristics
that are production users have found useful. Although the exact meaning/efficacy
of those arguments may change.

The function is called once for each new package and then cached until we detect
a (potential) change to `range`, otherwise it is cached, assuming that the
priority only depends on the arguments to this function.

If two packages have the same priority, PubGrub will bias toward a breadth first
search.

Once the highest priority package has been determined, we want to make a
decision for it and need a version. `choose_version` returns the next version
for to try for this package, which in the current partial solution is
constrained by `range`. If you are trying to solve for the latest version,
return the highest version of the package contained in `range`.

Finally, the solver needs to know the dependency of that version to determine
which packages to try next or if there are any conflicts. `get_dependencies`
returns the dependencies of a given package version. Usually, you return
`Ok(Dependencies::Available(rustc_hash::FxHashMap::from(...)))`, where the map
is the list of dependencies with their version ranges. You can also return
`Dependencies::Unavailable` if the dependencies could not be retrieved, but you
consider this error non-fatal, for example because the version use an
unsupported metadata format.

Aside from the required methods, there is an optional `should_cancel` method.
This method is regularly called in the solver loop, currently once per decision,
and defaults to doing nothing. If needed, you can override it to provide custom
behavior, such as giving some feedback, or stopping early to prevent ddos. Any
useful behavior would require mutability of `self`, and that is possible thanks
to interior mutability. Read on the [next section](./caching.md) for more info
on that!
