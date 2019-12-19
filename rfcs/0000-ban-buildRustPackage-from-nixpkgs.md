---
feature: ban_buildRustPackage_from_nixpkgs
start-date: 2019-12-19
author: Alyssa Ross
shepherd-team: (names, to be nominated and accepted by RFC steering committee)
shepherd-leader: (name to be appointed by RFC steering committee)
related-issues: (will contain links to implementation PRs)
---

# Summary
[summary]: #summary

Ban `rustPlatform.buildRustPackage` from being used in Nixpkgs, in
favour of `crate2nix`.

# Motivation
[motivation]: #motivation

## `rustPlatform.buildRustPackage`

`rustPlatform.buildRustPackage` is a function that vendors dependencies
for a Rust program, and then builds it and its dependencies all at
once in a single derivation.  It has several drawbacks that make it
unsuitable for use in Nixpkgs:

### `cargoSha256`

The vendoring is done using `cargo vendor` (previously an external
program, now part of Cargo).  This reads the Cargo.toml and Cargo.lock
for a Rust package, and prefetches all of the dependencies into a
`vendor` directory.  Because this requires network access, Nix requires
it to be a fixed-output derivation -- the expected outputHash has to
be specified.  `buildRustPackage` gets this from the mandatory
`cargoSha256` argument.
  
However, this is not a good use of a fixed-output derivation.  Unlike,
say, a single file downloaded from the internet, which is generated
once and has the same hash for the rest of time, what `cargo vendor`
does exactly is not clearly defined.  This means that a new version of
Cargo can change the `cargoSha256` of every package built with
`buildRustPackage`.  We are aware of at least [two][cargoSha256-change-1] [occasions][cargoSha256-change-2] where this
has happened.
  
It is difficult to notice when this has even happened, because Hydra
will cache the version with the old hash indefinitely, and continue
serving it every time somebody builds the Rust package.  This means
that each package is implicitly depending on the cached behaviour of a
previous version of Cargo no longer in the build closure, which is the
sort of impurity Nix is designed to avoid.
  
Because it's not easy to notice when `cargoSha256` has broken, users who
_do_ run into it (because they're overriding the package, not using
Hydra as a substituter, etc.) have to try to figure out why the
package works for everybody else and not for them, and experience
Nixpkgs should be trying to eliminate.
  
Additionally, having checksums that unexpectedly break devalues the
security benefit of checksums.  If we become used to assuming
something in Nixpkgs changed and update checksums every time they
break without asking why, it would be easy for a Nixpkgs dependency to
be replaced with malware.  Homebrew, another package manager, [has
fallen victim to this exact attack][homebrew-handbrake].

We could avoid having to have something like `cargoSha256` if we instead
used fixed-output derivations only for the sources of each dependency,
like we do elsewhere in Nixpkgs.

[cargoSha256-change-1]: https://github.com/NixOS/nixpkgs/issues/60668
[cargoSha256-change-2]: https://github.com/NixOS/nixpkgs/pull/76060
[homebrew-handbrake]: https://github.com/Homebrew/homebrew-cask/issues/33536

### No dependency information

Because `buildRustPackage` fetches all of its dependencies into a single
blob that is opaque at evaluation time, there's no way to specify
which non-Rust dependencies a Rust library has in a
`buildRustPackage`-compatible way.  This means that dependencies like
openssl need to be specified for every Rust program that has uses an
OpenSSL library, rather than just once for the library.

### Build times

`buildRustPackage` invokes `cargo build` once in total, to build a
package and all of its dependencies.  This means that, even if two
packages have substantially similar dependencies (not uncommon), the
whole Rust dependency graph has to be built from scratch for each
package.  This makes developing and testing Rust packages extremely
time consuming and frustrating.

## crate2nix

crate2nix is a program that can generate Nix expressions for Rust
programs.  If we were to use it instead of `buildRustPackage`, we would
never get broken checksums because Rust was updated, we wouldn't have
to non-Rust dependencies of Rust libraries for each Rust program that
used one of those libraries, and build times would be much lower on
average because each library would be build in a seperate derivation.

# Detailed design
[design]: #detailed-design

- The Nixpkgs manual should be updated to instruct contributors to use
  crate2nix instead of `buildRustPackage`.
  
- We should _immediately_ stop committing new uses of `buildRustPackage`
  into Nixpkgs, and encourage people to redo their expressions with
  crate2nix.
  
- crate2nix should be packaged in Nixpkgs as soon as possible, rather
  than requiring installation from its own repository.

- All uses of `buildRustPackage` in Nixpkgs should be rewritten to use
  crate2nix.  If they don't currently build with `buildRustPackage`,
  they should be dropped.  If they do, but don't with crate2nix, they
  can stay until crate2nix can handle them.

# Drawbacks
[drawbacks]: #drawbacks

- The Nix files generated by crate2nix can be a few thousand lines
  long, and need to be stored in Git.  `buildRustPackage` does not
  require any dependency information to be stored in Git at all.

- People already know how to use `buildRustPackage`, and will have to
  learn how to use crate2nix.

# Alternatives
[alternatives]: #alternatives

## Don't do anything

We'll continue having all the problems outlined above.  Every few
months, somebody will discover that they're getting a different
checksum for some reason, fix it for one package, and move on, and
then a few days later the same thing will happen to somebody else on
another package.  Nixpkgs contributors will become used to checksums
breaking, and we will be more vulnerable to attacks like the one that
happened to Homebrew.

## Recommend Carnix instead of crate2nix

Carnix is another program that does something similar to crate2nix.
It is much more difficult to use, less feature complete, and has no
documentation.

## Manually package Rust libraries

This could likely be done with a modified stdenv in the same way it is
done for C programs.  This might be worth exploring in future, but
would not be nearly as easy to switch to, so it would be much more
difficult for us to make progress in addressing the problems caused by
buildRustPackage.

# Unresolved questions
[unresolved]: #unresolved-questions

Should expressions for Rust programs be automatically generated at
all, or should Rust programs and their dependencies be manually
packaged like C/Perl/Python?  One reason we might want this is that it
would mean that we didn't end up packaging several different patch
versions of the same library because that's what the lockfiles say.
This is out of scope for this RFC, because it would be a much bigger
change.

# Future work
[future]: #future-work

- We should consider removing `buildRustPackage` entirely.  The same
  things that make it unsuitable for Nixpkgs likely make it unsuitable
  for other uses too.

- We should come up with a set of guidelines for when it is reasonable
  to use a fixed-output derivation, to avoid the mistakes of
  `buildRustPackage` being repeated.  We should evaluate other uses in
  Nixpkgs (e.g. `buildGoPackage`), and decide if should remain or face
  similar removal procedures.
