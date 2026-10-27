# Universal Annotation Language (UAL) 1.0

UAL describes executable properties of compiled Cardano programs using Lean 4
terms embedded in source-language annotations. Producers emit a CIP-0057 blueprint
and a detached assurance document; consumers check claims against the compiled
program and its declared checking environment.

## Specification

- [UAL 1.0 specification](./UniversalAnnotationLanguage.md): annotations, compiled
  scripts and functions, argument encodings, checking contexts, recursive schemas,
  and applied deployments.
- [Temporal operators for Blaster state machines](./TemporalOperators.md): execution
  traces, previous/next, weak until, and state-machine invariants.

The language version recorded in assurance documents is `1.0`. The corresponding
repository release tag is `v1.0.0`.

This repository publishes the specification. Producer and checker implementations
live in companion repositories and must explicitly support the declared language
version. Publishing the specification does not create verification evidence or
change an implementation's accepted version identifiers.

Earlier revisions are available in the
[release history](https://github.com/input-output-hk/UniversalAnnotationLanguage/releases).

## License

[Apache License 2.0](./LICENSE).
