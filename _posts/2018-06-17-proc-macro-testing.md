---
layout: post
title: "Procedural Macro Testing in Rust"
categories: testing
comments: true
---
As my inaugural post I wanted to share a technique I put together for testing
procedural macros in Rust. This came up as a part of implementing my embedded
lexer generator [Luther].

The root of the problem that I had to solve was that proc-macro crates cannot have
unit tests like most other Rust crates can. They can have integration tests as
described in (among other places) Brandon W Maister's post 
[Debugging Rust's new Custom Derive system].

Integration tests are very useful for testing the code produced by an invocation
of a proc-macro crate, but they can't directly test the proc-macro crate itself.
An integration test can indirectly test the success paths through a proc-macro
crate since the test cannot run if it cannot compile, and it cannot compile
if the proc-macro crate does not run successfully. What they can't test, even
indirectly, is the failure cases in the proc-macro crate. Integration tests also
cannot allow for test coverage reporting (i.e. on [Coveralls]).

[Coveralls]: https://coveralls.io/
[Debugging Rust's new Custom Derive system]: https://quodlibetor.github.io/posts/debugging-rusts-new-custom-derive-system/
[Luther]: https://crates.io/crates/luther

## Phase Differences
The source of the problem and the starting point for the solution is the phase
difference between proc-macro crates and the testing infrastructure built into
`cargo` through the `cargo test` command. Proc-macro crates are "run" at compile
time while the testing infrastructure is "run" at run time. Crates like [syn]
and [quote] help reduce the dissonance between "compile time" programming and
"run time" programming, but it is always present.

The key insight for me was that when proc-macro crates "run" they run _as a part
of the compiler_. I was always vaguely aware of this, but [syn], [quote] and the
compiler's `proc_macro` crate allowed me to paper over this fact while I was
writing my own proc-macro crate.

The testing technique I put together based on this insight was to run `rustc`
on carefully crafted input files that use my proc-macro crate and then compare
the exit status of `rustc` to the expected exit status to determine if the test
succeeded or failed. (This [appears][COMPILER_TESTS] to be the same strategy that 
the compiler itself uses for tests.)

[COMPILER_TESTS]: https://github.com/rust-lang/rust/blob/1.26.0/src/test/COMPILER_TESTS.md

## Running Rustc
After a bit of experimentation I was finally able to determine a set of options to `rustc`
that produced the results I was looking for. I wanted a command line that would compile my
simple one-file crates such as the following:

```rust
// succ_simple_lib.rs

#![crate_type = "lib"]

extern crate luther;

#[macro_use]
extern crate luther_derive;

#[derive(Lexer, Debug)]
pub enum Token {
    #[luther(regex = "ab")] Ab,
    #[luther(regex = "acc*")] Acc,
    #[luther(regex = "a(bc|de)")] Abcde(String),
}
```

The `luther` crate named above is the runtime crate that (among other things) 
defines the `Lexer` trait while the `luther_derive` crate is the proc-macro
crate that generates the code to implement the trait for (in this case) the
`Token` enum. The `luther_derive` crate is the crate that is run at compile time.

The `rustc` invocation that I found to work for compiling these one-file crates 
is the following:

```shell
$ rustc -L dependency=target/debug/deps
        --extern luther=target/debug/libluther.rlib 
        --extern luther_derive=target/debug/libluther_derive.so
        --out-dir path/to/unique/location
        succ_simple_lib.rs
```

This assumes that you have already run `cargo build` and that it is being run
from the root of workspace that has both the `luther` and `luther_derive` crates
as members.

It is useful to go through the options on this `rustc` invocation one by one.

| Option      | Value             | Comment                                                                  |
|-------------|-------------------|--------------------------------------------------------------------------|
| `-L`        | dependency=...    | The location of the dependencies of the luther and luther_derive crates. |
| `--extern`  | luther=...        | The location of the runtime library crate.                               |
| `--extern`  | luther_derive=... | The location of the compile time proc-macro crate.                       |
| `--out-dir` | path              | The location into which to generate the output artifacts of `rustc`      |

## Putting It All Together
Once I had figured out the `rustc` invocation to use, I was able to put this at
the heart of a test suite runner that invoked `rustc` on each of the one-file
crates in the test suite and compared the exit code of `rustc` with the expected
exit code. The convention that I adopted was that files that started with "succ"
indicated tests that should succeed and test that started with "fail" indicated
test that should fail.

It is even possible to run `rustc` with these options under [kcov] which 
allowed me to collect [test coverage][luther_derive_coverage] information for my
proc-macro crate.

[kcov]: http://simonkagstrom.github.io/kcov/index.html
[luther_derive_coverage]: https://coveralls.io/builds/17406967/source?filename=luther-derive/src/generate.rs

## Conclusion
`cargo` together with the [syn] and [quote] crates provide an means of papering
over the runtime/compile time phase difference between normal Rust library crates
and proc-macro crates. For this problem of directly testing the proc-macro crates,
however, the ability to face the phase difference head on by running `rustc`
directly was essential.

[syn]: https://crates.io/crates/syn
[quote]: https://crates.io/crates/quote
