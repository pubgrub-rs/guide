# PubGrub guide

Source for the PubGrub guide. The published version is available at
https://pubgrub-rs-guide.pages.dev. This guide is made with [mdBook][mdbook]. To
compile it locally, install the same mdBook version used in CI and run

```sh
cargo install mdbook --version 0.5.4 --locked
mdbook serve
```

We use prettier for consistent formatting. The easiest way to run it is through
npx:

```sh
npx prettier --write --print-width 80 --prose-wrap always "**/*.md"
npx prettier --write "**/*.yml"
```

[mdbook]: https://github.com/rust-lang/mdBook
