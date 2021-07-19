# Allowing multiple versions of a package

One of the main goals of PubGrub is to pick one version per package depended on, under the assumption that at most one version per package is allowed.
This assumption has two advantages.
First, it prevents data and functions interactions of the same library at different, potentially incompatible versions.
Second, it reduces the code size and therefore also the disk usage and the compilation times.

However, under some circonstances, we may relax the "single version allowed" constraint.
Imagine that our package "a" depends on "b" and "c", and "b" depends on "d" @ 1, and "c" depends on "d" @ 2, with versions 1 and 2 of "d" incompatible with each other.
With the default rules of PubGrub, that system has no solution and we cannot build "a".
Yet, if our usages of "b" and "c" are independant with regard to "d", we could in theory build "b" with "d" @ 1 and build "c" with "d" @ 2.

Such situation is sufficiently common that most package managers allow multiple versions of a package.
In rust for example, multiple versions are allowed as long as they cross a major frontier, so 0.7 and 0.8 or 2.0 and 3.0.
So the question now is can we circumvent this fundamental restriction of PubGrub?
The short answer is yes, with buckets and proxis.

## Buckets

We saw that implementing optional dependencies required the creation of on-demand feature packages, which are virtual packages created for each optional feature.
In order to allow for multiple versions of the same package to coexist, we are also going to play smart with packages.
Indeed, if we want two versions to coexist, there is only one possibility, which is that they are from different packages.
And so we introduce the concept of buckets.
A package bucket is a set of versions of the same package that cannot coexist in the solution of a dependency system, basically just like before.
So for rust crates, we can define one bucket per major frontier, such as one bucket for 0.1, one for 0.2, ..., one for 1.0, one for 2.0, etc.
As such, versions 0.7.0 and 0.7.3 would be in the same bucket, but 0.7.0 and 0.8.0 would be in different buckets and could coexist.

Are buckets enought? Not quite, because once we introduce buckets, we need a way to depend on multiple buckets alternatively.
Indeed, most packages should have dependencies on a single bucket, because it doesn't make sense to depend on potentially incompatible versions at the same time.
But rarely, dependencies are written such that they can depend on multiple buckets, such as if one write `v >= 2.0`.
Then, any version from the 2.0 bucket would satisfy it, as well as any version from the 3.0 or any other later bucket.
Thus, we cannot write "a depends on b in bucket 2.0".
So how do we write dependencies of "a"?
That's where we introduce the concept of proxis.

## Proxis

A proxi package is an intermediate on-demand package, placed just between one package and one of its dependencies.
So if we need to express that package "a" has a dependency on package "b" for different buckets, we create the intermediate proxi package "a->b".
Then we can say instead that package "a" depends on any version of the proxi package "a->b".
And then, we create one proxi version for each bucket.
So if there exists the following five buckets for "b", 0.1, 0.2, 1.0, 2.0, 3.0, we create five corresponding versions for the proxi package "a->b".
And since our package "a" depends on any version of the proxi package "a->b", it will be satisfied as soon as any bucket of "b" has a version picked.

## Example

We will consider versions in the form `Major.minor` with a major component starting at 1, and a minor component starting at 0.
The smallest version is 1.0, and each major component represents a bucket.
For convenience, we will use a string notation for proxis and buckets.
Buckets will be indicated by a "#", so "a#1" is the 1.0 bucket of package "a", and "a#2" is the 2.0 bucket of package "a".
And we will use "@" to denote exact versions, so "a" @ 1.0 means package "a" at version 1.0.
Proxis will be represented by the arrow "->" as explained before.
Since a proxi is tied to a specific version of the initial package, we also use the "@" symbol in the name of the proxi package.
For example, if "a" @ 1.4 depends on "b", we create the proxi package "a#1@1.4->b".
It's a bit of a mouthful, but once you learn how to read it, it makes sense.

Let's consider the following example, with "a" being our root package.
- "a" @ 1.4 depends on "b" in range `1.1 <= v < 2.9`
- "b" @ 1.3 depends on "c" at version `1.1`
- "b" @ 2.7 depends on "d" at version `3.1`

Using buckets and proxis, we can rewrite this dependency system as follows.
- "a#1" @ 1.4 depends on "a#1@1.4->b" at any version (we use the proxi).
- "a#1@1.4->b" proxi exists in two versions, one per bucket of "b".
- "a#1@1.4->b" @ 1.0 depends on "b#1" in range `1.1 <= v < 2.0` (the first bucket intersected with the dependency range).
- "a#1@1.4->b" @ 2.0 depends on "b#2" in range `2.0 <= v < 2.9` (the second bucket intersected with the dependency range).
- "b#1" @ 1.3 depends on "c#1" at version `1.1`.
- "b#2" @ 2.7 depends on "d#3" at version `3.1`.

There are potentially two solutions to that system, the first one being the following.
- "a#1" @ 1.4
- "a#1@1.4->b" @ 1.0
- "b#1" @ 1.3

And if we want to express the solution in terms of the original packages, we just have to remove the proxi packages from the solution.

## Example implementation

TODO
