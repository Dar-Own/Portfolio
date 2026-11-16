# Source-code-based

Project: Name-redacted perpetuals protocol (PERP-01)
Category: Perpetuals
Severity: Low — final client-report assignment; historical finding
Engagement: Private engagement
Chain: Solana
Tech stack: Rust, Anchor, Solana runtime, SPL Token/SPL Governance, oracle integrations; Rust local tests and configured Surfpool evidence where stated
Evidence: Original local test source and historical report; component, model or conditional-impact proof
Remediation: Acknowledged — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Wrapping Overflow in Financial Calculation

## Description

Accrued fee-debt fields approach integer limits; wrapping_add resets debt and plain summed debt can overflow, breaking correct AUM debt subtraction/distribution.

Theoretical AUM inflation or persistent distribution revert; report calls fee threshold extreme, and project says cited statistics are observability-only. Live reachable loss not established.

The PoC manually seeds lm_fee_debt_usd to u64::MAX-59; accrual of 60 then wraps to zero. This is an artificial boundary-state proof. The final response discusses observability-only accumulators and an operationally unreachable threshold, while the finding body cites operational debt fields. The historical Low/Acknowledged disposition does not establish a reachable live financial exploit.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Wrap accumulated fee debt at the integer limit

The historical operational fee-debt buckets use wrapping_add. A deliberately near-limit starting value can wrap downward after positive accrual.

Source: `programs/[project]/src/state/pool.rs:664-680`

```rust
        self.manager_fee_debt_usd = self.manager_fee_debt_usd.wrapping_add(manager_fee_usd);
        self.cumulative_manager_fee_usd = self
            .cumulative_manager_fee_usd
            .wrapping_add(manager_fee_usd);

        self.lm_fee_debt_usd = self.lm_fee_debt_usd.wrapping_add(lm_fee_usd);
        self.cumulative_lm_fee_usd = self.cumulative_lm_fee_usd.wrapping_add(lm_fee_usd);

        self.protocol_fee_debt_usd = self.protocol_fee_debt_usd.wrapping_add(protocol_fee_usd);
        self.cumulative_protocol_fee_usd = self
            .cumulative_protocol_fee_usd
            .wrapping_add(protocol_fee_usd);

        self.referrers_fee_debt_usd = self.referrers_fee_debt_usd.wrapping_add(referrer_fee_usd);
        self.cumulative_referrer_fee_usd = self
            .cumulative_referrer_fee_usd
            .wrapping_add(referrer_fee_usd);
```

#### Step 2 — Sum debt with ordinary integer addition

The sum can overflow under overflow checks. The report alleges AUM under-deduction or settlement aborts, but naturally reaching the extreme debt state is not established.

Source: `programs/[project]/src/instructions/public/position/distribute_fees.rs:211-215`

```rust
    let total_fee_debt_usd = pool.lm_fee_debt_usd
        + pool.protocol_fee_debt_usd
        + pool.referrers_fee_debt_usd
        + pool.manager_fee_debt_usd;

```

## Proof of concept

Test: `programs/[project]/tests/[auditor]_tests/l12_fee_debt_wrapping_overflow.rs`; function `l3_poc_fee_debt_wraps_silently_at_u64_max`.

The PoC manually seeds lm_fee_debt_usd to u64::MAX-59; accrual of 60 then wraps to zero. This is an artificial boundary-state proof. The final response discusses observability-only accumulators and an operationally unreachable threshold, while the finding body cites operational debt fields. The historical Low/Acknowledged disposition does not establish a reachable live financial exploit.

The excerpts show the original test assertions. Local funding, position construction, oracle inputs and any privileged setup are fixtures; they do not establish those conditions on a live deployment. The tests require their matching historical source revision, and repaired code may correctly reject the exploit expectation.

## Relevant test excerpts

Source: `programs/[project]/tests/[auditor]_tests/l12_fee_debt_wrapping_overflow.rs:141-175`

```rust
pub async fn l3_poc_fee_debt_wraps_silently_at_u64_max() {
    let mut pool = pool_with_valid_fee_config();

    // Position debt just below overflow:
    //   next accrual of lm_fee_usd=60 will push past u64::MAX
    //   (u64::MAX - 59) + 60 = u64::MAX + 1 → wraps to 0
    pool.lm_fee_debt_usd = u64::MAX - 59;

    let debt_before = pool.lm_fee_debt_usd;

    // non_lp_fee_usd=100 → lm_fee_usd = 100 × 3_000 / 5_000 = 60
    let result = pool.accrue_fee_debt(100, 0, false);

    assert!(
        result.is_ok(),
        "L-3 PoC: accrue_fee_debt must NOT return an error on overflow — \
         it wraps silently. If this is Err, wrapping_add was replaced with checked_add."
    );

    let debt_after = pool.lm_fee_debt_usd;

    // With wrapping_add: (u64::MAX - 59) + 60 = 0
    // With checked_add:  would have returned Err (test would have failed above)
    assert_eq!(
        debt_after, 0,
        "L-3 PoC: lm_fee_debt_usd must wrap to 0 when u64::MAX is crossed. \
         Got: {}. Before: {}.",
        debt_after, debt_before
    );

    assert!(
        debt_after < debt_before,
        "L-3 PoC: debt DECREASED after adding fees — the wrap destroyed \
         u64::MAX worth of accumulated fee entitlement silently."
    );
```
