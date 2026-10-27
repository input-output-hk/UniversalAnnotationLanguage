# Universal Annotation Language (UAL) — Specification 1.0

UAL describes executable properties of compiled Cardano programs using Lean 4
terms embedded in surface-language comments. A producer emits a CIP-0057
`plutus.json` and a detached blueprint-assurance `assurance.json`. A consumer
imports the compiled UPLC, reconstructs the declared argument encodings, and
checks each complete proposition. Generation alone provides no verification
evidence.

## 1. Annotation syntax

Plinth annotations are comments delimited by `{-@` and `@-}`. Other surface
languages may supply their own comment delimiters. All examples below use Plinth.
The Plinth producer reads UTF-8 source; annotations under CPP conditionals are
not supported. An unsupported construct MUST cause a diagnostic rather than
silently change a property's meaning.

### ONCHAIN

```haskell
{-@ ONCHAIN [version: PlutusV3] [steps: 2500, semantics: E]
    vestingValidator :: { VestingParams : asData }
                     -> { BuiltinData : asData }
                     -> BuiltinUnit
@-}
```

The name designates the compiled target and its stable blueprint validator `id`.
It MUST be unique across all modules supplied to one build. The ordered argument
list describes the *unapplied compiled program*: it includes parameters and
runtime inputs, including script context. The result type is not an argument.
The producer MUST resolve argument schemas and check the binding's arity.
A `BuiltinData` surface-language argument may be refined to a named data schema
when the function decodes that argument itself. Arity checking does not establish
that this refinement agrees with the implementation.

A bare argument type means `asData`. This encoding applies `IsData.toData`, then
wraps the result as a UPLC Data constant. For raw Data it is the identity encoding.
`asScott` denotes a distinct encoding; consumers without that encoder MUST reject
verification of targets requiring it. They MUST NOT substitute Data encoding.
The initial Plinth implementation resolves plain type names and lists of them;
unsupported type syntax is rejected.

The blueprint carries `arguments`, with `encoding` and `schema` on each entry.
These are optional CIP-57-compatible extension fields. `parameters`, `datum`, and
`redeemer` alone do not establish the compiled program's calling convention.

### Execution settings

`[steps: n, semantics: E]` emits `budget: {"steps": n, "semantics": "E"}`.
The semantics variant MUST be one of A, B, C, D, E and is passed explicitly to the
Plutus CEK interpreter. It is not inferred from the installed library's default.
The producer's language version MUST agree with the blueprint preamble.
Selecting the correct variant for a deployment's language and protocol version
is an explicit responsibility of the checking configuration.

`steps` is a bound on interpreter transitions, not ledger CPU or memory units.
Reaching it leaves an unfinished computation. It MUST NOT count as script
failure, successful execution, or proof of rejection. `isSuccessful` means
normal termination; `isUnsuccessful` means actual script evaluation failure;
`isExhausted` identifies unfinished execution. In particular,
`¬ isSuccessful result` does **not** establish rejection.
A claim about ledger acceptance additionally needs the relevant ledger context,
return-value, cost-model, and protocol conditions.

`[exCPU: n, exMem: m]` remains a distinct ledger-budget declaration. A consumer
without support for it MUST reject that verification request, rather than convert
it to steps or invent a default. Steps and ledger units MUST NOT be mixed.
The initial Blaster 1.0 checking profile supports explicit step budgets only.
Legacy annotations without execution settings remain parseable, but are not
executable under this profile. Duplicate and unknown options are errors.

### UPLC_DATA

```haskell
{-@ UPLC_DATA VestingParams @-}
```

The named type MUST be available to the producer and have the blueprint schemas
needed to generate Lean types and Data encodings. The Plinth producer uses the
existing `HasBlueprintSchema`/`HasBlueprintDefinition` derivations. The annotation
does not itself manufacture a correct serialization instance. Consumers MUST
preserve constructor indices, record order, map/list distinctions, and nested
schema references. Unsupported encodings must be diagnosed.

### PREDICATE

```haskell
{-@ PREDICATE
def ownerAuthorized (owner : PubKeyHash) (ctx : ScriptContext) : Prop :=
  txSignedBy owner ctx.scriptContextTxInfo
@-}
```

A block contains reusable Lean `def` or `abbrev` declarations, including helper
functions returning data as well as Bool or Prop. It introduces no theorem or
assumption. Axioms, admitted proofs, commands, and unchecked replacement
implementations are not predicate definitions. Lean checks types and termination.
Multiple blocks in one module are concatenated in source order.

A producer emits one `formalFragments` entry per module with predicates. Its
`imports` lists the prerequisite fragments. If module M1 imports M2, M1's
fragment depends on M2's fragment, not the reverse. Dependencies through modules
without predicates MUST be preserved. Fragment ids are unique and the dependency
graph MUST be acyclic. Fragment array order has no meaning.

The 1.0 blueprint profile elaborates a property's fragment closure in the
blueprint's generated namespace. Helper names in that closure must be unambiguous;
a producer or consumer MUST diagnose collisions. Qualified/module-isolated
helpers may be introduced by a future profile; silently merging collisions is
not permitted.

### PROPERTY

```haskell
{-@ PROPERTY [scope: vestingValidator] owner_signature_required
    "A transaction that the owner has not signed fails script evaluation."
  : ∀ (p : VestingParams) (ctx : ScriptContext),
      ¬ txSignedBy p.vpOwner ctx.scriptContextTxInfo →
        isUnsuccessful (vestingValidator p (toLedgerData ctx))
@-}
```

A property has a document-unique id (`[A-Za-z0-9_-]+`), a nonempty quoted
natural-language description, and a nonempty Lean term of type Prop.
`[scope: firstValidator secondValidator]` names one or more ONCHAIN ids separated
by whitespace. A single-target producer may omit scope and use its sole target;
multiple targets require an explicit scope. A producer MUST reject unknown ids
and MUST NOT assign unrelated properties to an arbitrary default validator.

The formal body is a proposition, not a theorem declaration or a `:= by ...`
proof script. The initial consumer chooses Blaster. A property may depend on
its module's predicates and imported predicate fragments, even when its own
module declares no predicates. The generated `formal.uses` plus transitive
`imports` MUST provide all those definitions.

Each scoped id denotes the wrapper around that validator's exact compiled UPLC.
A title rename MUST NOT change this binding. Properties must constrain those
programs; assigning `scope` to a proposition about only an abstract model is
insufficient. Consumers SHOULD check dependencies on scoped compiled programs.
Such a check detects omissions, not semantic relevance: an unused reference or
vacuous implication can still mention a program. Specification review remains
necessary.

The body MUST contain the complete quantified proposition, including all premises.
Assurance `assumptions` document conditions already represented in the proposition
or external deployment assumptions. They are NOT silently injected as axioms.
An external assumption is not established by verifying the proposition. Validity
predicates from CardanoLedgerApi describe the selected formal model; the package
must identify the library revision and avoid claiming more ledger coverage than
that model supplies.

## 2. Generated documents and identity

The producer MUST:

1. Reject duplicate module, target, and property ids and unresolved argument schemas.
2. Generate the blueprint with compiled code, recomputed script hashes, stable ids,
   schema definitions, ordered encodings, and explicit execution settings.
3. Write the final blueprint bytes before hashing them into `assurance.blueprint.hash`.
4. Emit a detached assurance document with language name, version `1.0`, and URI
   `https://github.com/input-output-hk/UniversalAnnotationLanguage`.
5. Emit claims with no evidence until verification actually runs; never fabricate
   a `verified` record at compilation time.

A template's hash identifies its unapplied code. A deployment claim additionally
requires applied parameters and the resulting deployed script identity, or a
universal parameter theorem whose premises hold for that deployment.

## 3. Blaster checking profile

A consumer validates the pinned assurance schema and cross-references, verifies
available content digests, recomputes blueprint script hashes, imports the UPLC,
and elaborates propositions against generated types, stable wrappers, and the
explicit fragment closure. Each property is checked independently; definitions
from another property's undeclared fragments MUST NOT leak into its environment.
UAL 1.0 requires stable ids, `asData` arguments, and explicit step/semantics settings
in the initial implementation. Unsupported settings are errors, not fallback
verification.

Every runnable property is checked afresh, including properties whose old evidence
was partial, inconclusive, falsified, or absent. Historical evidence never dictates
the expected fresh verdict. The report distinguishes verified, falsified,
inconclusive, and not checked. A regression-checking command must fail on falsified
or inconclusive runnable claims. Skipped or unsupported properties must be visible.

A fresh check is not authentication of historical evidence. A remote artifact that
was not fetched and digest-checked remains explicitly unverified. Local artifacts
are checked against their declared digest before use. Unknown digest algorithms
must not silently pass. Reproducing an old run requires its pinned environment and
commands, not merely a matching outcome label.

The checking artifact must record the Lean toolchain, Blaster/PlutusCoreBlaster/
CardanoLedgerApiBlaster revisions, solver version and options, interpreter semantics,
and input hashes. Imported ledger namespaces must be supplied by that environment.
Lean source and elaborators are executable code: third-party checking must run in
an OS sandbox with bounded resources. A JSON digest does not make source safe.

Current Blaster SMT verification does not reconstruct a Lean kernel proof. Evidence
and user interfaces must disclose the trusted translation, optimization, evaluator,
and solver components. Kernel-reconstructed evidence is a separate capability.

## 4. End-to-end acceptance criteria

A reproducible example must compile the annotated source, generate both documents,
validate their bindings, and check properties against the generated compiled bytes.
Include both successful execution and actual rejection cases, a falsified property,
and regressions for altered bytes/hashes, unknown scopes, missing fragments, and
step exhaustion. Passing properties about an always-rejecting placeholder alone
is not an adequate demonstration.

The current integration example is Plutus's `Example/Ual/Game/Main.hs`, paired
with PlutusCoreBlaster's `Tests/BlueprintVerify/run_generated.py`. It verifies the
compiled hash-comparison rule from plutus-apps and separately executes a known
hash vector. Cryptographic functions are opaque in the SMT model; a theorem
about hash comparison must not silently assume collision freedom or a concrete
hash result unavailable to that model. The runner emits a local provenance report and, after successful checks, a separate
assurance document with `smt-check` evidence and a digest-bound run artifact.
Neither is a signed attestation or reconstructed kernel proof.
