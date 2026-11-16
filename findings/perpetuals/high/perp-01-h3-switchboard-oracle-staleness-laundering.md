# Source-code-based

Project: Name-redacted perpetuals protocol (PERP-01)
Category: Perpetuals
Severity: High — final client-report assignment; historical finding
Engagement: Private engagement
Chain: Solana
Tech stack: Rust, Anchor, Solana runtime, SPL Token/SPL Governance, oracle integrations; Rust local tests and configured Surfpool evidence where stated
Evidence: Original local test source and historical report; component, model or conditional-impact proof
Remediation: Resolved — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Switchboard Oracle Staleness Laundering

## Description

Permissionless updater captures a signed quote, waits within provider's roughly 20-second maximum age, chooses max_age_slots and calls update_oracle; timestamp is restamped current, so downstream freshness checks accept stale price.

Stale-price trades and compound oracle/liquidation attacks; valid signature/provider outer freshness limit still required.

The displayed unit test directly constructs a restamped record with a one-hour age comment. That intentionally synthetic example only proves the downstream predicate; it does not establish provider acceptance of a one-hour-old signed quote. Integration variants in the same file install a mock provider and constructed quote accounts. The final severity is High, reflecting composition with downstream oracle consumers.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Replay a valid quote inside the upstream provider window

The updater chooses max_age_slots, which controls this protocol-level age check. A large caller value weakens that check; provider signature/quote verification remains a prerequisite.

Source: `programs/[project]/src/state/oracle.rs:560-570`

```rust
            let quote_age_slots = current_slot.saturating_sub(parsed_quote.slot);
            if quote_age_slots > params.max_age_slots {
                msg!(
                    "Switchboard quote stale: quote_slot={}, current_slot={}, age_slots={}, max_age={}",
                    parsed_quote.slot,
                    current_slot,
                    quote_age_slots,
                    params.max_age_slots
                );
                return Err([project]Error::SwitchboardQuoteTooStale.into());
            }
```

#### Step 2 — Restamp the accepted quote as current

The original quote timestamp is replaced by current_time, so downstream consumers cannot measure its actual age.

Source: `programs/[project]/src/state/oracle.rs:609-613`

```rust
            if stored_price.timestamp < current_time {
                stored_price.timestamp = current_time;
                stored_price.price = new_price;
                stored_price.confidence = get_confidence_from_price(new_price, *required_feed_id)?;
            }
```

#### Step 3 — Consume the laundered timestamp

The freshness predicate accepts the stored timestamp for the configured window. The final report bounds the provider replay opportunity to about 50 slots or 20 seconds, rather than accepting arbitrarily old production quotes.

Source: `programs/[project]/src/state/oracle.rs:1068-1078`

```rust
    pub fn check_price_staleness_with(
        &self,
        current_time: i64,
        staleness_seconds: i64,
    ) -> Result<()> {
        require!(
            self.timestamp + staleness_seconds >= current_time,
            [project]Error::StaleOraclePrice
        );
        Ok(())
    }
```

## Proof of concept

Test: `programs/[project]/tests/[auditor]_tests/h3_switchboard_staleness_laundering.rs`; function `switchboard_laundered_timestamp_passes_staleness_check`.

The displayed unit test directly constructs a restamped record with a one-hour age comment. That intentionally synthetic example only proves the downstream predicate; it does not establish provider acceptance of a one-hour-old signed quote. Integration variants in the same file install a mock provider and constructed quote accounts. The final severity is High, reflecting composition with downstream oracle consumers.

The excerpts show the original test assertions. Local funding, position construction, oracle inputs and any privileged setup are fixtures; they do not establish those conditions on a live deployment. The tests require their matching historical source revision, and repaired code may correctly reject the exploit expectation.

## Relevant test excerpts

Source: `programs/[project]/tests/[auditor]_tests/h3_switchboard_staleness_laundering.rs:195-226`

```rust
fn switchboard_laundered_timestamp_passes_staleness_check() {
    let staleness_seconds: i64 = 7;
    let current_time: i64 = 1_700_000_000;
    let _actual_feed_age_seconds: i64 = 3600; // 1 hour old

    // After ingestion, stored_price.timestamp = current_time (oracle.rs:610)
    let laundered_price = OraclePrice {
        price: 1_500_000_000_000,
        confidence: 15_000_000,
        timestamp: current_time, // ← actual age erased
        exponent: 0,
        feed_id: 142, // Switchboard feed_id range: 142-255
        _padding: [0u8; 3],
        name: [project]::utils::limited_string::LimitedString::default(),
    };

    assert!(
        laundered_price
            .check_price_staleness_with(current_time, staleness_seconds)
            .is_ok(),
        "VULNERABILITY: A 1-hour-old Switchboard price passes the 7-second \
         staleness check because its timestamp was laundered to current_time"
    );

    let slightly_later = current_time + 6;
    assert!(
        laundered_price
            .check_price_staleness_with(slightly_later, staleness_seconds)
            .is_ok(),
        "Laundered price remains 'fresh' for the full staleness window"
    );
}
```
