- Feature Name: `cargo_cli_rustflag`
- Start Date: 2022-08-18
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

One paragraph explanation of the feature.

This RFC aims to improve the experience of enabling Rust compiler flags for specific crates when building Rust projects
by adding a new option, `--rustflag=<RUSTFLAG>`. This would have the same effect as `cargo rustc -- <RUSTFLAG>` but would
also be available for use by other subcommands such as `bench, build, check, run` and `test`.

# Motivation
[motivation]: #motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

Today, there currently exists multiple ways of specifying `RUSTFLAGS` to pass to invocations of the Rust compiler.
All of the existing ways have the limitation of not being able to specify which invocation of rustc the compiler flag
is set for.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This RFC proposes adding a new cargo flag, `--rustflag`, which accepts a pairing of `crate:RUSTFLAG` that instructs cargo to set the
given flag when invoking rustc for the crate specified. This allows setting a Rust compiler flag for local crates as well as upstream
dependencies. Setting a specific Rust compiler flag for the standard libraries is currently out of scope for this RFC.

## An example: code coverage

The Rust compiler currently supports instrumenting Rust built libraries to measure code coverage for a given crate through tests.
In order to instruct rustc to instrument a given crate, a user would need to pass the Rust compiler flag `-Cinstrument-coverage`
to the invocation of rustc when building said crate. This can be done via the `RUSTFLAGS` environment variable but this would have
the side effect of enabling this flag for every crate in the dependency graph including upstream dependencies, transitive dependencies,
as well as the standard libraries. There are a couple of other options for setting Rust compiler flags but most of them have the
same issue as using the `RUSTFLAGS` environment variable.

Another way of setting a rustc flag for a specific crate is through the `cargo rustc` subcommand. A Rust compiler flag can be passed
directly to rustc by setting it as an argument directly to the rustc compiler. For example:

```
cargo rustc -- -Cinstrument-coverage
```

This example will pass the flag `-Cinstrument-coverage` directly to rustc but only for the current crate. Running another cargo command
after this will cause a new build of the crate without the flag. For example running `cargo test` will cause the crate to be re-compiled
without the rustc flag `-Cinstrument-coverage` specified. This would cause tests to be run without first instrumenting any of the libraries
thus losing out on collecting any code coverage.

## --rustflag

The `--rustflag` option will allow a Rust developer to pass any rustc flag to the root crate being built. This will allow a simple
command such as `cargo test` to have the option of setting the `-Cinstrument-coverage` flag for a single crate and run the unit tests
ensuring coverage data is collected for this crate and this crate only. For example, let's take crate `foo`:

Cargo.toml:
```toml
[package]
name = "foo"
version = "0.1.0"
```

cargo cli:
```
cargo test --rustflag=-Cinstrument-coverage
```

will result in the following input:
```
Compiling foo v0.1.0 (D:\projects\foo)
     Running `rustc --crate-name foo --edition=2021 src\lib.rs --crate-type lib ... -Cinstrument-coverage`
     Running `rustc --crate-name foo --edition=2021 src\lib.rs --test ... -Cinstrument-coverage`
    Finished test [unoptimized + debuginfo] target(s) in 1.35s
     Running `target/debug/deps/foo-669448d9b4043564`

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Running this command will build the crate `foo` with the flag `-Cinstrument-coverage` passed to the invocation of rustc
for crate `foo`. Dependencies such as the standard libraries and other upstream dependencies would not be instrumented
saving on compilation time.

## --rustflag for a workspace

When building a workspace with multiple members


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

### Alternative 3: new build.<crate_name>.rustflags manifest key

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
