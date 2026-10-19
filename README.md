# Universal Annotation Language (UAL) — Specification 0.6-draft

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

`ONCHAIN` accepts `[kind: script]` (the default) or `[kind: function]`.
Both describe actual compiled UPLC. A script is a ledger entry point in
`plutus.json`; a function is a separately compiled library entry point stored
only in the assurance document's `functions` registry. Function specifications
remain properties, with `scope.functions`; mixed properties may have both
`scope.validators` and `scope.functions`. Ids must be unique across both kinds.

```haskell
{-@ ONCHAIN [kind: function] [version: PlutusV3] [steps: 100, semantics: E]
    outbids :: { Integer : asNative } -> { Integer : asNative } -> { Bool : asNative }
@-}
{-@ PROPERTY [scope: outbids] outbids_exact
    "The compiled helper returns the comparison result."
  : ∀ (old proposed : Integer), outbids_returns old proposed (decide (old < proposed))
@-}
```

The function result has an encoding just as its arguments do. No ledger-purpose
or datum/redeemer/context convention applies to function arguments. The initial
producer supports native Integer, BuiltinByteString, BuiltinString, Bool and
BuiltinUnit, and Data schemas referencing a finite assurance `definitions` registry. The checker
keeps Data function inputs/results as raw `Data`, including when their schemas
describe narrower domains; properties must state those domain premises. Native
inputs/results use their native Lean types. `<id>` returns the CEK state;
`<id>_returns ... expected` means the machine halted with precisely the expected
constant in the declared result encoding. It does not equate an error or step
exhaustion with a return value. Schema constraints are descriptions, not a proof
that every execution inhabits the described domain.

Every function annotation must have one `compiledFunction id compiledCode`
artifact supplied to `writeInterfaceBundleWithFunctions environment preamble
defaultId modules blueprint functions`. Missing or extra artifacts are errors.
The artifact is built from Plinth `CompiledCode`, serialized as single-CBOR Flat,
and SHA-256 is computed over decoded compiledCode bytes. `writeInterfaceBundle`
remains the script-only convenience entry point. No function code is copied to
the blueprint. A function checking target contains `function`, `functionHash`
and `functionInterface`:

```json
{
  "function": "outbids",
  "functionHash": { "alg": "sha256", "digest": "<SHA-256 of compiled bytes>" },
  "functionInterface": {
    "serialization": "cbor-flat",
    "plutusVersion": "v3",
    "arguments": [{ "dataType": "#integer" }, { "dataType": "#integer" }],
    "result": { "dataType": "#boolean" }
  }
}
```

The digest above is a placeholder. The interface must exactly match the function
registry entry. The context's execution settings apply to the function too;
both the context reference and the function digest are checked.

A separately compiled helper property establishes behavior of those exact
helper bytes. Sharing a Haskell binding with a validator does not by itself
prove that an optimized/inlined occurrence in the validator is equivalent.
Validator-level claims must still check the validator bytes, or provide a
separate linking/equivalence argument.


```haskell
{-@ ONCHAIN [version: PlutusV3] [steps: 2500, semantics: E]
    vestingValidator :: { VestingParams : asData }
                     -> { BuiltinData : asData }
                     -> BuiltinUnit
@-}
```

For a script, the name designates its stable blueprint validator `id`; for a function, it is the assurance registry key.
It MUST be unique across all modules supplied to one build. The ordered argument
list describes the *unapplied compiled program*: it includes parameters and
runtime inputs, including script context. The result type is not an argument.
The producer MUST resolve argument schemas and check the binding's arity.
A `BuiltinData` surface-language argument may be refined to a named data schema
for compile-time parameters when the function decodes that argument itself.
Runtime script ONCHAIN binders are raw `BuiltinData`: narrowed datum/redeemer/context
domains must appear explicitly as premises in the proposition. Arity checking does not establish
that this refinement agrees with the implementation.

A bare argument or function-result type means `asData`. Script parameter wrappers apply `IsData.toData`, then wrap the result as a UPLC Data constant. Raw runtime script inputs and Data function interfaces receive raw `Data` directly; their wire encoding is the identity.
`asNative` requires a native schema (`#integer`, `#bytes`, `#unit`, `#boolean`,
`#string`, `#list`, or `#pair`) and passes the corresponding UPLC constant.
`asScott` denotes a distinct encoding; consumers without that encoder MUST reject
verification of targets requiring it. They MUST NOT substitute Data encoding.
The initial Plinth implementation resolves plain type names and lists of them;
unsupported type syntax is rejected.

The blueprint uses the proposed compiled-interface dialect:
`https://cips.cardano.org/cips/cip57/extensions/compiled-interface/v1/schema.json`.
Its required vocabulary is the same base URI ending in `vocabulary`. This is an
unpublished draft for CIP-editor review. Legacy CIP-57 is not silently reinterpreted.

Each validator's `interface` declares a ledger calling convention and purpose-specific
ordered invocations. Parameter arguments refer to `/parameters/i`; their schemas
are the single wire-encoding authority. Data `integer` and native `#integer` remain
distinct, including through definitions. Annotation encodings must agree with those
schemas. V1/V2 spending suffixes are datum, redeemer, context; other V1/V2 purposes
use redeemer, context. V3 uses context only. All unapplied parameters precede these
runtime arguments exactly once. The current producer requires explicit purposes
and the checking bundle selects one purpose per target.

Scott schemas require a versioned `#scott` profile. The initial executable producer
and consumer support guarded recursive Data schemas, and reject Scott schemas. They must not claim a
compiler implements the proposed Scott convention merely because it uses the name.
The legacy validator-level `arguments` and `budget` keys are forbidden in this dialect.

### Execution settings

`[steps: n, semantics: E]` emits a separate checking-context artifact whose
`execution` is `{"semanticsVariant":"E","budget":{"kind":"cek-steps","steps":n},"acceptance":"evaluation"}`.
Each property's `checkingContext` selects a digest-bound registry entry in the
assurance document. Context and environment hashes cover exact raw file bytes.
Artifact URIs resolve relative to the containing document. Execution settings do
not belong in the blueprint. Each property gets wrappers using its own context;
a wrapper from another context must never be reused implicitly.
The semantics variant MUST be one of A, B, C, D, E and is passed explicitly to the
Plutus CEK interpreter. It is not inferred from the installed library's default.
A script's language version MUST agree with the blueprint preamble. Each function declares its own language version in assurance.
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
The initial Blaster 0.6-draft checking profile supports explicit step budgets only.
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

The 0.6-draft blueprint profile elaborates a property's fragment closure in the
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

Each scoped id denotes the wrapper around that script or function's exact compiled UPLC.
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
   schema definitions and complete `interface` invocations. Store compiled functions and their input/result schemas only in assurance. Preserve Data/native encodings
   in the parameter schemas. Export execution settings into checking-context artifacts.
3. Write the final blueprint bytes before hashing them into `assurance.blueprint.hash`.
4. Emit a detached assurance document with language name, version `0.6-draft`, the `assurance-v2.json` schema, and URI
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
UAL 0.6-draft requires the new blueprint dialect, stable ids, and digest-bound
contexts using `https://cips.cardano.org/cips/cipXXXX/profiles/uplc-step-check/v1`.
The initial implementation supports Data/native parameters, raw Data runtime inputs,
universal and fully applied parameter modes, and explicit CEK step/semantics settings
with evaluation acceptance. Scott and ledger-unit checking are explicit errors. Universal mode requires the proposition to quantify over all
parameters and include any required domain premises; metadata is not a theorem.

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

The local consumer's `#write_checking_environment "environment.json"` command
captures `blaster-loaded-environment-v1`: hashes of the Lean executable, the Z3
executable selected from PATH, and every imported `.olean`, plus Lean version and
solver timeout, heartbeat and recursion settings. If the driver loads a native
Blaster library, it supplies the same path through `ASSURANCE_NATIVE_LIBRARY`
and `--load-dynlib`; the manifest additionally hashes that library. A check recomputes this manifest and requires exact equality.
Thus stale compiled modules or a different ledger model cannot inherit a source-only
version label. Generate this manifest with the same imports as the checking driver;
ledger-dependent claims must import CardanoLedgerApi before capturing it. This is
an executable-input binding, not a hermetic OS/container image or source-build proof.

The reproducibility artifact must additionally record the Lean toolchain, Blaster/PlutusCoreBlaster/
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
assurance document with `smt-check` evidence, the selected `checkingContextHash`,
and a digest-bound run artifact. Changing a context makes old evidence stale even
when the template hash is unchanged.
Neither is a signed attestation or reconstructed kernel proof.

## 5. Producer API and legacy compatibility

Plinth's `writeInterfaceBundle environment preamble defaultId modules blueprint`
validates the interface and writes `plutus.json`, `assurance.json`, and one
`checking-<property>.json` file per claim. Supply the previously captured environment
manifest. `Example/Ual/Game/Main.hs` accepts its path as its sole CLI argument.
`interfaceBlueprint` provides the pure interface-export step for other producers.
The bundle emits no evidence. All targets in a single property must share its
explicit step/semantics settings; multi-purpose target selection is not yet exposed
by this initial producer API.

The [0.5 specification](./SPEC-0.5.md), original `attachUal`/`writeAssurance` APIs and
legacy fixtures are retained for reproducibility. They emit the old prototype
format. Consumers must select a profile by its explicit schema/language version;
they must not silently upgrade old evidence or treat new contexts as optional hints.

`Example/Ual/Parameters/Main.hs` adds a compiled Data/native parameter comparison.
The corresponding `run_parameters.py` checks both properties, then swaps the
native parameter schema to Data without changing code and requires falsification.

## Detached specifications and the Auction example

Annotations may live in `FormalSpec.hs`. Import the implementation bindings and
the types named by ONCHAIN into that module, enable `TemplateHaskell` and
`TypeApplications`, then define `annotations = $(ualModule)` there. The splice
reads that module's own source and resolves imported names. The bundle generator
imports its `annotations`; there is no required file name. Use an unqualified
binding name as the stable target id. Local bindings still need the documented
TH declaration-group boundary; imported bindings do not.

The Plutus example `Example/Ual/Auction` follows this layout: `OnChain.hs`
applies the quick-start auction parameters, `FormalSpec.hs` contains all
annotations, and `Main.hs` compiles the validator and `outbids` helper. It adapts
bid and payout scenarios from CardanoLedgerApiBlaster's `main`
branch at commit `9938562fd452351655fe2f6b63e583c62422687c` through the library's
`CardanoLedgerApi.Examples.Auction` context constructors. It recompiles current
Plutus source; it does not claim byte identity with the upstream CBOR artifact.

`ctx` has the type required at each boundary. Ledger V3 UPLC receives raw Data;
a predicate can accept `CardanoLedgerApi.V3.Contexts.ScriptContext` and pass
`CardanoLedgerApi.IsData.Class.toLedgerData ctx` to the raw generated wrapper.
Auction demonstrates this as a local `typedAuction` predicate. A typed context
need not be a ledger-valid transaction: these are explicitly bounded transaction
shapes with symbolic integer values, not verification of every possible auction
transaction or the upstream vulnerability audit. Raw malformed-context claims
remain expressible alongside the typed scenarios.

Function checking targets also carry `functionInterface`, an exact copy of the
registry entry's `serialization`, `plutusVersion`, ordered `arguments` and
`result`. The checker requires equality. Together with `functionHash`, this
binds the bytes and their wire interpretation into `checkingContextHash`;
changing only an argument encoding cannot reuse the old context or evidence.

### Recursive Data schemas

Recursive function schemas use `#/definitions/<id>` references into the
assurance document's `definitions` registry. Blueprint parameters resolve the
same pointer syntax against blueprint definitions. Definitions are retained as
a finite graph, never truncated to a fixed expansion depth. The checker accepts
self and mutual recursion through constructor fields or container elements;
reference-only cycles and native values nested in Data are rejected.

Each function checking interface copies the assurance `definitions` registry
alongside its arguments and result; omitted definitions mean an empty registry.
The checker requires exact equality, so changing a referenced definition changes
the context and invalidates old evidence. Recursive Data arguments/results stay
raw `Data`; this includes malformed values. Schema declarations alone do not
establish refinements or termination.

The `Example/Ual/Recursive` example compiles a tree-sum validator and a tree-mirror
helper. Its detached specification checks symbolic values at nested tree shapes
and rejects malformed input. A concrete accepted tree establishes inhabitation.
For generic recursive Haskell types, use `deriveRecursiveDefinitions`, which
carries a visited set across self and mutually recursive types. The existing
`deriveDefinitions` API retains its behavior for custom `Unroll` instances;
opaque custom types can supply an explicit finite closure with `definitionsFor`. The checker retains arbitrary finite Data depth,
but the CEK budget still bounds execution.

## Applied deployments

`ONCHAIN` describes the template's complete argument list. A property's checking
context may bind every parameter with `mode: applied`; the formal statement then
calls the generated wrapper with only its remaining runtime arguments. The
Plutus API `writeInterfaceBundleWithBindings` selects bindings by property and
validator id. `compiledParameter` supplies Flat value artifacts and
`appliedParameters` constructs the unoptimized template application.

The context binds each term, the specialized program artifact and its script
hash. The checker validates wire representations, parameter order, complete
coverage, artifact digests and exact program structure before checking the
proposition. Unsupported schema refinements fail explicitly. This is direct
verification of the specialized program; it does not automatically instantiate a
previous universal theorem or trust historical evidence.
