- Feature Name: `cargo_cli_rustflag`
- Start Date: 2022-08-18
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

One paragraph explanation of the feature.

This RFC aims to improve the experience of enabling Rust compiler flags for specific crates when building Rust projects
by adding a new option, `--rustflag=<RUSTFLAG>`. This would have the same effect as `cargo rustc -- <RUSTFLAG>` but would
also be available for use by other subcommands such as `bench, build, check, run` and `test` thus allowing a Rust project
to be built and tested for instance without forcing a new compilation and losing the rustflags that were set when invoking
`cargo rustc`.

# Motivation
[motivation]: #motivation

Today, there currently exists multiple ways of specifying `RUSTFLAGS` to pass to invocations of the Rust compiler.
All of the existing ways have the limitation of not being able to specify which invocation of rustc the compiler flag
is set for. When a Rust developer tries to enable a Rust compiler option for the current crate being built, they will also
have this compiler option set for all dependencies including the standard libraries. With the feature proposed by this RFC,
a Rust developer can simply use Cargo CLI to pass rustflags for a given crate without the need to worry about how other crates
might be affected.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This RFC proposes adding a new cargo flag, `--rustflag=<RUSTFLAG>`, which would instruct cargo to pass the given flag when invoking rustc
for the current crate being compiled. This allows setting a Rust compiler flag for local crates only and not forcing this upon
dependencies including transitive dependencies and standard libraries.

## An example: code coverage

The Rust compiler currently supports instrumenting Rust built libraries to measure code coverage for a given crate through tests.
In order to instruct rustc to instrument a given crate, a user would need to pass the Rust compiler flag `-Cinstrument-coverage`
to the invocation of rustc when building said crate. This can be done via the `RUSTFLAGS` environment variable but this would have
the side effect of enabling this flag for every crate in the dependency graph including upstream dependencies, transitive dependencies,
as well as the standard libraries. There are a couple of other options for setting Rust compiler flags but most of them have the
same issue as using the `RUSTFLAGS` environment variable.

Another way of setting a rustc flag for a specific crate is through the `cargo rustc` subcommand. A Rust compiler flag can be passed
to the invocation of rustc by cargo for the current crate being built by setting it as an argument directly to the rustc compiler.
For example:

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

cargo CLI (Output lines have been removed for simplicity):
```
cargo test --rustflag=-Cinstrument-coverage

Compiling foo v0.1.0 (...)
     Running `rustc --crate-name foo --edition=2021 src\lib.rs --crate-type lib ... -Cinstrument-coverage`
     Running `rustc --crate-name foo --edition=2021 src\lib.rs --test ... -Cinstrument-coverage`
    Finished test [unoptimized + debuginfo] target(s) in 1.35s
     Running `target/debug/deps/foo-669448d9b4043564`

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Running this command will build the crate `foo` with the flag `-Cinstrument-coverage` passed to the invocation of rustc
for the crate `foo`. Dependencies such as the standard libraries and other upstream dependencies would not be instrumented
saving on compilation time.

## --rustflag for a workspace

When building a workspace with multiple members, any `--rustflag=<RUSTFLAG>` options set will be passed to the invocation
of the Rust compiler for all members unless `workspace.default-members` manifest key has been set. In that case, only the default
members being compiled will have the rustflag options specified passed through to rustc. For example:

Cargo.toml:
```toml
[workspace]
members = ["foo", "bar"]
```

cargo CLI (Output lines have been removed for simplicity):
```
cargo build --rustflag=-Cinstrument-coverage

Compiling foo v0.1.0 (.../foo/foo)
Compiling bar v0.1.0 (.../foo/bar)
    Running `rustc --crate-name foo ... -Cinstrument-coverage`
    Running `rustc --crate-name bar ... -Cinstrument-coverage`
Finished dev [unoptimized + debuginfo] target(s) in 0.98s
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

As mentioned above a new Cargo option, `--rustflag=<RUSTFLAG>`, would be added to several of the existing cargo subcommands.
Those subcommands would be, `bench, build, check, run` and `test`. The `--rustflag=<RUSTFLAG>` option will require an `=`
sign between the option name and the option value, due to the way Cargo parses the options passed to its subcommands. Cargo under
the covers uses the `clap` crate to parse the command line invocation and set the relevant options passed to it. Since all rust
compiler flags start with a `-`, without the `=` delimitter `clap` parses the rustc flag as a new Cargo flag instead leading to
an error from Cargo.

For each rustc flag specified by a Rust developer, Cargo will pass the flag through to the invocation of rustc for the current crate.
This includes all invocations of rustc for a given crate including all targets, such as, lib, bin, examples and the test targets
being built. For example, a given crate `foo` that contains a lib, bin, examples and test target:

Cargo.toml:
```toml
[package]
name = "foo"
version = "0.1.0
```

cargo CLI (Output lines have been removed for simplicity):
```
cargo test --rustflag=-Cinstrument-coverage

Compiling foo v0.1.0 (D:\projects\foo)
    Running `rustc --crate-name foo --edition=2021 src\lib.rs ... --crate-type lib -Cinstrument-coverage`
    Running `rustc --crate-name foo --edition=2021 src\lib.rs --test ... -Cinstrument-coverage`
    Running `rustc --crate-name bin1 --edition=2021 src\bin\bin1.rs ... -Cinstrument-coverage`
    Running `rustc --crate-name example --edition=2021 examples\example.rs... --crate-type bin -Cinstrument-coverage`
    Running `rustc --crate-name foo --edition=2021 src\main.rs --test ... -Cinstrument-coverage`
Finished test [unoptimized + debuginfo] target(s) in 1.51s
    Running `.../debug/deps/foo-669448d9b4043564`

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

    Running `.../target/debug/deps\bin1-88f2f72473b4679f`

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

    Running `.../target/debug/deps/foo-53f9cd70087d575a`

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

Doc-tests foo
    Running `rustdoc --edition=2021 --crate-type lib --crate-name foo --test ...`

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

In the example above, the only invocation which does not include the user specified rustc flags is the invocation of
`rustdoc`. Since `rustdoc` flags are treated different than normal `rustflags`, the flags specifed through `--rustflag=<RUSTFLAG>`
will not be passed to the invocation of `rustdoc`.

## Build scripts

The new `--rustflag=<RUSTFLAG>` feature will not be passed to the invocation of rustc for build scripts that are being compiled and run
on the host. This currently out of scope for this RFC since rustc flags are treated differently for build scripts depending on cargo
configuration settings as well as the target specified.

## Integration with existing RUSTFLAGS

In Cargo there exists numerous ways to specify which Rust compiler flags should be set when compiling a crate and its dependencies.

1. `CARGO_ENCODED_RUSTFLAGS` environment variable
2. `RUSTFLAGS` environment variable
3. `target.*.rustflags` from the config (.cargo/config)
4. `target.cfg(..).rustflags` from the config (.cargo/config)
5. `host.*.rustflags` from the config (.cargo/config) if compiling a host artifact or without `--target`
6. `build.rustflags` from the config (.cargo/config)

The `--rustflag` values would override the set of rustflags calculated from the options listed above only for the current crate or the set of crates in the workspace. All upstream and transitive dependencies will still use the rustflags calculated from the environment
variables and cargo config.

As of today, the `profile.rustflags` manifest key is appended to the set of rustflags calculated from the environment variables and
cargo config settings. With the addition of the `--rustflag=<RUSTFLAG>` feature, the `profile.rustflags` compiler options will work in
the same manner and be appended to the invocation of `rustc`. The user specified command line rustflags will not override the values of
the `profile.rustflags` values.

# Drawbacks
[drawbacks]: #drawbacks

There currently exists multiple ways of setting Rust compiler flags when building a Rust project with Cargo. As we mentioned
earlier, there about 7 different ways that already exist today and this RFC is proposing to add yet another option. This could
lead to confusion about the best way to set Rust compiler flags in the community.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Rationale

This design provides a simple mechanism to specify the set of Rust compiler flags
a given crate should be built with. This design also has the benefit of not forcing
all crates in the dependency graph, including upstream dependencies and transitive
dependencies, to be built in the same manner. As with the given example above, setting
the rustc flag `-C instrument-coverage` forces the compiler to do an extra amount
of work to instrument all of the libraries in a given crate. If this flag was passed
for all transitive dependencies, that would only add to the amount of work that needs
to be done by the compiler. With this new feature, the rustc flags set via the `--rustflag`
cargo option would only affect the root crate or the members of the current workspace.

## Alternatives

### Alternative 1: existing RUSTFLAGS manifest keys

### Alternative 2: new build.<crate_name>.rustflags manifest key

# Prior art
[prior-art]: #prior-art

### Rustflags manifest keys

### cargo rustc subcommand

# Unresolved questions
[unresolved-questions]: #unresolved-questions

1. Should the cargo feature `--rustflag=<RUSTFLAG>` be dependent on the existing unstable cargo feature `-Ztarget-applies-to-host` to determine whether or not the rustc flags specified by the user on the cargo CLI should be passed to the invocation of rustc for build scripts defined in the given crate?

# Future possibilities
[future-possibilities]: #future-possibilities

## Crate specific --rustflag

A natural extension to this feature would be to add support for specifying specific crates a Rust compiler flag should be enabled for.
For example, if in a given workspace there exists 2 default members, `foo` and `bar`, having the ability to set rustc flags only for the
crate `foo` and not for the crate `bar`. An example of said feature would be:

cargo CLI (Output lines have been removed for simplicity):
```
cargo build --rustflag=foo:-Cstrip=debuginfo
```

This would result in the `foo` crate stripping away all debuginfo and not included in the generated binary and/or `PDB`. This would not have the same effect on the `bar` crate or any other upstream dependencies.

## --rustflag support for dependencies

Allowing a Rust developer to manually specify which rustflags are passed to upstream dependencies seems like a natural extension of this
feature. As with the above mentioned future possibilities, `--rustflag=<crate_name>:<RUSTFLAG>`, would be sufficient for adding support
for specifying rustc flags for upstream dependency. If a crate is selected which does not exist, or which has not been pulled in as a
dependency, then an warning would be raised notifying the user that the specified rustc flag was unused. A simple use case for this would
be allowing the instrumentation of targeted upstream dependencies or local dependencies through the use of the `-C instrument-coverage`
rustc feature.

## --rustflag support for build scripts

The feature proposed by this RFC does not extend any support to passing Rust compiler flags specified by the `--rustflag` feature to
invocations of rustc when compiling build scripts. There is support today for setting rustc flags for build scripts depending on certain
configuration settings such as, whether the host and target triples match, and/or if the unstable `-Ztarget-applies-to-host` flag has
been enabled and the `[host]`/`[build]` sections have the `rustflags` manifest key set.

Allowing a mechanism for setting rustc flags for build scripts via the `--rustflag` cargo feature would extend the flexibility of
setting compiler flags from the cargo CLI.
