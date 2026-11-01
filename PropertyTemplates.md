# Property templates — proposed source extension

This extension adds parameterized property templates to UAL source annotations.
It is not part of the published UAL 1.0 syntax. The producer expands a template
into a complete, ordinary Lean proposition in an assurance document. The lowered
document uses the existing UAL 1.0 checking profile; no template execution or
inference of intent is delegated to the checker.

```haskell
{-@ PROPERTY [scope: auction]
    [template: purpose-isolation spending]
@-}
```

The scope names exactly one `ONCHAIN` script. The template supplies the assertion;
no property id, natural-language statement or Lean body is repeated in the source.
Trailing text, duplicate options, unknown templates and unknown arguments are
errors. The generated id is `<scope>_purpose_isolation`, subject to the existing
property-id rules and duplicate-id checks.

## Purpose isolation

The arguments name a nonempty set of allowed purposes:

```haskell
{-@ PROPERTY [scope: validator]
    [template: purpose-isolation spending minting]
@-}
```

The accepted names are `spending`, `minting`, `rewarding`, `certifying`, `voting`
and `proposing`. Duplicates and a set containing all six purposes are errors.
Order does not affect the proposition. At least one purpose must be excluded.

The allowed set is specified entirely by the template arguments. It MUST NOT be
read from, checked against, or copied into the blueprint's purpose declarations.
Blueprint purposes describe invocations; they are not premises restricting the
contexts quantified by this property. A spending invocation in the checking
context must still permit this claim to execute the same compiled program on a
minting context.

The initial implementation supports Plutus V3 scripts with an explicit version
and a raw `BuiltinData` context argument. Functions and V1/V2 scripts are rejected
for this template. Script parameters are universally quantified using the
compiled wrapper's declared parameter types and encodings. Applied-parameter
bindings for generated properties are not supported in this initial profile.

Let `allowed` be the argument set. The generated claim is:

```text
For every script parameter and every typed V3 ScriptContext ctx:
  if the purpose kind of ctx.scriptContextScriptInfo is outside allowed,
  executing the exact compiled script on those parameters and encode(ctx)
  terminates in an evaluation error.
```

Every payload of the six `ScriptInfo` constructors is covered. The transaction
fields, redeemer and spending datum are universally quantified through `ctx`.
No contract-specific scenario constructor, fixed transaction fixture, ledger
validity premise or allowed-purpose premise is inserted. This is a claim about
typed contexts and compiled evaluation; a counterexample need not describe a
ledger-valid transaction.

The generated Lean source uses the ledger library's V3 `ScriptContext` codec and
the wrapper around the imported compiled bytes. It uses `isUnsuccessful`, meaning
an actual CEK error. A step-limit exhaustion is unfinished execution and does
not satisfy the claim. The property establishes no positive acceptance claim:
a script that always fails satisfies purpose isolation. Demonstrations should
also check successful allowed-purpose execution and replay a counterexample
against a deliberately vulnerable script.

The producer emits the equivalent disjunction `allowedPurpose(ctx) ∨
isUnsuccessful(run(ctx))`. This keeps the compiled call in ordinary proposition
structure for counterexample observation and replay. It does not add an
assumption or restrict the quantified domain.

Proving the claim may be expensive or inconclusive for a script which decodes
unbounded transaction structures before inspecting the purpose. A template
generates an assertion, not a guarantee that the assertion holds or that every
checker can decide it. Reports must preserve those distinctions.

Templates introduce no axioms, proofs or handwritten replacement validator. Their
generated formal source, checking context and compiled program remain subject to
the ordinary assurance digest bindings and fresh verification requirements.
