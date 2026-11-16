# Source-code-based

Project: Name-redacted perpetuals protocol (PERP-01)
Category: Perpetuals
Severity: Medium — final client-report assignment; historical finding
Engagement: Private engagement
Chain: Solana
Tech stack: Rust, Anchor, Solana runtime, SPL Token/SPL Governance, oracle integrations; Rust local tests and configured Surfpool evidence where stated
Evidence: Original local test source and historical report; component, model or conditional-impact proof
Remediation: Resolved — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Incorrect exponent handling

## Description

A caller to checked_decimal_ceil_div with positive exponent reaches addition where subtraction is required, double-counting the exponent and inflating result by 10^(2*exponent1).

Arithmetic counterexample established; function has zero production callers in the final report, so deployed economic exploitation is not established.

This is a direct arithmetic test. The final report explicitly states the public helper had zero production callers. The historical report rated this Medium and marked it Resolved; economically reachable deployed exploitation is not established.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Double-count a positive numerator exponent

The ceil-division helper moves a positive exponent into the scaling factor, then adds it again to target_power rather than compensating by subtraction.

Source: `programs/[project]/src/math.rs:102-125`

```rust
pub fn checked_decimal_ceil_div(
    coefficient1: u64,
    exponent1: i32,
    coefficient2: u64,
    exponent2: i32,
    target_exponent: i32,
) -> Result<u64> {
    if coefficient2 == 0 {
        msg!("Error: Overflow in {} / {}", coefficient1, coefficient2);
        return err!([project]Error::MathOverflow);
    }

    if coefficient1 == 0 {
        return Ok(0);
    }

    // Compute scale factor for the dividend
    let mut scale_factor = 0;
    let mut target_power = exponent1 - exponent2 - target_exponent;

    if exponent1 > 0 {
        scale_factor += exponent1;
        target_power += exponent1;
    }
```

#### Step 2 — Observe the arithmetic mismatch against the floor helper

For the exactly divisible input 50/2, the reference returns 25 while the historical ceil helper returns 2,500. The ratio is 10^(2*exponent1).

Source: `programs/[project]/tests/[auditor]_tests/m7_checked_decimal_ceil_div_exponent_bug.rs:116-140`

```rust
pub async fn m1_poc_ceil_div_inflates_result_with_positive_exponent1() {
    let result = checked_decimal_ceil_div(
        5,   // coefficient1
        1,   // exponent1 = 1  ← triggers the bug
        2,   // coefficient2
        0,   // exponent2
        0,   // target_exponent
    )
    .expect("Function must not return an error for valid inputs");

    // The correct answer: (5 × 10^1) / (2 × 10^0) = 50/2 = 25
    let correct = checked_decimal_div(5, 1, 2, 0, 0)
        .expect("checked_decimal_div must succeed for the same inputs");

    assert_eq!(
        correct, 25,
        "Sanity check: checked_decimal_div must return 25"
    );

    assert_eq!(
        result, 2500,
        "M-1 PoC: checked_decimal_ceil_div with exponent1=1 must return 2500 \
         (100× the correct value of 25), proving the double-count bug. \
         If this assertion fails, the bug may already be fixed."
    );
```

## Proof of concept

Test: `programs/[project]/tests/[auditor]_tests/m7_checked_decimal_ceil_div_exponent_bug.rs`; function `m1_poc_ceil_div_inflates_result_with_positive_exponent1`.

This is a direct arithmetic test. The final report explicitly states the public helper had zero production callers. The historical report rated this Medium and marked it Resolved; economically reachable deployed exploitation is not established.

The excerpts show the original test assertions. Local funding, position construction, oracle inputs and any privileged setup are fixtures; they do not establish those conditions on a live deployment. The tests require their matching historical source revision, and repaired code may correctly reject the exploit expectation.

## Relevant test excerpts

Source: `programs/[project]/tests/[auditor]_tests/m7_checked_decimal_ceil_div_exponent_bug.rs:116-140`

```rust
pub async fn m1_poc_ceil_div_inflates_result_with_positive_exponent1() {
    let result = checked_decimal_ceil_div(
        5,   // coefficient1
        1,   // exponent1 = 1  ← triggers the bug
        2,   // coefficient2
        0,   // exponent2
        0,   // target_exponent
    )
    .expect("Function must not return an error for valid inputs");

    // The correct answer: (5 × 10^1) / (2 × 10^0) = 50/2 = 25
    let correct = checked_decimal_div(5, 1, 2, 0, 0)
        .expect("checked_decimal_div must succeed for the same inputs");

    assert_eq!(
        correct, 25,
        "Sanity check: checked_decimal_div must return 25"
    );

    assert_eq!(
        result, 2500,
        "M-1 PoC: checked_decimal_ceil_div with exponent1=1 must return 2500 \
         (100× the correct value of 25), proving the double-count bug. \
         If this assertion fails, the bug may already be fixed."
    );
```
