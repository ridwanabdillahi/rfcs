- Feature Name: `rustflag`
- Start Date: 2022-08-18
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

One paragraph explanation of the feature.

This RFC aims to improve the experience of enabling user specified Rust compiler flags
when invoking a build of a Rust project through Cargo, which most Rust developers use
to build their project. This feature would allow a Rust developer to selectively choose
which rustc flags are set when inoking the Rust compiler for a given crate.

# Motivation
[motivation]: #motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

Running a build of a Rust project via Cargo, there are limitations on trying to set Rust compiler flags. Since Cargo is aware of
all crates being built in the dependency graph, cargo is properly positioned to be able to set Rust compiler flags  

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

This RFC proposes adding a new cargo flag, `--rustflag`, which accepts a pairing of `crate:RUSTFLAG` that instructs cargo to set the
given flag when invoking rustc for the crate specified. This allows setting a Rust compiler flag for local crates as well as upstream
dependencies. Setting a specific Rust compiler flag for the standard libraries is currently out of scope for this RFC.

## An example: code coverage

The Rust compiler currently supports instrumenting Rust built libraries to measure code coverage for a given crate via test runs.
In order to instruct rustc to instrument a given crate, a user would need to pass the `-Cinstrument-coverage` flag to the invocation
of rustc when building said crate. This can be done via the `RUSTFLAGS` environment variable but this would have the side effect of
enabling this flag for every crate in the dependency graph including upstream dependencies, transitive dependencies, as well as the
standard libraries. There are a couple of other options for setting Rust compiler flags but most of them have the same issue as using
the `RUSTFLAGS` environment variable.

Another way of setting a rustc flag for a specific crate is through the `cargo rustc` subcommand. A Rust compiler flag can be passed
directly to rustc by setting it as an argument directly to the rustc compiler. For example:

```
cargo rustc -- -Cinstrument-coverage
```

This example will pass the flag `-Cinstrument-coverage` directly to rustc but only for the current crate. Also running another cargo command
after this will cause a new build of the crate without the flag. For example running `cargo test` will cause the crate to be re-compiled
without the rustc flag and cause tests to be run without first instrumenting any of the libraries.

## --rustflag

The `--rustflag` option will allow a Rust developer to pass any rustc flag to the crate of their choosing. This will allow a simple
command such as `cargo test` to have the option of setting the `-Cinstrument-coverage` flag for a single crate and run the unit tests
ensuring coverage data is collected. For example, let's take crate `foo`:

```
cargo test --rustflags foo:-Cinstrument-coverage
```

Running this command will build the crate `foo` with the flag `-Cinstrument-coverage` passed only to the invocation of rustc for crate `foo`.
Any and all dependencies would not have this flag enabled, forcing coverage data to only be collected for this crate and not for dependencies. This will also run the tests for said crate generating coverage data for each test executed.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation



# Drawbacks
[drawbacks]: #drawbacks

Multiple ways of setting rustflags already, this would add another way and would have to work
with all of the existing ways.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Rationale

This design provides a simple mechanism to specify the set of Rust compiler flags
a given crate should be built with. This design also has teh benefit of not forcing
all crates in the dependency graph, including upstream dependencies and transitive
dependencies, to be built in the same manner. As with the given example above, setting
the rustc flag `-C instrument-coverage` forces the compiler to do an extra amount
of work to instrument all of the libraries in a given crate. If this flag was passed
for all transitive dependencies, that would only add to the amount of work that needs
to be done by the compiler. With this new feature, the rustc flags set via the `--rustflag`
cargo option would only affect the root crate or the members of the current workspace.

## Alternatives

### Alternative 1: existing build.rustflags manifest key

### Alternative 2: existing RUSTFLAGS environment variable

# Prior art
[prior-art]: #prior-art

### Rustflags manifest keys

### cargo rustc subcommand

# Unresolved questions
[unresolved-questions]: #unresolved-questions

# Future possibilities
[future-possibilities]: #future-possibilities

## Crate specific --rustflag

## --rustflag support for dependencies

## --rustflag support for build scripts and proc-macros
