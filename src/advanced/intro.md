# Advanced usage and limitations

By design, the current implementation of PubGrub is rather well suited to handle
a dependency system with the following constraints:

1. Packages are uniquely identified.
2. Versions are in a discrete set, with a total order.
3. Dependencies of a package version are fixed.
4. Exactly one version must be selected per package depended on.

The fact that packages are uniquely identified (1) is perhaps the only
constraint that makes sense for all common dependency systems. But for the rest
of the constraints, they are all inadequate for some common real-world
dependency systems. For example, it's possible to have dependency systems where
order is not required for versions (2). In such systems, dependencies must be
specified with exact sets of compatible versions, and bounded ranges make no
sense. Having fixed dependencies (3) is also not followed in programming
languages allowing optional dependencies. In Rust packages, optional
dependencies are called "features" for example. Finally, restricting solutions
to only one version per package (4) is also too constraining for dependency
systems allowing breaking changes. In cases where packages A and B both depend
on different ranges of package C, we sometimes want to be able to have a
solution where two versions of C are present, and let the compiler decide if
their usages of C in the code are compatible.

In the following subsections, we try to show how we can circumvent those
limitations with clever usage of dependency providers.
