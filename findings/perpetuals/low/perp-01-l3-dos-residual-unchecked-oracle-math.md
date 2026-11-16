# Source-code-based

Project: Name-redacted perpetuals protocol (PERP-01)
Category: Perpetuals
Severity: Low — final client-report assignment; historical finding
Engagement: Private engagement
Chain: Solana
Tech stack: Rust, Anchor, Solana runtime, SPL Token/SPL Governance, oracle integrations; Rust local tests and configured Surfpool evidence where stated
Evidence: Original local test source and historical report; component, model or conditional-impact proof
Remediation: Resolved — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# DOS: Residual Unchecked Oracle Math

## Description

With corrupted/extreme provider price/confidence state, a user operation invokes OraclePrice.low/high; plain arithmetic underflows/overflows with overflow checks enabled and aborts.

Affected-custody closes, liquidations and collateral changes can freeze until correction; extreme-state reachability is conditional, not a silent-wrap theft.

The PoC directly constructs price=500,000 and confidence=600,000 and catches the panic. This proves arithmetic behavior, not an ordinary caller ability to install malicious oracle state. Integration variants inject local oracle state. The historical rating is Low/Resolved; the provider-state prerequisite remains conditional.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Underflow the lower confidence bound

A malformed or extreme oracle state with confidence greater than price reaches plain subtraction. Overflow checking makes this abort rather than silently wrap.

Source: `programs/[project]/src/state/oracle.rs:922-931`

```rust
    pub fn low(&self) -> Self {
        Self {
            price: self.price - self.confidence,
            exponent: self.exponent,
            confidence: 0,
            timestamp: self.timestamp,
            feed_id: self.feed_id,
            ..Default::default()
        }
    }
```

#### Step 2 — Overflow the upper confidence bound

Likewise price+confidence beyond u64 limits aborts. Operations depending on the affected custody price cannot proceed until valid state is restored.

Source: `programs/[project]/src/state/oracle.rs:933-941`

```rust
    pub fn high(&self) -> Self {
        Self {
            price: self.price + self.confidence,
            exponent: self.exponent,
            confidence: 0,
            timestamp: self.timestamp,
            feed_id: self.feed_id,
            ..Default::default()
        }
```

## Proof of concept

Test: `programs/[project]/tests/[auditor]_tests/l3_dos_unchecked_oracle_math.rs`; function `l1_poc_low_aborts_when_confidence_exceeds_price`.

The PoC directly constructs price=500,000 and confidence=600,000 and catches the panic. This proves arithmetic behavior, not an ordinary caller ability to install malicious oracle state. Integration variants inject local oracle state. The historical rating is Low/Resolved; the provider-state prerequisite remains conditional.

The excerpts show the original test assertions. Local funding, position construction, oracle inputs and any privileged setup are fixtures; they do not establish those conditions on a live deployment. The tests require their matching historical source revision, and repaired code may correctly reject the exploit expectation.

## Relevant test excerpts

Source: `programs/[project]/tests/[auditor]_tests/l3_dos_unchecked_oracle_math.rs:96-115`

```rust
pub async fn l1_poc_low_aborts_when_confidence_exceeds_price() {
    let oracle = OraclePrice {
        price: 500_000,       // $0.0005 — small price
        confidence: 600_000,  // confidence exceeds price → underflow
        timestamp: 1_700_000_000,
        exponent: -9,
        feed_id: 0,
        _padding: [0u8; 3],
        name: Default::default(),
    };

    let result = panic::catch_unwind(AssertUnwindSafe(|| oracle.low()));

    assert!(
        result.is_err(),
        "L-1 PoC: low() must abort (panic/trap) when confidence ({}) > price ({}). \
         If this passes without aborting, the arithmetic is no longer plain subtraction.",
        oracle.confidence, oracle.price,
    );

```
