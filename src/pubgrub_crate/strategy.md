# Strategical decision making in a DependencyProvider

In PubGrub, decision making is the process of choosing the next package and
version that will be appended to the solution being built. Every time such a
decision must be made, potential valid packages are preselected with
corresponding valid ranges of versions. Then, there is some freedom regarding
which of those package versions to choose.

The strategy employed to choose such package and version cannot change the
existence of a solution or not, but can drastically change the performance of
the solver, or the properties of the solution.

In our implementation of PubGrub, decision making responsibility is divided into
two pieces. The resolver takes care of making a preselection for potential
packages and corresponding ranges of versions. Then it's the dependency provider
that has the freedom of employing the strategy of picking a single package
version through `prioritize` and `choose_version`.

## Picking a package

Potential packages basically are packages that appeared in the dependencies of a
package we've already added to the solution. So once a package appear in
potential packages, it will continue to be proposed as such until we pick it, or
a conflict shows up and the solution is backtracked before needing it.

Imagine that one such potential package is limited to a range containing no
existing version, we are heading directly to a conflict! So we are better
dealing with that conflict as soon as possible, instead of delaying it for later
since we will have to backtrack anyway. Consequently, we always want to pick
first a conflictual potential package with no valid version. Similarly,
potential packages with only one valid version give us no choice and limit
options going forward, so we might want to pick such potential packages before
others with more version choices. Generalizing this strategy to picking the
potential package with the lowest number of valid versions is a rather good
heuristic performance-wise. This strategy is the one employed by the
`OfflineDependencyProvider`. You can use the `PackageResolutionStatistics`
passed into `prioritize` for a heuristic for conflicting-ness: The more conflict
a package had, the higher its priority should be.

## Picking a version

In general, letting the dependency provider choose a version in `choose_version`
provides a great deal of flexibility and enables things like

- choosing the newest versions,
- choosing the oldest versions,
- choosing already downloaded versions,
- choosing versions specified in a lock file,

and many other desirable behaviors for the resolver, controlled directly by the
dependency provider.
