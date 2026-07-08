# Universal Annotation Language (UAL) -- Design Specification

## 1. Overview and Goals

* UAL is a domain-specific annotation based on the Lean 4 syntax.
It is intended to be used as an annotation language for any Cardano smart contract surface language
(i.e., Aiken, Plinth, Scalus, etc) for specifying both reusable predicate definitions and
properties to be satisfied by the annotated functions or validator script.
* The language serves two primary purposes: (a) expressing machine-checkable properties on
Cardano smart contract functions/scripts, and (b) bridging Lean 4 formalizations with UPLC
(Untyped Plutus Lambda Calculus) execution semantics.
* All annotations are syntactically inert to the Lean 4 elaborator unless explicitly
invoked by UAL macros or tactics; they do not alter the computational meaning of
the annotated function.
* UAL is designed to be composable: predicates and properties defined in one module
can be imported and referenced across module boundaries.

## 2. Annotation Kinds

### 2.1 Predicate Annotation

* A `PREDICATE` annotation section allows the introduction of reusable definitions
that may be required when specifying properties to be proved. Multiple `PREDICATE` sections
can be specified. Each `PREDICATE` section may also reference definitions introduced
in previously defined sections.
* Predicates are first-class UAL citizens: once defined, they can be referenced by name inside
any `PROPERTY` annotation within scope.
* A predicate body must be a well-typed Lean 4 `Prop` or `Bool` expression; it may call any
previously defined UAL predicate or any definition in scope, including those from
the `CardanoLedgerApi` Lean 4 formalization.
* Predicates are declared outside function bodies and are scoped to the namespace in
which they appear. The Lean 4 namespace will be determined according to the
annotated surface language file.
* Predicates may be polymorphic (universe-polymorphic or type-polymorphic) and may accept
implicit or instance arguments, following standard Lean 4 binder conventions.
A predicate annotation does **not** generate a proof obligation by itself; it is purely definitional.

***Example in Plinth***

```
{-@ PREDICATE

def sellerIsPaid' (sellDatum : SellDatum) (amount : Integer) (ctx : ScriptContext) : Bool :=
  Recursor.any out in ctx.scriptContextTxInfo.txInfoOutputs =>
    out.txOutAddress = sellDatum.seller && lovelaceOf out.txOutValue ≥ amount

def sellerIsPaid (input : SpendingInput) : Prop :=
  match IsData.fromData input.datum with
  | some sell => sellerIsPaid' sell sell.price input.ctx
  | none => False

@-}
```

### 2.2 Function Annotation

```
{-@ ONCHAIN [exCPU : <nat>, exMem : <nat>]
    <functionName> : { <Type₁> : <encoding₁> } | <Type₁>
                  -> { <Type₂> : <encoding₂> } | <Type₂>
                  -> ...
                  -> { <TypeN> : <encodingN> } | <TypeN>
-@}
```


* The `ONCHAIN` annotation is the primary entry point for designating a surface language
function as a target for UPLC translation and formal verification.
* A function marked `ONCHAIN` can be referenced by its name in any `PROPERTY` annotation.
* The annotation accepts an optional budget field, written as `[exCPU : <nat>, exMem : <nat>]`,
that can be used to specify the maximum permitted CPU steps and memory units the compiled UPLC term
may consume under the current Plutus cost model. If the budget is omitted a default budget is considered.
* The annotation also accepts a version field, written as `[version : <PlutusLanguageVersion>]`,
that may be used to specify the targeted Plutus language version. The supported versions are `PlutusV1`, `PlutusV2`,
`PlutusV3` and `PlutusV4`. The version field is mainly used when applying the cost model and
the transaction validation rules when trying to prove the annotated function.
* The `ONCHAIN` annotation also carries a refined function signature in which each argument type is
wrapped in a brace pair `{ Type : encodingScheme }` that declares how that argument is to be serialised
into a UPLC Data or lambda term. Supported encoding scheme identifiers are `asData` and `asScott`.
These encoding schemes are considered when applying the function’s arguments to its UPLC representation,
thus obtaining a fully-applied UPLC term. The `asData` scheme is considered by default when the brace
pair annotation is not used.

***Function Annotation in Plinth***

```
{-@ ONCHAIN [version: PlutusV2] [exCPU: 1883313, exMem: 12342]
    findScriptHash :: { ByteString  : asData  }
                   -> { Integer     : asScott }
                   -> { [TxInInfo]  : asData  }
                   -> { TxOut       : asData  }
-@}
findScriptHash :: ByteString -> Integer -> [TxInInfo] -> TxOut
findScriptHash sh adaValue inputs =
...
```

***Script annotation in Plinth with Concrete Data Types***

```
{-@ ONCHAIN [version: PlutusV3] [exCPU: 1883313, exMem: 12342]
    mintingContract :: CurrencySymbol
                    -> ScriptContext
			-> ()
-@}
mintingContract :: CurrencySymbol -> ScriptContext -> ()
mintingContract currencySymbol ctx =
...
```
Here, the `asData` scheme is applied by default for all arguments.

***Script annotation in Plinth with Builtin Data types***

```
{-@ ONCHAIN [version: PlutusV3] [exCPU: 1883313, exMem: 12342]
    mintingContract :: CurrencySymbol
                    -> BuiltinData
			-> BuiltinUnit
-@}
mintingContract :: CurrencySymbol -> BuiltinData -> BuiltinUnit
mintingContract currencySymbol ctx =
...
```
The `asData` scheme is still applied by default for all arguments but will correspond
to the identity function for the Builtin Data types.

### 2.3 Data Type Annotation

```
{-@ UPLC_DATA <typeName> -@}
```

* The `UPLC_DATA` annotation is essentially used to designate a user-defined data type as a target for Lean 4 translation.
This annotation is expected to be used only for data types not explicitly defined in the `CardanoLedgerApi` Lean 4 formalization.
* The annotation will also automatically generate the `IsData` and `IsScott` instances for the designated data type in
the produced Lean 4 formalization. These instances may be used either when there is a need to apply the data type
to a UPLC function or when writing predicates and properties.
* The designated data type must also be composed of data types for which a builtin representation exists at the UPLC level.
Otherwise, an error will be triggered at elaboration time.

***Data Type Annotation in Plinth***

```
{-@ UPLC_DATA SellDatum -@}
data SellDatum =
  SellDatum
  { seller :: Addresss
  , price :: Integer
  }
 deriving stock (Prelude.Show, Generic, Prelude.Eq)
 deriving anyclass (FromJSON, ToJSON)
```

### 2.4 Property Annotation

```
{-@ PROPERTY <theoremName> : <theoremBody> [:= by <tactic> ] -@}
```

* A `PROPERTY` annotation is mainly used to specify a theorem to be satisfied by one (or more) function(s) marked as `ONCHAIN`.
It expects a theorem name, a theorem body and an optional Lean 4 tactic to be used.
The `blaster` tactic is by default used when trying to prove a proof obligation.
* Multiple `PROPERTY` annotations may be specified within the same surface language file.

***Annotated Property for Minting Validator in Plinth***

```
{-@ PROPERTY minting_policy_success_imp_mintScript_info_withdrawal :
  ∀ (pparamsCs : CurrencySymbol) (ctx : ScriptContext),
      validMintingContext ctx →
      isSuccessful (mintingContract pparamsCs ctx) →
      mintingLogicInWithdrawalMap pparamsCs ctx
-@}
```

## 3. Predicate Derivation from CardanoLedgerApi

* UAL predicates may be freely composed of any definitions present in the `CardanoLedgerApi` Lean 4 formalization
(e.g., `findOwnInput`, `txSignedBy`, `onlySignedBy`, `validScriptContext`, etc).
* A UAL-specific `import CardanoLedgerApi.Vx` module will expose the types and definitions
forming the standard Cardano Ledger Api library for the expected Plutus version.
E.g. `V1` for `PlutusV1`, `V2` for `PlutusV2`, etc.
* Users may derive new predicates by combining standard library predicates with logical connectives (`∧`, `∨`, `¬`, `→`, `∀`, `∃`)
in the `PREDICATE` section.
* New predicates are checked for well-typedness at elaboration time via Lean 4's kernel,
providing early feedback before any proof is attempted.
* Recursive predicates (e.g., over lists of outputs) are permitted provided Lean 4's termination checker
accepts the recursive structure.


## 4. Lean 4 Formalization Elaborator

* All `PREDICATE` annotations under a declaring surface language module will be regrouped within the same Lean 4 file
under namespace `<surface_language_module>Predicates`.
* Inter-dependencies between surface language modules must be preserved at the Lean 4 level.
* For instance, if a module `M1` imports a module `M2` and both modules have `PREDICATE` annotations
then the elaborated Lean4 module `M2Predicates` will import Lean 4 module `M1Predicates`.
* A flat encoded UPLC will be generated for each function marked as `ONCHAIN`. A corresponding Lean 4 file will
be elaborated to call both `#import_uplc` and `#prep_uplc` on the flat encoded UPLC generated by the surface language compiler.
* One Lean4 file will be generated for each `PROPERTY` annotation. In this way, we can benefit
from the Lean 4 caching mechanism to avoid re-execution of the proof for already proven and
unaltered properties. Each Lean 4 property file will implicitly import the corresponding predicates
and preprocessed UPLC Lean 4 files.

## 5. Plutus Contract BluePrint Extension

To facilitate the integration of UAL within any Cardano smart contract surface language,
it is expected to extend the Plutus Contract BluePrint format, i.e., the `JSON` schema
that every smart contract compiler must already produce.
Hence, only a single parser is required to read blueprint `JSON` and to generate
the corresponding Lean 4 formalization. Note that the extension of the blueprint
`JSON` schema is **not** part of the UAL design.
