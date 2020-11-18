# Writing your own error reporting logic

The `DerivationTree` is a custom binary tree
where leaves are external incompatibilities, defined as follows,

```rust
pub enum External<P: Package, V: Version> {
    /// Initial incompatibility aiming at picking the root package for the first decision.
    NotRoot(P, V),
    /// No versions from range satisfy given constraints.
    NoVersions(P, Range<V>),
    /// Dependencies of the package are unavailable for versions in that range.
    UnavailableDependencies(P, Range<V>),
    /// Incompatibility coming from the dependencies of a given package.
    FromDependencyOf(P, Range<V>, P, Range<V>),
}
```

and nodes are derived incompatibilities, defined as follows.

```rust
pub struct Derived<P: Package, V: Version> {
    /// Terms of the incompatibility.
    pub terms: Map<P, Term<V>>,
    /// Indicate if that incompatibility is present multiple times
    /// in the derivation tree.
    /// If that is the case, it has a unique id, provided in that option.
    /// Then, we may want to only explain it once,
    /// and refer to the explanation for the other times.
    pub shared_id: Option<usize>,
    /// First cause.
    pub cause1: Box<DerivationTree<P, V>>,
    /// Second cause.
    pub cause2: Box<DerivationTree<P, V>>,
}
```

The `terms` hashmap contains the terms of the derived incompatibility.
The rule is that terms of an incompatibility are terms that
cannot be all true at the same time.
So a dependency can for example be expressed with an incompatibility
containing a positive term, and a negative term.
For example, `"root"` at version 1 depends on `"a"` at version 4,
can be expressed by the incompatibility `{root: 1, a: not 4}`.
A missing version can be expressed by an incompatibility
with a single term. So for example, if version 4 of package `"a"` is missing,
it can be expressed with the incompatibility `{a: 4}` which forbids that version.
If you want to write your own reporting logic,
I'd highly suggest a good understanding of incompatibilities
by reading first the section of this book on internals of the PubGrub algorithm.

The root of the derivation tree is usually a derived incompatibility
containing a single term such as `{ "root": 1 }`
if we were trying to solve dependencies for package `"root"` at version 1.
Imagine we had one simple dependency on `"a"` at version 4,
but somehow that version does not exist.
Then version solving would fail and return a derivation tree looking like follows.

```txt
Derived { root: 1 }
	├─ External dependency { root: 1, a: not 4 }
	└─ External no version { a: 4 }
```

The goal of error reporting is to transform that tree into a friendly
human-readable output. It could be something like:

```txt
This project depends on version 4 of package a which does not exist
so version solving failed.
```

One of the subtleties of error reporting is that the binary derivation tree
is actually a DAG (directed acyclic graph).
Occasionally, an incompatibility may cause multiple derived incompatibilities
in the same derivation tree such as below,
where the incompatibility 2 is used to derive both incompatibilities 4 and 5.

```txt
                          +----------+        +----------+
                 +------->|incompat 4|------->|incompat 1|
                 |        +----------+        +----------+
                 |                |
          +----------+            |           +----------+
  (root)  |incompat 6|            +---------->|incompat 2|  (shared)
          +----------+            |           +----------+
                 |                |
                 |        +----------+        +----------+
                 +------->|incompat 5|------->|incompat 3|
                          +----------+        +----------+
```

For ease of processing, the `DerivationTree` duplicates such nodes in the tree,
but their `shared_id` attribute will hold a `Some(id)` variant.
In error reporting, you may want to check if you already gave an explanation
for a shared derived incompatibility,
and in such cases maybe use line references instead of re-explaning the same thing.
