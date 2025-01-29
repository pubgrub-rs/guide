# Solution and error reporting

When everything goes well, the algorithm finds and returns a set of packages and
versions satisfying all the constraints of direct and indirect dependencies.
Sometimes however, there is no solution because dependencies are incompatible.
In such cases, the algorithm returns a
`PubGrubError::NoSolution(derivation_tree)` where the provided derivation tree
is a binary tree containing the chain of reasons why there is no solution.

The items in the tree are called incompatibilities and may be either external,
derived or custom. Leaves of the tree are external and custom incompatibilities,
and nodes are derived. External incompatibilities express facts that are
independent of the way this algorithm is implemented such as package "a" at
version 1 depends on package "b" at version 4, or that there is no version of
package "a" higher than version 5. Custom incompatibilities are a user provided
generic type parameter that can express missing versions, such as that
dependencies of package "a" are not in cache, but the user requested an offline
resolution.

In contrast, derived incompatibilities are obtained during the algorithm
execution by deduction, such as if "a" depends on "b" and "b" depends on "c",
then "a" depends on "c".

Processing a derivation tree in a custom way to generate a failure report that
is human-friendly is not an easy task. For convenience, this crate provides a
`DefaultStringReporter` able to convert a derivation tree into a human-friendly
`String` explanation of the failure. You may use it as follows.

```rust
use pubgrub::{resolve, DefaultStringReporter, PubGrubError, Reporter};

match resolve(&dependency_provider, root_package, root_version) {
    Ok(solution) => println!("{:?}", solution),
    Err(PubGrubError::NoSolution(mut derivation_tree)) => {
        derivation_tree.collapse_no_versions();
        eprintln!("{}", DefaultStringReporter::report(&derivation_tree));
    }
    Err(err) => panic!("{:?}", err),
};
```

Notice that we also used `collapse_no_versions()` above. This method simplifies
the derivation tree to get rid of the `NoVersions` external incompatibilities in
the derivation tree. So instead of seeing things like this in the report:

```txt
Because there is no version of foo in 1.0.1 <= v < 2.0.0
and foo 1.0.0 depends on bar 2.0.0 <= v < 3.0.0,
foo 1.0.0 <= v < 2.0.0 depends on bar 2.0.0 <= v < 3.0.0.
...
```

You will directly see something like:

```txt
Because foo 1.0.0 <= v < 2.0.0 depends on bar 2.0.0,
...
```
