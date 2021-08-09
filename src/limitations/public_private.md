# Public and Private packages

We just saw in the previous section that it was possible to have multiple versions of the same package, as long as we create independant virtual packages called buckets.
Oftentimes, creating buckets with fixed version boundaries and which are shared among the whole dependency system can cause versions mismatches in dependency chains.
This is for example the case with a chain of packages publicly re-exporting types of their dependencies and interacting with each other.
Concretely, let's say that we depend on two packages "a" and "b" and let's create a conflictual situation.

- "root" depends on "a" at any version
- "root" depends on "b" at version 2.0
- "a" @ 1.0 depends on "b" at version 1.0
- "b" @ 1.0 has no dependencies
- "b" @ 2.0 depends on "c" at version 1.0
- "c" @ 1.0 has no dependencies

Without buckets, there is no solution because "a" depends on "b" at version 2.0 and our root package depends on "b" at version 1.0.
If we introduce buckets with boundaries at major versions, we have a solution since "a#1" depends on bucket "b#1" while we depend on bucket "b#2" and one version can be picked in each bucket.

But, what if package "a" @ 1.0 re-exports in it's public interface a type coming from its dependency "b" @ 1.0?
If we feed a function of "a" with a type from "b" @ 2.0, compilation will fail, complaining about the argument being of the wrong type.
Currently in the rust compiler, this create cryptic error messages of the kind "Expected type T, found type T" where "T" is exactly the same thing.
Of course, those compiler errors should be improved, but there is a way of preventing that situation entirely at the time of solving dependencies instead of at compilation time.
We can call that the public/private scheme.
In consists in marking dependencies with re-exported types as "public", while other dependencies are considered "private".
So instead of saying that "a" depends on "b", we say that "a" publicly depends on "b".
Therefore public dependencies must be unique even across buckets.

Note that there is one inconvenience though, which is that we could reject too early situations that the compiler would have accepted to compile.
Indeed, it's not because "a" has public types of "b" exposed that we are necessarily using them!

Now let's detail a bit more the public/private scheme and how it could be implemented with PubGrub.

## Public subgraph

As we explained above, two versions of a package can only conflict with each other if they interact through a chain of publicly connected dependencies.
This means that any private dependency can cut the chain of public dependencies.
In turn, the dependencies of a private dependency "p" are guaranteed to be independant (usage wise) of the rest of the dependency graph above package "p".
This means that there is not one list of public packages, but multiple subgraphs of public packages, which start either at the root, or at a private dependency within the dependency graph.
And in each public subgraph, there can only be one version of each package.

## Private seeds

We already use the "root" term to designate our root package, so let's call the private root of every public subgraph a "seed" and note it with a dollar sign "\$".
If we say that package "a" has a public dependency to package "b" we can use the same seed for packages "a" and "b" in that subgraph.
The previous example thus becomes the following system.

- "root" depends on "a\$root" at any version (use "root" for the seed)
- "root" depends on "b\$root" at version 2.0 (use "root" for the seed)
- "a\$root" @ 1.0 depends on "b\$root" at version 1.0 (same seed since public dependency)
- "b\$root" @ 1.0 has no dependencies
- "b\$root" @ 2.0 depends on "c\$b@2.0" at version 1.0 (use "b" @ 2.0 for the seed)
- "c\$b@2.0" @ 1.0 has no dependencies

With the above system, there is no solution since the package "b\$root" is required both at version 1.0 and version 2.0.
However, if we now say that "a" has a private dependency to "b" then it is different.

- "root" depends on "a\$root" at any version (use "root" for the seed)
- "root" depends on "b\$root" at version 2.0 (use "root" for the seed)
- "a\$root" @ 1.0 depends on "b\$a@1.0" at version 1.0 (use "a" @ 1.0 for the seed)
- "b\$a@1.0" @ 1.0 has no dependencies
- "b\$root" @ 2.0 depends on "c\$b@2.0" at version 1.0 (use "b" @ 2.0 for the seed)
- "c\$b@2.0" @ 1.0 has no dependencies

Then this has the following solution:

- "a\$root" at version 1.0
- "b\$a@1.0" at version 1.0
- "b\$root" at version 2.0
- "c\$b@2.0" at version 1.0

## Example implementation

TODO
