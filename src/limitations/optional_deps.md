# Optional dependencies

Sometimes, we want the ability to express that some functionalities are optional.
Since those functionalities may depend on other packages, we also want the ability to mark those dependencies optional.

Let's imagine a simple scenario in which we are developing package "A" and depending on package "B", itself depending on "C".
Now "B" just added new functionalities, very convenient but also very heavy.
For architectural reasons, they decided to release them as optional features of the same package instead of in a new package.
To enable those features, one has to activate the "heavy" option of package "B", bringing a new dependency to package "H".
We will mark optional features in dependencies with a slash separator "/".
So we have now "A" depends on "B/heavy" instead of previously "A" depends on "B".
And the complete dependency resolution is thus now ["A", "B/heavy", "C", "H"].

But what happens if our package "A" start depending on another package "D", which depends on "B" without the "heavy" option?
We would now have ["A", "B", "B/heavy", "C", "D", "H"].
Is this problematic?
Can we conciliate the fact that both "B" and "B/heavy" are in the dependencies?

## Strictly additive optional dependencies

The most logical solution to this situation is to require that optional features and dependencies are strictly additive.
Meaning "B/heavy" is entirely compatible with "B" and only brings new functions and dependencies.
"B/heavy" cannot change dependencies of "B" only adding new ones.
Once this hypothesis is valid for our dependency system, we can model "B/heavy" as a different package entirely, depending on both "B" and "H", leading to the solution ["A", "B", "B/heavy", "C", "D", "H"].
Whatever new optional features that get added to "B" can similarly be modeled by a new package "B/new-feat", also depending on "B" and on its own new dependencies.
When dependency resolution ends, we can gather all features of "B" that were added to the solution and compile "B" with those.

## Dealing with versions

In the above example, we eluded versions and only talked about packages.
Adding versions to the mix actually does not change anything, and solves the optional dependencies problem very elegantly.
The key point is that an optional feature package, such as "B/heavy", would depend on its base package, "B", exactly at the same version.
So if the "heavy" option of package "B" at version v = 3 brings a dependency to "H" at v >= 2, then we can model dependencies of "B/heavy" at v = 3 by ["B" @ v = 3, "H" @ v >= 2].

## Example implementation

TODO: link and explain the example in advanced_dependency_providers.
