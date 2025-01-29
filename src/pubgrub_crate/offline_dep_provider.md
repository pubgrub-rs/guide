# Basic example with OfflineDependencyProvider

Let's imagine that we are building a user interface with a menu containing
dropdowns with some icons, icons that we are also directly using in other parts
of the interface. For this scenario our direct dependencies are `menu` and
`icons`, but the complete set of dependencies looks like follows.

- `user_interface` depends on `menu` and `icons`
- `menu` depends on `dropdown`
- `dropdown` depends on `icons`
- `icons` has no dependency

We can model that scenario as follows.

```rust
use pubgrub::{resolve, OfflineDependencyProvider, Ranges};

// Initialize a dependency provider.
let mut dependency_provider = OfflineDependencyProvider::<&str, Ranges<u32>>::new();

// Add all known dependencies.
dependency_provider.add_dependencies(
    "user_interface",
    1u32,
    [("menu", Ranges::full()), ("icons", Ranges::full())],
);
dependency_provider.add_dependencies("menu", 1u32, [("dropdown", Ranges::full())]);
dependency_provider.add_dependencies("dropdown", 1u32, [("icons", Ranges::full())]);
dependency_provider.add_dependencies("icons", 1u32, []);

// Run the algorithm.
let solution = resolve(&dependency_provider, "user_interface", 1u32).unwrap();
```

The key function of the PubGrub version solver is `resolve`. Besides the
dependency provider, it takes the root package and its version as
(`"user_interface"` at version 1) as arguments.

The resolve function gets package metadata from the `DependencyProvider` trait
defined in this crate. That trait defines methods that the resolver can call
when looking for packages and versions to try in the solver loop. For
convenience and for testing purposes, we already provide an implementation of a
dependency provider called `OfflineDependencyProvider`. As the names suggest, it
doesn't do anything fancy and you have to pre-register all known dependencies
with calls to `add_dependencies(package, version, vec_of_dependencies)` before
being able to use it in the `resolve` function.

In the `OfflineDependencyProvider`, dependencies are specified with `Ranges`,
defining version constraints. In most cases, you would use
`Range::between(v1, v2)` which means any version higher or equal to `v1` and
strictly lower than `v2`. In the previous example, we just used `Range::full()`
which means that any version is acceptable.
