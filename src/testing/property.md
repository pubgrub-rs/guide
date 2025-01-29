# Property testing

To test pubgrub, we employ a mix of unit tests, integration tests and property
tests. Some unit tests are present for example in the `version` module to
validate that the parsing of semantic versions is correct. Integration tests are
located in the `tests/` directory. Property tests are co-located both with unit
tests and integration tests depending on required access to some private
implementation details.

`version_ranges` additionally exposes `version_ranges::proptest_strategy` to
help testing both pubgrub and user code.

## Examples

We have multiple example cases inside `tests/examples.rs`. Those mainly come
from [dart documentation of the solver][dart-solver] and are simple end-to-end
examples for which we know the expected results. The first example called
`no_conflict` is a simple case where the root package depends on one package
which itself has a dependency to another package. The tests there compare the
expected result with the solution of the solver for each of those examples.
These were very useful when making progress on PubGrub implementation and can
now also be referred to as example usage of the API.

[dart-solver]: https://github.com/dart-lang/pub/blob/master/doc/solver.md

## Unit property testing

Property testing consists in defining invariants, generating random valid inputs
for the tested functions, and verifying that the invariants hold for the
results. Here are some examples extracted from the `range` module.

```rust
proptest! {
    #[test]
    fn negate_is_different(range in strategy()) {
        assert_ne!(range.negate(), range);
    }

    #[test]
    fn double_negate_is_identity(range in strategy()) {
        assert_eq!(range.negate().negate(), range);
    }

    #[test]
    fn negate_contains_opposite(range in strategy(), version in version_strat()) {
        assert_ne!(range.contains(&version), range.negate().contains(&version));
    }
}
```

As you can see, the input of the testing function is specified in an unusual
manner, using a function called a "strategy". This is the terminology used by
the [proptest crate][proptest] and it simply is a way to describe how are
randomly generated the values used as inputs to those property tests. Don't
hesitate to have a look at the corresponding `strategy()` function defined just
above the extracted code if you want to know more about that.

[proptest]:
  https://altsysrq.github.io/rustdoc/proptest/latest/proptest/index.html

## End-to-end property testing

Defining and testing properties for small units of code like ranges
implementation is rather easy. Coming up with interesting properties for
end-to-end testing such as results of full resolution is different. But the most
difficult part is probably finding a "strategy" to generate random but valid
registries of packages and dependencies. This is the work that has been done in
`tests/proptest.rs` for the `registry_strategy()` function.

The simplest implementation of `registry_strategy()` would be
`any::<Map<P, Map<V, Map<P, Range<V>>>>()`, with some machinery to convert to an
`OfflineDependencyProvider`. While this works, it would almost certainly
generate unsolvable dependencies. Indeed random package names in dependencies
have an almost null probability of being the same than existing package names.

Let's defer picking the dependencies until we have a list of available names.
Luckily proptest has an [Index] type for this use case of picking something out
of a list that has not yet been generated. This leaves us with
`any::Map<P, Map<V, Map<Index, Range<V>>>()`, with more machinery to get the `P`
for each index. While this works, the generated versions have very low
probability of being the same than existing versions for a given package.

[Index]: https://docs.rs/proptest/0.10/proptest/sample/struct.Index.html

Let's also use [Index] to ensure our `Range`s are relevant. To identify each
[Index] we will name `Ip` the one for package names, and `Ia` and `Ib` the ones
to define version ranges. This leaves us with
`any::Map<P, Map<V, Map<Ip, (Ia, Ib)>>()`. The conversion from
`Map<Ip, (Ia, Ib)>` to `Map<P, Range<V>>` is done by first picking a dependency
`P` using `Ip` and then picking up two versions for that package with the `Ia`
and `Ib` indices. We can then call
`Range::between(versions[Ia], versions[Ib].bump())` to build a range. If
`versions[Ib] < versions[Ia]`, we use
`Range::between(versions[Ib], versions[Ia].bump())` instead.

Now we finally have something that makes interesting registries! But not
particularly realistic ones since almost all of them end up with packages that
indirectly depend on themselves. PubGrub is designed to solve dependencies with
at most one version per package, so that kind of circular dependencies will very
often end up unsolvable.

<!-- One problem solved only to make a new one, -->
<!-- the [pigeonhole principle](https://en.wikipedia.org/wiki/Pigeonhole_principle) strikes again! -->

To fix this lets make sure that we have a DAG, by having each package only
depend on packages with a lower index. One solution is to represent dependencies
in a side list `Vec<(Ip, Id, (Ia, Ib))>`. If `Ip < Id` then we switch them to
maintain the acyclic nature of the graph. This is currently how we generate
registries of dependencies, suggestions to improve are welcome!

Generating random registries of packages may produce cases where dependency
resolution would take too long. For this reason, we introduced in the
`DependencyProvider` trait definition a function called `should_cancel` which is
called in the solver loop. By default it does nothing but can be overwritten
such as in `TimeoutDependencyProvider` (defined in `tests/proptest.rs`), where
the solver is stopped after a certain amount of time.

Once all this is setup, we have to come up with good properties. Here are some
of these:

- **The solver should return the same result on multiple runs with the same
  input**. That may seem trivial, but thinks like hashmaps do add some
  randomness. So that test ensures that we configured properly everything that
  could prevent reproducibility of the solver.
- **Changing the order in which valid package versions are tried should not
  change the existence of a solution or not**. Indeed, some freedom is available
  for the dependency provider to pick which package and version to choose next.
  We must ensure that it cannot change the existence of solution for our
  implementation of the solver algorithm.
- **Removing a dependency cannot prevent existence of a solution**. If a
  solution was found in a given situation, removing a dependency cannot get us
  in a situation where the solver does not find a solution anymore. Only adding
  dependencies should impact that.
- **Removing a package that does not appear in a solution cannot break that
  solution**. Just as before, it should not impact the existence of a solution.
