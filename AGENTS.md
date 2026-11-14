# DodOS system

Verification, before every commit and identical in CI:

    nix develop -c ./verify

This repository holds everything in user space: the WIT contracts
(`contracts/`), and later the runtime, services, drivers, shell, applications,
SDK and image build. The kernel lives in its own repository and is pinned here
by commit.

Design and decisions are in the documentation repository. `contracts/` is the
source of truth for the interfaces.

## Always

- English for code, comments and commits.
- A test comes with the code it guards, and fails without it.
- Update `plan/progress.md` in the documentation repository in the same
  change as the work: the colour of the box, the "Now" table, the date. Moving
  a box is part of "done".
- A component gets only the capabilities it is handed; nothing is ambient.

## Rust style

- Propagate errors with `?`; no hand-written `match` that only returns the
  error.
- One test per behaviour, named after the behaviour.
- An enum instead of a `bool` parameter; no glob import (`use x::*`) outside
  tests; no `Rc` or `Arc` unless ownership is really shared. (These three are
  taken from the Google Rust style guide.)

## Ask first

- Any change to `contracts/`: a contract is re-read by the two humans, and a
  breaking change needs a decision record.
- A new dependency. Code that ends in a Linux driver process must be
  compatible with GPL-2.0 (no Apache-2.0-only crate there).

## Never

- Delete or weaken a test to make `verify` pass.
- Call the kernel directly from a component: only the runtime does.
