# Allowing multiple versions of a package

One of the main goals of PubGrub is to pick one version per package depended on, under the assumption that at most one version per package is allowed.
Enforcing this assumption has two advantages.
First, it prevents data and functions interactions of the same library at different, potentially incompatible versions.
Second, it reduces the code size and therefore also the disk usage and the compilation times.

However, under some circonstances, we may relax the "single version allowed" constraint.
Imagine that our package "a" depends on "b" and "c", and "b" depends on "d" @ 1, and "c" depends on "d" @ 2, with versions 1 and 2 of "d" incompatible with each other.
With the default rules of PubGrub, that system has no solution and we cannot build "a".
Yet, if our usages of "b" and "c" are independant with regard to "d", we could in theory build "b" with "d" @ 1 and build "c" with "d" @ 2.

Such situation is sufficiently common that most package managers allow multiple versions of a package.
In Rust for example, multiple versions are allowed as long as they cross a major frontier, so 0.7 and 0.8 or 2.0 and 3.0.
So the question now is can we circumvent this fundamental restriction of PubGrub?
The short answer is yes, with buckets and proxies.

## Buckets

We saw that implementing optional dependencies required the creation of on-demand feature packages, which are virtual packages created for each optional feature.
In order to allow for multiple versions of the same package to coexist, we are also going to play smart with packages.
Indeed, if we want two versions to coexist, there is only one possibility, which is that they come from different packages.
And so we introduce the concept of buckets.
A package bucket is a set of versions of the same package that cannot coexist in the solution of a dependency system, basically just like before.
So for Rust crates, we can define one bucket per major frontier, such as one bucket for 0.1, one for 0.2, ..., one for 1.0, one for 2.0, etc.
As such, versions 0.7.0 and 0.7.3 would be in the same bucket, but 0.7.0 and 0.8.0 would be in different buckets and could coexist.

Are buckets enought? Not quite, since once we introduce buckets, we also need a way to depend on multiple buckets alternatively.
Indeed, most packages should have dependencies on a single bucket, because it doesn't make sense to depend on potentially incompatible versions at the same time.
But rarely, dependencies are written such that they can depend on multiple buckets, such as if one write `v >= 2.0`.
Then, any version from the 2.0 bucket would satisfy it, as well as any version from the 3.0 or any other later bucket.
Thus, we cannot write "a depends on b in bucket 2.0".
So how do we write dependencies of "a"?
That's where we introduce the concept of proxies.

## Proxies

A proxy package is an intermediate on-demand package, placed just between one package and one of its dependencies.
So if we need to express that package "a" has a dependency on package "b" for different buckets, we create the intermediate proxy package "a->b".
Then we can say instead that package "a" depends on any version of the proxy package "a->b".
And then, we create one proxy version for each bucket.
So if there exists the following five buckets for "b", 0.1, 0.2, 1.0, 2.0, 3.0, we create five corresponding versions for the proxy package "a->b".
And since our package "a" depends on any version of the proxy package "a->b", it will be satisfied as soon as any bucket of "b" has a version picked.

## Example

We will consider versions in the form `major.minor` with a major component starting at 1, and a minor component starting at 0.
The smallest version is 1.0, and each major component represents a bucket.

> Note that we could start versions at 0.0, but since different dependency system tends to interpret versions before 1.0 differently, we will simply avoid that problem by saying versions start at 1.0.

For convenience, we will use a string notation for proxies and buckets.
Buckets will be indicated by a "#", so "a#1" is the 1.0 bucket of package "a", and "a#2" is the 2.0 bucket of package "a".
And we will use "@" to denote exact versions, so "a" @ 1.0 means package "a" at version 1.0.
Proxies will be represented by the arrow "->" as previously mentioned.
Since a proxy is tied to a specific version of the initial package, we also use the "@" symbol in the name of the proxy package.
For example, if "a" @ 1.4 depends on "b", we create the proxy package "a#1@1.4->b".
It's a bit of a mouthful, but once you learn how to read it, it makes sense.

> Note that we might be tempted to just remove the version part of the proxy, so "a#1->b" instead of "a#1@1.4->b".
> However, we might have "a" @ 1.4 depending on "b" in range `v >= 2.2` and have "a" @ 1.5 depending on "b" in range `v >= 2.6`.
> Both dependencies would map to the same "a#1->b" proxy package, but we would not know what to put in the dependency of "a#1->b" to the "b#2" bucket.
> Should it be "2.2 <= v < 3.0" as in "a" @ 1.4, or should it be "2.6 <= v < 3.0" as in "a" @ 1.5?
> That is why each proxy package is tied to an exact version of the source package.

Let's consider the following example, with "a" being our root package.
- "a" @ 1.4 depends on "b" in range `1.1 <= v < 2.9`
- "b" @ 1.3 depends on "c" at version `1.1`
- "b" @ 2.7 depends on "d" at version `3.1`

Using buckets and proxies, we can rewrite this dependency system as follows.
- "a#1" @ 1.4 depends on "a#1@1.4->b" at any version (we use the proxy).
- "a#1@1.4->b" proxy exists in two versions, one per bucket of "b".
- "a#1@1.4->b" @ 1.0 depends on "b#1" in range `1.1 <= v < 2.0` (the first bucket intersected with the dependency range).
- "a#1@1.4->b" @ 2.0 depends on "b#2" in range `2.0 <= v < 2.9` (the second bucket intersected with the dependency range).
- "b#1" @ 1.3 depends on "c#1" at version `1.1`.
- "b#2" @ 2.7 depends on "d#3" at version `3.1`.

There are potentially two solutions to that system.
The one with the newest versions is the following.
- "a#1" @ 1.4
- "a#1@1.4->b" @ 2.0
- "b#2" @ 2.7
- "d#3" @ 3.1

Finally, if we want to express the solution in terms of the original packages, we just have to remove the proxy packages from the solution.

## Example implementation

A complete example implementation of this extension allowing multiple versions is available in the [`allow-multiple-versions` crate of the `advanced_dependency_providers` repository][multiple-versions-crate].
In that example, packages are of the type `String` and versions of the type `SemanticVersion` defined in pubgrub, which does not account for pre-releases, just the (Major, Minor, Patch) format of versions.

[multiple-versions-crate]: https://github.com/pubgrub-rs/advanced_dependency_providers/tree/main/allow-multiple-versions

### Defining an index of packages

Inside the `index.rs` module, we define a very basic `Index`, holding all packages known, as well as a helper function `add_deps` easing the writing of tests.

```rust
/// Each package is identified by its name.
pub type PackageName = String;
/// Alias for dependencies.
pub type Deps = Map<PackageName, Range<SemVer>>;

/// Global registry of known packages.
pub struct Index {
    /// Specify dependencies of each package version.
    pub packages: Map<PackageName, BTreeMap<SemVer, Deps>>,
}

// Initialize an empty index.
let mut index = Index::new();
// Add package "a" to the index at version 1.0.0 with no dependency.
index.add_deps::<R>("a", (1, 0, 0), &[]);
// Add package "a" to the index at version 2.0.0 with a dependency to "b" at versions >= 1.0.0.
index.add_deps("a", (2, 0, 0), &[("b", (1, 0, 0)..)]);
```

### Implementing a dependency provider for the index

Since our `Index` is ready, we now have to implement the `DependencyProvider` trait for it.
As explained previously, we'll need to differenciate packages representing buckets and proxies, so we define the following new `Package` type.

```rust
/// A package is either a bucket, or a proxy between a source and a target package.
pub enum Package {
    /// "a#1"
    Bucket(Bucket),
    /// source -> target
    Proxy {
        source: (Bucket, SemVer),
        target: String,
    },
}

/// A bucket corresponds to a given package, and match versions
/// in a range identified by their major component.
pub struct Bucket {
    pub name: String, // package name
    pub bucket: u32, // 1 maps to the range 1.0.0 <= v < 2.0.0
}
```

In order to implement the first required method, `choose_package_version`, we simply reuse the `choose_package_with_fewest_versions` helper function provided by pubgrub.
That one requires a list of available versions for each package, so we have to create that list.
As explained previously, listing the existing (virtual) versions depend on if the package is a bucket or a proxy.
For a bucket package, we simply need to retrieve the original versions and filter out those outside of the bucket.

```rust
match package {
    Package::Bucket(p) => {
        let bucket_range = Range::between((p.bucket, 0, 0), (p.bucket + 1, 0, 0));
        self.available_versions(&p.name)
            .filter(|v| bucket_range.contains(*v))
    }
    ...
```

If the package is a proxy however, there is one version per bucket in the target of the proxy.

```rust
match package {
    Package::Proxy { target, .. } => {
        bucket_versions(self.available_versions(&target))
    }
    ...
}

/// Take a list of versions, and output a list of the corresponding bucket versions.
/// So [1.1, 1.2, 2.3] -> [1.0, 2.0]
fn bucket_versions(
    versions: impl Iterator<Item = SemVer>
) -> impl Iterator<Item = SemVer> { ... }
```

Additionally, we can filter out buckets that are outside of the dependency range in the original dependency leading to that proxy package.
Otherwise it will add wastefull computation to the solver, but we'll leave that out of this walkthrough.

The `get_dependencies` method is slightly hairier to implement, so instead of all the code, we will just show the structure of the function in the happy path, with its comments.

```rust
fn get_dependencies(
    &self,
    package: &Package,
    version: &SemVer,
) -> Result<Dependencies<Package, SemVer>, ...> {
    let all_versions = self.packages.get(package.pkg_name());
    ...
    match package {
        Package::Bucket(pkg) => {
            // If this is a bucket, we convert each original dependency into
            // either a dependency to a bucket package if the range is fully contained within one bucket,
            // or a dependency to a proxy package at any version otherwise.
            let deps = all_versions.get(version);
            ...
            let pkg_deps = deps.iter().map(|(name, range)| {
                    if let Some(bucket) = single_bucket_spanned(range) {
                        ...
                        (Package::Bucket(bucket_dep), range.clone())
                    } else {
                        ...
                        (proxy, Range::any())
                    }
                })
                .collect();
            Ok(Dependencies::Known(pkg_deps))
        }
        Package::Proxy { source, target } => {
            // If this is a proxy package, it depends on a single bucket package, the target,
            // at a range of versions corresponding to the bucket range of the version asked,
            // intersected with the original dependency range.
            let deps = all_versions.get(&source.1);
            ...
            let mut bucket_dep = Map::default();
            bucket_dep.insert(
                Package::Bucket(Bucket {
                    name: target.clone(),
                    bucket: target_bucket,
                }),
                bucket_range.intersection(target_range),
            );
            Ok(Dependencies::Known(bucket_dep))
        }
    }
}

/// If the range is fully contained within one bucket,
/// this returns that bucket identifier, otherwise, it returns None.
fn single_bucket_spanned(range: &Range<SemVer>) -> Option<u32> { ... }
```

That's all!
The implementation also contains tests, with helper functions to build them.
Here is the test corresponding to the example we presented above.

```rust
#[test]
/// Example in guide.
fn success_when_simple_version() {
    let mut index = Index::new();
    index.add_deps("a", (1, 4, 0), &[("b", (1, 1, 0)..(2, 9, 0))]);
    index.add_deps("b", (1, 3, 0), &[("c", (1, 1, 0)..(1, 1, 1))]);
    index.add_deps("b", (2, 7, 0), &[("d", (3, 1, 0)..(3, 1, 1))]);
    index.add_deps::<R>("c", (1, 1, 0), &[]);
    index.add_deps::<R>("d", (3, 1, 0), &[]);
    assert_map_eq(
        &resolve(&index, "a#1", (1, 4, 0)).unwrap(),
        &select(&[("a#1", (1, 4, 0)), ("b#2", (2, 7, 0)), ("d#3", (3, 1, 0))]),
    );
}
```

Implementing a dependency provider allowing both optional dependencies and multiple versions per package is left to the reader as an exercise (I've always wanted to say that).
