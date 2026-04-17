# Verus Syntax Support in tree-sitter-verus

This document describes all Verus-specific syntax extensions supported by this grammar, beyond standard Rust. Each section shows the syntax, how it parses, and notes any limitations.

## Table of Contents

1. [Primitive Types](#primitive-types)
2. [Function Modes](#function-modes)
3. [Specification Clauses](#specification-clauses)
4. [Named Return Types](#named-return-types)
5. [Ghost and Tracked Bindings](#ghost-and-tracked-bindings)
6. [View Expression (`@`)](#view-expression-)
7. [Proof Blocks](#proof-blocks)
8. [Quantifier Expressions](#quantifier-expressions)
9. [Assert Expressions](#assert-expressions)
10. [Assume Expression](#assume-expression)
11. [Operators](#operators)
12. [Chained Conjunction and Disjunction](#chained-conjunction-and-disjunction)
13. [Attributed Expressions](#attributed-expressions)
14. [Loop Specifications](#loop-specifications)
15. [Broadcast Declarations](#broadcast-declarations)
16. [Verus Blocks](#verus-blocks)
17. [Test Coverage](#test-coverage)
18. [Known Limitations](#known-limitations)

---

## Primitive Types

Verus adds `int` and `nat` to Rust's numeric types. These work as type names and as integer literal suffixes.

```verus
spec fn example(x: int, y: nat) -> int { x + y as int }
let n = 42int;
let m = 0nat;
```

**Grammar:** `int` and `nat` are in `numericTypes`, making them valid in `primitive_type` and `integer_literal` suffix positions.

---

## Function Modes

Verus functions have three modes that determine whether they are compiled or ghost.

```verus
spec fn abs(n: int) -> int { if n < 0 { -n } else { n } }
proof fn lemma_add(x: int) requires x > 0 ensures x + 1 > 0 { }
fn runtime_add(x: u32, y: u32) -> u32 { x + y }
```

**Modifiers** (can be combined):

| Modifier | Meaning |
|----------|---------|
| `spec` | Mathematical specification (ghost, not compiled) |
| `proof` | Verification proof (ghost, not compiled) |
| `exec` | Executable code (compiled, this is the default) |
| `open` | Spec body visible to other modules |
| `closed` | Spec body hidden from other modules |
| `broadcast` | Lemma automatically applied by the verifier |
| `axiom` | Accepted without proof |

```verus
pub open spec fn visible_spec() -> int { 42 }
pub closed spec fn hidden_spec() -> int { 42 }
broadcast proof fn auto_lemma() ensures true { }
```

**Grammar:** All modifiers are alternatives in `function_modifiers`. They appear before `fn` and after optional `pub`.

---

## Specification Clauses

Functions can have `requires`, `ensures`, `recommends`, and `decreases` clauses. These appear after the parameter list / return type and before the body, in any order.

```verus
fn add(x: u32, y: u32) -> (result: u32)
    requires x < 1000, y < 1000,
    ensures result == x + y,
{
    x + y
}

spec fn fib(n: nat) -> nat
    decreases n,
{
    if n <= 1 { n } else { fib(n - 1) + fib(n - 2) }
}

spec fn div_safe(x: int, y: int) -> int
    recommends y != 0,
{
    x / y
}
```

**Grammar rules:**
- `requires_clause` — `requires expr1, expr2, ...`
- `ensures_clause` — `ensures expr1, expr2, ...`
- `recommends_clause` — `recommends expr1, expr2, ...`
- `decreases_clause` — `decreases expr1, expr2, ...`

Each clause contains a comma-separated list of expressions with an optional trailing comma. The clauses are parsed as part of `function_item` and `function_signature_item` in a `repeat(choice(...))` so they can appear in any order, interleaved with `where_clause`.

---

## Named Return Types

Verus supports naming the return value for use in `ensures` clauses.

```verus
fn add_one(x: u32) -> (result: u32)
    ensures result == x + 1,
{
    x + 1
}
```

**Grammar:** `named_return_type` — `(name: type)` — is an alternative to `_type` in the return type position.

---

## Ghost and Tracked Bindings

The `ghost` and `tracked` qualifiers create ghost (specification-only) or tracked (linear-type) bindings.

```verus
let ghost len = v@.len();
let ghost mut counter = 0;
let tracked resource = take_resource();
```

**Grammar:** `let_declaration` has `optional(choice('ghost', 'tracked'))` after `let` and before the optional `mut` specifier.

**Parameters** can also be ghost or tracked:

```verus
fn example(ghost x: int, tracked y: Resource) { }
```

**Grammar:** `parameter` has `optional(choice('ghost', 'tracked'))` before the pattern.

---

## View Expression (`@`)

The `@` postfix operator converts a runtime value to its specification view (e.g., `Vec<T>` to `Seq<T>`).

```verus
v@              // view of v
v@.len()        // method call on view
v@[i]           // index into view
v@.subrange(a, b)  // subrange of view
old(v)@         // view of old value
```

**Grammar:** `view_expression` — `prec(PREC.field, seq(value: _expression, '@'))`. At field-access precedence (15), so `v@.len()` parses as `(v@).len()` and `v@[i]` as `(v@)[i]`.

---

## Proof Blocks

Proof blocks contain ghost code (assertions, lemma calls) inside executable functions.

```verus
fn example() {
    // exec code
    proof {
        assert(1 + 1 == 2);
        lemma_something();
    }
    // more exec code
}
```

**Grammar:** `proof_block` — `seq('proof', block)`. It is an `_expression_ending_with_block`, so it can appear as a statement without a trailing semicolon.

---

## Quantifier Expressions

Verus has `forall`, `exists`, and `choose` quantifier expressions. They use closure-like parameter syntax with `|params|`.

```verus
forall |i: int| 0 <= i < s.len() ==> s[i] > 0

exists |i: int| 0 <= i < s.len() && s[i] == 0

let w = choose |i: int| 0 <= i < s.len() && s[i] > 10;
```

**Grammar:** `forall_expression`, `exists_expression`, `choose_expression` — each is `prec(PREC.closure, seq(keyword, closure_parameters, body: _expression))`. The body extends to the right as far as possible (closure precedence is -1, the lowest).

---

## Assert Expressions

Verus has several assert forms. They split into two categories: non-block-ending (need `;`) and block-ending (end with `{ }`).

### Simple assert (non-block-ending)

```verus
assert(condition);
assert(condition) by(method);
```

Parsed as `assert_expression`. Needs `;` in a statement.

### Assert by block (block-ending)

```verus
assert(x > 0) by {
    // proof code
}

assert(x > 0) by(nonlinear_arith) {
    // proof code
}
```

Parsed as `_assert_by_expression` (aliased to `assert_expression` in the tree). Does not need `;`.

### Assert by solver with requires

```verus
assert(x * y > 0) by(nonlinear_arith)
    requires x > 0, y > 0;

assert(x * y > 0) by(nonlinear_arith)
    requires x > 0, y > 0,
{
    // proof code
}
```

The `requires` content is parsed as **opaque tokens** (not structured expressions) to avoid expression-boundary conflicts with the tree-sitter parser generator. This is correct because the transpiler skips all assert/proof constructs. Balanced `()`, `[]`, and `{}` are tracked so that expressions like `if x { a } else { b }` inside requires clauses parse correctly.

When there is no `{ body }`, the `;` terminates the statement.

### Assert forall (quantified assertions)

```verus
assert forall |i: int| condition implies conclusion by {
    // proof of each case
}
```

Both block-ending (`by { }`) and non-block-ending variants are supported. The `forall` form uses `closure_parameters` for the quantified variables.

### Assert by solver (no body, no requires)

```verus
assert(x & 1 == 0) by(bit_vector);
```

Parsed as `_assert_by_expression` with just the solver identifier. The `;` following is parsed as an `empty_statement`.

---

## Assume Expression

```verus
assume(condition);
```

**Grammar:** `assume_expression` — `seq('assume', '(', condition: _expression, ')')`. Non-block-ending, needs `;`.

---

## Operators

### Implication (`==>`, `implies`)

Right-associative at precedence 2 (between `range` and `or`).

```verus
a ==> b         // if a then b
a implies b     // same, keyword form
a ==> b ==> c   // a ==> (b ==> c)  (right-associative)
```

### Biconditional (`<==>`)

Right-associative at precedence 2 (same as implication).

```verus
a <==> b        // a if and only if b
```

### Triple equality (`===`, `!==`)

Structural equality. At comparative precedence (5), same as `==`.

```verus
a === b         // structural equality
a !== b         // structural inequality
```

### Extensional equality (`=~=`, `=~~=`)

Collection equality. At comparative precedence (5).

```verus
seq1 =~= seq2   // extensional equality (elements equal)
seq1 =~~= seq2  // deep extensional equality
```

### All Verus operators in token tree

The following operators are recognized in macro/token-tree contexts: `===`, `!==`, `==>`, `<==>`, `=~=`, `=~~=`, `&&&`, `|||`.

---

## Chained Conjunction and Disjunction

Prefix-form logical operators for multi-clause specifications.

```verus
spec fn multi_cond(x: int, y: int) -> bool {
    &&& x > 0
    &&& y > 0
    &&& x + y > 0
}

spec fn any_cond(x: int) -> bool {
    ||| x == 0
    ||| x == 1
    ||| x == 2
}
```

**Grammar:**
- `chained_conjunction` — `prec.left(PREC.and, repeat1(seq('&&&', prec(PREC.and + 1, _expression))))`
- `chained_disjunction` — `prec.left(PREC.or, repeat1(seq('|||', prec(PREC.or + 1, _expression))))`

Each arm binds tighter than the chaining operator, so `&&& a > 0 &&& b > 0` parses as two arms `[a > 0, b > 0]`.

---

## Attributed Expressions

Expression-level attributes, most commonly `#[trigger]` for quantifier trigger annotations.

```verus
forall |i: int| 0 <= i < n ==> #[trigger] s[i] > 0

assert forall |j: int| condition implies
    0 <= (#[trigger] out@[j]).sem() < 100
by { }
```

**Grammar:** `attributed_expression` — `prec(-1, seq(repeat1(attribute_item), expression: _expression))`. The low precedence (-1) ensures that `#[attr]` before a declaration (like `#[verifier::opaque] fn ...`) is preferred over treating it as an attributed expression.

---

## Loop Specifications

`while`, `loop`, and `for` loops can have `invariant`, `invariant_except_break`, and `decreases` clauses.

```verus
while i < n
    invariant
        i <= n,
        sum == triangle(i as nat),
    decreases n - i,
{
    // loop body
}

for k in 0..n
    invariant n <= v@.len(),
{
    // loop body
}

loop
    invariant_except_break x > 0,
{
    // loop body
}
```

**Grammar:** `invariant_clause` — `seq(choice('invariant', 'invariant_except_break'), expr1, expr2, ...)`. Both `invariant_clause` and `decreases_clause` appear in `repeat(choice(...))` after the loop condition/header and before the body.

---

## Broadcast Declarations

### Broadcast use

Imports broadcast lemmas so they are automatically applied.

```verus
broadcast use lemma_seq_push_index;
broadcast use group_ring_axioms;
broadcast use crate::module::lemma_a, crate::module::lemma_b;
```

**Grammar:** `broadcast_use_item` — `seq('broadcast', 'use', paths, ';')`.

### Broadcast group

Groups broadcast lemmas under a name.

```verus
pub broadcast group group_ring_axioms {
    lemma_add_comm,
    lemma_mul_comm,
}
```

**Grammar:** `broadcast_group` — `seq(visibility?, 'broadcast', 'group', name, '{', paths, '}')`.

---

## Verus Blocks

The `verus! { }` macro block parses its body as structured Verus declarations rather than opaque macro tokens.

```verus
verus! {
    spec fn double(x: int) -> int {
        x + x
    }

    proof fn lemma_double(x: int)
        ensures double(x) == x + x,
    {
    }
}
```

**Grammar:** `verus_block` — `seq('verus', '!', declaration_list)`. The body is parsed with the full declaration grammar (functions, types, impls, etc.), not as a token tree.

---

## Test Coverage

All syntax described above is covered by tests in `test/corpus/verus.txt` (47 tests). The test corpus includes:

| Category | Tests |
|----------|-------|
| Function modes | Spec function, Open spec function, Broadcast proof function |
| Specification clauses | Requires/ensures, Decreases, Recommends, Multiple spec clauses |
| Named return types | Named return type |
| Ghost/tracked | Let ghost binding, Let tracked binding, Ghost parameter, Ghost bool capture, Ghost wrapper call |
| View expression | Basic `@`, With indexing, With subrange, In ensures |
| Proof blocks | Basic proof block, With assert forall by, Turbofish generic call |
| Quantifiers | Forall, Exists, Choose |
| Assert | Simple, By block, By solver no body, By solver requires no body, By solver with body, By solver requires with body, Assert forall with trigger, Assert with method name |
| Operators | `==>`, `implies`, `<==>`, `===`/`!==`, `=~=`/`=~~=` |
| Chained operators | `&&&`, `\|\|\|` |
| Attributed expressions | `#[trigger]` in quantifiers |
| Loop specifications | While with invariant/decreases, For with invariant, Invariant_except_break |
| Broadcast | Broadcast use, Broadcast group |
| Verus blocks | `verus! { }` |
| Misc | Deref-reref `&*`, Cross-module calls, Complex combined example |

---

## Known Limitations

### Remaining ERROR sources

The grammar achieves **0 ERRORs** on the primary Verus codebase files (`limb_ops.rs`, `limb_ops_proofs.rs`, `gpu_perturbation_entry.rs`). Some constructs may still produce ERRORs in other files:

1. **`no_unwind` clauses** — Not yet supported. Rarely used.

2. **`decreases ... via fn_name`** — The `via` termination proof clause is not specifically handled. The `via` keyword would be parsed as part of the decreases expression.

3. **`open spec fn` with complex ensures** — Ensures clauses with deeply nested expressions may hit parsing ambiguities in rare cases.

### Opaque requires in assert-by

The `requires` clause in `assert(...) by(solver) requires R;` and `assert(...) by(solver) requires R { body }` is parsed as **opaque tokens** rather than structured expressions. This means:

- The parse tree contains raw tokens (identifiers, operators, literals) rather than expression nodes inside the requires clause.
- Balanced `()`, `[]`, and `{}` are correctly tracked.
- The `;` terminator and `{` body opener are correctly excluded from the requires content.
- This design avoids cascading expression-boundary conflicts in the tree-sitter parser generator.
- The transpiler is unaffected since it skips all assert/proof constructs.

Function-level `requires` clauses (on `fn` declarations) are parsed with full expression structure.

### `@` view operator whitespace

The `@` view operator does not enforce zero whitespace between the expression and `@`. Both `v@` and `v @` parse identically. In practice, Verus code always uses `v@` without a space.

### Captured pattern vs view expression

The `@` token is used for two purposes:
- **Patterns:** `x @ pattern` (captured pattern, e.g., in `match` arms)
- **Expressions:** `expr@` (view operator)

These are in different syntactic contexts (pattern vs expression) and do not conflict.
