# Source-code-based

Project: Name-redacted perpetuals protocol (PERP-01)
Category: Perpetuals
Severity: High — final client-report assignment; historical finding
Engagement: Private engagement
Chain: Solana
Tech stack: Rust, Anchor, Solana runtime, SPL Token/SPL Governance, oracle integrations; Rust local tests and configured Surfpool evidence where stated
Evidence: Original local test source and historical report; component, model or conditional-impact proof
Remediation: Remediated (published label; residual exposure described) — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Oracle guarantees degrade under staleness

## Description

With min_agree=1 and stale/unavailable backup feeds, a trader or malicious primary/crank operator submits/uses a primary price; selector falls back without backup agreement, allowing single-feed trades/liquidations.

Loss of configured multi-provider protection; adverse liquidation/pricing conditional on primary manipulation or stale replay. Residual path remains in report body.

The test constructs fresh and stale provider records and compares selector outputs. The final report labels H2 Remediated, but its body says the min_agree=1 path remains open, a fresh backup for another asset can satisfy the liquidation circuit breaker, and other instructions lack that breaker. The remediation label does not establish closure of the remaining path.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Allow a primary-only price when backups expire

With min_agree=1, the threshold comparison runs only when both backups are present. Once one backup is stale or unavailable, a fresh primary is returned without corroboration.

Source: `programs/[project]/src/state/oracle.rs:641-651`

```rust
        if pool_cfg.min_agree == 1 {
            if let Some(p) = primary_price {
                if let (Some(s), Some(t)) = (secondary_price, tertiary_price) {
                    if within_threshold(p.price, s.price, pool_cfg.price_diff_threshold_bps)?
                        || within_threshold(p.price, t.price, pool_cfg.price_diff_threshold_bps)?
                    {
                        return Ok(p);
                    }
                    return Ok(s);
                }
                return Ok(p);
```

#### Step 2 — Fall back to a single backup if the primary is absent

The remaining fallbacks likewise return one available provider. A compromised primary or bad single provider can therefore drive dependent pricing; merely observing backup staleness does not give an attacker control of a signed price.

Source: `programs/[project]/src/state/oracle.rs:653-659`

```rust
            if let Some(s) = secondary_price {
                return Ok(s);
            }
            if let Some(t) = tertiary_price {
                return Ok(t);
            }
            return Err([project]Error::MissingOraclePrice.into());
```

## Proof of concept

Test: `programs/[project]/tests/[auditor]_tests/h2_autonom_single_oracle_degradation.rs`; function `m3_poc_stale_backups_disable_threshold_protection`.

The test constructs fresh and stale provider records and compares selector outputs. The final report labels H2 Remediated, but its body says the min_agree=1 path remains open, a fresh backup for another asset can satisfy the liquidation circuit breaker, and other instructions lack that breaker. The remediation label does not establish closure of the remaining path.

The excerpts show the original test assertions. Local funding, position construction, oracle inputs and any privileged setup are fixtures; they do not establish those conditions on a live deployment. The tests require their matching historical source revision, and repaired code may correctly reject the exploit expectation.

## Relevant test excerpts

Source: `programs/[project]/tests/[auditor]_tests/h2_autonom_single_oracle_degradation.rs:283-305`

```rust
    let price_with_stale_backups = oracle_stale
        .get_selected_price_by_feed_id_or_name(
            LimitedString::new("eth"),
            ChaosLabsFeedIdEnum::ETH.to_u8(),
            &cfg,
            now,
        )
        .expect("Must succeed — primary fresh, backups stale → Ok(primary)")
        .price;

    assert_eq!(
        price_with_stale_backups, manip_price,
        "M-3 PoC: With STALE backups, manipulated primary MUST be accepted \
         without corroboration (threshold protection evaporates). \
         Got: {}, expected: {}",
        price_with_stale_backups, manip_price
    );

    assert_ne!(
        price_with_fresh_backups, price_with_stale_backups,
        "M-3 PoC: The two cases MUST return different prices, proving the \
         oracle security guarantee disappears under stale backup conditions."
    );
```
