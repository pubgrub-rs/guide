# Public and Private packages

We just saw in the previous section that it was possible to have multiple versions of the same package, as long as we create independant virtual packages called buckets.
Oftentimes, creating buckets with fixed version boundaries and which are shared among the whole dependency system can cause versions mismatches in dependency chains.
This is for example the case with a chain of packages publicly re-exporting types of their dependencies and interacting with each other.
Concretely, let's say that we depend on two packages "a" and "b", with "a" also depending on "b" but at a different version.

- "root" depends on "a" @ 1
- "root" depends on "b" @ 1
- "a" @ 1 depends on "b" @ 2

Without buckets, there is no solution since "b" is depended on at two different versions.
If we introduce buckets with boundaries at major versions, we have a solution since "root" depends on "b#1" while "a#1" depends on bucket "b#2".

But, what if package "a" re-exports in it's public interface a type coming from its "b" dependency?
If we feed a function of "a" with a type from "b#1", compilation will fail, complaining about the argument being of the wrong type.
Currently in the rust compiler, this creates cryptic error messages of the kind "Expected type T, found type T", where "T" is exactly the same thing.
Of course, those compiler errors should be improved, but there is a way of preventing that situation entirely at the time of solving dependencies instead of at compilation time.
We can call that the public/private scheme.
It consists in marking dependencies with re-exported types as "public", while other dependencies are considered "private" since they don't leak their types.
So instead of saying that "a" depends on "b", we say that "a" publicly depends on "b".
Therefore public dependencies must be unique even across multiple major versions.

Note that there is one inconvenience though, which is that we could have false positives, i.e. reject situations that the compiler would have accepted to compile.
Indeed, it's not because "a" has public types of "b" exposed that we are necessarily using them!
Now let's detail a bit more the public/private scheme and how it could be implemented with PubGrub.

## Public subgraph

Two versions of a package can only conflict with each other if they interact through a chain of publicly connected dependencies.
This means that any private dependency can cut the chain of public dependencies.
If "a" privately depends on "b", dependencies of "b" are guaranteed to be independant (usage wise) of the rest of the dependency graph above package "a".
This means that there is not one list of public packages, but rather multiple subgraphs of publicly connected packages, which start either at the root, or at a private dependency frontier.
And in each public subgraph, there can only be one version of each package.

## Seed markers

Since dependencies form a directed graph, each public subgraph can be uniquely identified by a root package, that we will call the "seed" of the public subgraph.
This "seed" is in fact the source package of a private dependency link, and all publicly connected packages following the target package of the private dependency link can be marked with that seed.
The diagram below provides a visual example of dependency graphs where seed markers are annotated next to each package.

<div style="text-align:center"><img src="/img/private-seed.svg" /></div>

In fact, as soon as there is one private dependency, all branches under the source package must be marked with the seed marker of the source package.
This is because all branches contain code that is used by the source package.
As a result, the number of seed markers along a public dependency chain grows with the number of branches with private dependencies, as visible in the diagram below.

<div style="text-align:center"><img src="/img/multiple-private-seed.svg" /></div>

## Example

Let's consider the previous branching example where "b" is depended on both by our root package and by our dependency "a".
If we note seed markers with a dollar symbol "\$" that example can be adapted to the following system.

- "root" depends on "a\$root" @ 1
- "root" depends on "b\$root" @ 1
- "a\$root" @ 1 depends privately on "b\$a@1" @ 2

Seed markers must correspond to an exact package version because multiple versions of a given package will have different dependency graphs, and we don't want to wrongly assume that all subgraphs are the same for all versions.
Here, since "a" depends privately on "b", "b" is marked with the seed "\$a@1".
Thus, this system has the following solution.

- "a\$root" @ 1
- "b\$root" @ 1
- "b\$a@1" @ 2

If instead of a private dependency, "a" had a public dependency on "b", there would be no new seed marker and it would read:

- "a\$root" @ 1 depends publicly on "b\$root" @ 2

Leading to no solution, since the package "b\$root" is now required both at version 1 and 2.

## Example implementation

The seed markers scheme presented above can easily be implemented with pubgrub by keeping seed markers together with package names in the `Package` type involved in the `DependencyProvider`.
A complete example implementation of this extension allowing public and private dependencies is available in the [`public-private` crate of the `advanced_dependency_providers` repository][public-private-crate].
In that example, packages are of the type `String` and versions of the type `SemanticVersion` defined in pubgrub, which does not account for pre-releases, just the (Major, Minor, Patch) format of versions.

[public-private-crate]: https://github.com/pubgrub-rs/advanced_dependency_providers/tree/main/public-private

### Defining an index of packages

Inside the `index.rs` module, we define a very basic `Index`, holding all packages known, as well as a helper function `add_deps` easing the writing of tests.

```rust
/// Each package is identified by its name.
pub type PackageName = String;
/// Alias for dependencies.
pub type Deps = Map<PackageName, (Privacy, Range<SemVer>)>;
/// Privacy indicates if a dependency is public or private.
pub enum Privacy { Public, Private }

/// Global registry of known packages.
pub struct Index {
    /// Specify dependencies of each package version.
    pub packages: Map<PackageName, BTreeMap<SemVer, Deps>>,
}

// Initialize an empty index.
let mut index = Index::new();
// Add package "a" to the index at version 1.0.0 with no dependency.
index.add_deps::<R>("a", (1, 0, 0), &[]);
// Add package "a" to the index at version 2.0.0 with a private dependency to "b" at versions >= 1.0.0.
index.add_deps("a", (2, 0, 0), &[("b", Private, (1, 0, 0)..)]);
// Add package "a" to the index at version 3.0.0 with a public dependency to "b" at versions >= 1.0.0.
index.add_deps("a", (3, 0, 0), &[("b", Public, (1, 0, 0)..)]);
```

### Implementing a dependency provider for the index

Since our `Index` is ready, we now have to implement the `DependencyProvider` trait for it.
As explained previously, we need to identify to which public subgraph a given dependency belongs to.
That is why each package also holds a seed, which is an identifier of the package just before the private dependency initiating the public subgraph.
Thanks to that seed, we guarantee that there can only be one version of each package per public subgraph.

```rust
/// A package is identified by its name and by the public subgraphs
/// it belongs to, themselves identified by "seeds".
pub struct Package {
    name: String,
    seeds: PkgSeeds,
}

/// Each public subgraph is identified by a seed,
/// and some packages belong to multiple public subgraphs
/// and can thus have multiple seed markers.
/// Since we also need to have a unique hash per package, per public subgraph,
/// Each `Markers` variant of a package will also have a dependency on
/// one `Constraint` variant per seed, resulting in one unique identifier
/// per public subgraph that PubGrub can use to check constraints on versions.
pub enum PkgSeeds {
    Constraint(Seed),
    Markers {
        seed_markers: Set<Seed>,
        pkgs: Set<String>,
    },
}

/// A seed is the identifier associated with the private package
/// at the origin of a public subgraph.
pub struct Seed {
    /// Seed package identifier
    pub name: String,
    /// Seed version identifier
    pub version: SemVer,
}
```

Implementing `choose_package_version` is trivial if we simply use the function `choose_package_with_fewest_versions` provided by pubgrub.
Implementing `get_dependencies` is slightly more complicated.
Have a look at the complete implementation if needed, the main ideas are the following.

```rust
fn get_dependencies(&self, package: &Package, version: &SemVer)
-> Result<Dependencies<Package, SemVer>, ...> {
    match &package.seeds {
        // A Constraint variant does not have any dependencies
        PkgSeeds::Constraint(_) => Ok(Dependencies::Known(Map::default())),
        // A Markers variant has dependencies to:
        // - one Constraint variant per seed marker
        // - one Markers variant per original dependency
        PkgSeeds::Markers { seed_markers, pkgs } => {
            let seed_constraints = ...;
            // Figure out if there are are private dependencies.
            let has_private = ...;
            // Chain the seed constraints with actual dependencies.
            Ok(Dependencies::Known(
                seed_constraints
                    .chain(index_deps.iter().map(|(p, (privacy, r))| {
                        let seeds = if privacy == &Privacy::Private {
                            // make a singleton seed package
                        } else if has_private {
                            // add the current package to the seed markers
                        } else {
                            // simply reuse the parent seed markers
                        };
                        ( Package { name: p, seeds }, r )
                    }))
                    .collect(),
            ))
        }
    }
}
```
