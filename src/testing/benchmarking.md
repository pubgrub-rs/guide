# Benchmarking

If you are interested in performance optimization for pubgrub, we suggest
reading [The Rust Performance Book][perf-book].
[Microbenchmarks generally do not represent real world performance](https://youtu.be/eh3VME3opnE&t=315),
so it's best to start with a slow case in [uv](https://github.com/astral-sh/uv)
or [cargo](https://github.com/Eh2406/pubgrub-crates-benchmark) and look at a
profile, e.g. with [samply](https://github.com/mstange/samply).

[perf-book]: https://nnethercote.github.io/perf-book/

A first step is optimizing IO/network requests, caching and type size in the
downstream code, which is usually the bottleneck over the pubgrub algorithm.
Using `Arc` and splitting types into large and small variants often helps a lot.
The next step is to optimize prioritization, to reduce the amount of work that
pubgrub has to do, here pubgrub should give better information to make this
easier for users. These usually need to be done before pubgrub itself becomes a
bottleneck.

## Side note about code layout and performance

I highly recommend watching the talk ["Performance Matters" by Emery
Berger][perf-talk] presented at strangeloop 2019 for more information on "sound
performance analysis" and "effective performance profiling". There is an [issue
in criterion][criterion-stabilizer] about the "stabilizer" tool. And for causal
profiling, it seems possible to directly use [coz][coz] for rust programs.

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/r-TLSBdHe1A?start=656" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

[perf-talk]: https://youtu.be/r-TLSBdHe1A?t=656
[criterion-stabilizer]: https://github.com/bheisler/criterion.rs/issues/334
[coz]: https://github.com/plasma-umass/coz/tree/master/rust
