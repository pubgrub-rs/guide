# Optional dependencies

Sometimes, we want the ability to express that some functionalities are
optional. Since those functionalities may depend on other packages, we also want
the ability to mark those dependencies optional.

Let's imagine a simple scenario in which we are developing package "A" and
depending on package "B", itself depending on "C". Now "B" just added new
functionalities, very convenient but also very heavy. For architectural reasons,
they decided to release them as optional features of the same package instead of
in a new package. To enable those features, one has to activate the "heavy"
option of package "B", bringing a new dependency to package "H". We will mark
optional features in dependencies with a slash separator "/". So we have now "A"
depends on "B/heavy" instead of previously "A" depends on "B". And the complete
dependency resolution is thus now ["A", "B/heavy", "C", "H"].

But what happens if our package "A" start depending on another package "D",
which depends on "B" without the "heavy" option? We would now have ["A", "B",
"B/heavy", "C", "D", "H"]. Is this problematic? Can we conciliate the fact that
both "B" and "B/heavy" are in the dependencies?

## Strictly additive optional dependencies

The most logical solution to this situation is to require that optional features
and dependencies are strictly additive. Meaning "B/heavy" is entirely compatible
with "B" and only brings new functions and dependencies. "B/heavy" cannot change
dependencies of "B" only adding new ones. Once this hypothesis is valid for our
dependency system, we can model "B/heavy" as a different package entirely,
depending on both "B" and "H", leading to the solution ["A", "B", "B/heavy",
"C", "D", "H"]. Whatever new optional features that get added to "B" can
similarly be modeled by a new package "B/new-feat", also depending on "B" and on
its own new dependencies. When dependency resolution ends, we can gather all
features of "B" that were added to the solution and compile "B" with those.

## Dealing with versions

In the above example, we eluded versions and only talked about packages. Adding
versions to the mix actually does not change anything, and solves the optional
dependencies problem very elegantly. The key point is that an optional feature
package, such as "B/heavy", would depend on its base package, "B", exactly at
the same version. So if the "heavy" option of package "B" at version v = 3
brings a dependency to "H" at v >= 2, then we can model dependencies of
"B/heavy" at v = 3 by ["B" @ v = 3, "H" @ v >= 2].

## Example implementation

A complete example implementation is available in the [`optional-deps` crate of
the `advanced_dependency_providers` repository][optional-deps-crate]. Let's give
an explanation of that implementation. For the sake of simplicity, we will
consider packages of type `String` and versions of type `NumberVersion`, which
is just a newtype around `u32` implementing the `Version` trait.

### Defining an index of packages

We define an `Index`, storing all dependencies (`Deps`) of every package version
in a double map, first indexed by package, then by version.

```rust
/// Each package is identified by its name.
pub type PackageName = String;

/// Global registry of known packages.
pub struct Index {
    /// Specify dependencies of each package version.
    pub packages: Map<PackageName, BTreeMap<u32, Deps>>,
}
```

Dependencies listed in the `Index` include both mandatory and optional
dependencies. Optional dependencies are identified, grouped, and gated by an
option called a "feature".

```rust
pub type Feature = String;

pub struct Deps {
    /// The regular, mandatory dependencies.
    pub mandatory: Map<PackageName, Dep>,
    /// The optional, feature-gated dependencies.
    pub optional: Map<Feature, Map<PackageName, Dep>>,
}
```

Finally, each dependency is specified with a version range, and with a set of
activated features.

```rust
pub struct Dep {
    /// The range dependended upon.
    pub range: Range<Version>,
    /// The activated features for that dependency.
    pub features: Set<Feature>,
}
```

For convenience, we added the `add_deps` and `add_feature` functions to help
building an index in the tests.

```rust
let mut index = Index::new();
// Add package "a" to the index at version 0 with no dependency.
index.add_deps("a", 0, &[]);
// Add package "a" at version 1 with a dependency to "b" at version "v >= 1".
index.add_deps("a", 1, &[("b", 1.., &[])]);
// Add package "c" at version 1 with a dependency to "d" at version "v < 4" with the feature "feat".
index.add_deps("c", 1, &[("d", ..4, &["feat"])]);
// Add feature "feat" to package "d" at version 1 with the optional dependency to "f" at version "v >= 1".
// If "d" at version 1 does not exist yet in the index, this also adds it with no mandatory dependency,
// as if we had called index.add_deps("d", 1, &[]).
index.add_feature("d", 1, "feat", &[("f", 1.., &[])]);
```

### Implementing a dependency provider for the index

Now that our `Index` is ready, let's implement the `DependencyProvider` trait on
it. As we explained before, we'll need to differenciate optional features from
base packages, so we define a new `Package` type.

```rust
/// A package is either a base package like "a",
/// or a feature package, corresponding to a feature associated to a base package.
pub enum Package {
    Base(String),
    Feature { base: String, feature: String },
}
```

Any `prioritize` will work equally well for this example, even just returning a
constant value.

Let's implement the second function required by a dependency provider,
`choose_version`. For that we defined the `base_pkg()` method on a `Package`
that returns the string of the base package, and the `available_versions()`
method on an `Index` to list existing versions of a given package in descending
order.

```rust
fn choose_version(
    &self,
    package: &Self::P,
    range: &Self::VS,
) -> Result<Option<Self::V>, Self::Err> {
    Ok(self.available_versions(p.base_pkg()).find(|version| range.contains(version)).cloned())
}
```

This was very straightforward. Implementing the second function,
`get_dependencies`, requires just a bit more work but is also quite easy. The
exact complete version is available in the code, but again, for the sake of
simplicity and readability, let's just deal with the happy path.

```rust
fn get_dependencies(
    &self,
    package: &Package,
    version: &Version,
) -> Result<Dependencies<Package, Version>, ...> {
    // Retrieve the dependencies in the index.
    let deps =
        self.packages
            .get(package.base_pkg()).unwrap()
            .get(version).unwrap();

    match package {
        // If we asked for a base package, we simply return the mandatory dependencies.
        Package::Base(_) => Ok(Dependencies::Available(from_deps(&deps.mandatory))),
        // Otherwise, we concatenate the feature deps with a dependency to the base package.
        Package::Feature { base, feature } => {
            let feature_deps = deps.optional.get(feature).unwrap();
            let mut all_deps = from_deps(feature_deps);
            all_deps.insert(
                Package::Base(base.to_string()),
                Range::exact(version.clone()),
            );
            Ok(Dependencies::Available(all_deps))
        },
    }
}
```

Quite easy right? The helper function `from_deps` is where each dependency is
transformed from its `String` form into its `Package` form. In pseudo-code (not
compiling, but you get how it works) it looks like follows.

```rust
/// Helper function to convert Index deps into what is expected by the dependency provider.
fn from_deps(deps: &Map<String, Dep>) -> DependencyConstraints<Package, Version> {
    deps.iter().map(create_feature_deps)
        .flatten()
        .collect()
}

fn create_feature_deps(
    (base_pkg, dep): (&String, &Dep)
) -> impl Iterator<Item = (Package, Range<Version>)> {
    if dep.features.is_empty() {
        // If the dependency has no optional feature activated,
        // we simply add a dependency to the base package.
        std::iter::once((Package::Base(base_pkg), dep.range))
    } else {
        // Otherwise, we instead add one dependency per activated feature,
        // but not to the base package itself (we could but no need).
        dep.features.iter().map(|feat| {
            (Package::Feature { base: base_pkg, feature: feat }, dep.range)
        })
    }
}
```

We now have implemented the `DependencyProvider` trait. The only thing left is
testing it with example dependencies. For that, we setup few helper functions
and we wrote some tests, that you can run with a call to `cargo test --lib`.
Below is one of those tests.

```rust
#[test]
fn success_when_multiple_features() {
    let mut index = Index::new();
    index.add_deps("a", 0, &[("b", .., &["feat1", "feat2"])]);
    index.add_feature("b", 0, "feat1", &[("f1", .., &[])]);
    index.add_feature("b", 0, "feat2", &[("f2", .., &[])]);
    index.add_deps::<R>("f1", 0, &[]);
    index.add_deps::<R>("f2", 0, &[]);
    assert_map_eq(
        &resolve(&index, "a", 0).unwrap(),
        &select(&[
            ("a", 0),
            ("b", 0),
            ("b/feat1", 0),
            ("b/feat2", 0),
            ("f1", 0),
            ("f2", 0),
        ]),
    );
}
```

[optional-deps-crate]:
  https://github.com/pubgrub-rs/advanced_dependency_providers/tree/main/optional-deps
