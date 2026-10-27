# UAL 1.0 — Temporal operators for Blaster state machines

## 1. Execution

Let `sm : StateMachine α β`, as defined in the Blaster formalization of state machines with input type `α` and state type `β`.
We define two total sequences: `in : ℤ → α` and `st : ℤ → β`, respectively the sequence of inputs and states.
Indices `i`, `j`, and `r` denote integer times.
The machine starts at `0` with:

```text
st₀ = sm.init(in₀)
stᵢ₊₁ = sm.next(inᵢ₊₁, stᵢ)                 for i ≥ 0
```

Negative-time samples `(inᵢ, stᵢ)` are free symbolic values of their types.
They satisfy no implicit initialization, transition, or assumption constraints. They can however be constrained by temporal operators.

## 2. Expressions

An expression has a fixed Lean type. It is formed from constants, `input`,
`state`, ordinary Lean function applications, and the operators defined below.
Let `e`, `a`, and `b` denote expressions, `c` a time-independent value, and `f` a time-independent Lean function. Write `eᵢ` for the value of `e` at time `i`.

```text
cᵢ = c for all i ∈ ℤ
f(a, b, …)ᵢ = f(aᵢ, bᵢ, …) for all i ∈ ℤ
```

The arguments `a`, `b`, … are subexpressions evaluated at the same time.

## 3. Operators

**Previous and next.**

Both preserve their operand's type. For every `i ∈ ℤ`, we define two expressions `pre(e)` and `next(e)`:

```text
pre(e)ᵢ  = eᵢ₋₁
next(e)ᵢ = eᵢ₊₁
```

Hence, including at `i = 0`:

```text
pre(next(e))ᵢ = eᵢ
next(pre(e))ᵢ = eᵢ
pre(c)ᵢ = c
next(c)ᵢ = c
pre(f(a, b, …))ᵢ = f(aᵢ₋₁, bᵢ₋₁, …)
```
**Weak until.** Let `p` and `q` be expressions of type `Prop`. Their weak until also has type `Prop`:

We define `p W q` as:

```text
(p W q)ᵢ ↔
  (∃ j ≥ i, qⱼ ∧ (∀ r, (i ≤ r ∧ r < j) → pᵣ))
  ∨ (∀ r ≥ i, pᵣ)
```

`p` holds strictly before the first `q`; it is not required at that first `q`.
If `q` never holds, `p` holds forever.

## 4. State Machine Invariants

A root property `p` requires `p₀`.

Blaster state machines define invariants who can be seen as:
```text
invariant p = ∀ i ≥ 0, pᵢ
```
