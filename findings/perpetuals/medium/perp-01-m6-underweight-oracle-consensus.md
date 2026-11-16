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

# Underweight Oracle Consensus

## Description

For hypothetical min_agree=3 pool, attacker controlling two backup feeds provides matching values; selector has no dedicated three-agreement branch and takes two-of-three fallback.

Configured three-provider requirement is bypassed; pool-drain consequence conditional on compromised feeds/configuration, which report says no existing pool used.

The test supplies two matching manipulated backups, a disagreeing primary and min_agree=3. The final report says no existing pool used this configuration. Exploit impact requires the configuration and control or failure of the relevant feeds; the synthetic counterexample is not a demonstrated production pool drain.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Select the branch for the configured agreement count

The implementation has a special min_agree=1 branch; min_agree=3 instead reaches the same subsequent logic as min_agree=2.

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

#### Step 2 — Accept two agreeing backups without the primary

If the three-way test fails, the secondary/tertiary test still returns their price. A configuration requiring all three is silently weakened.

Source: `programs/[project]/src/state/oracle.rs:665-679`

```rust
        if let (Some(p), Some(s), Some(t)) = (primary_price, secondary_price, tertiary_price) {
            if within_threshold(p.price, s.price, pool_cfg.price_diff_threshold_bps)?
                && within_threshold(p.price, t.price, pool_cfg.price_diff_threshold_bps)?
            {
                return Ok(p);
            }
        }

        if let (Some(s), Some(t)) = (secondary_price, tertiary_price) {
            if within_threshold(s.price, t.price, pool_cfg.price_diff_threshold_bps)? {
                return Ok(latest_of_two(Some(s), Some(t)).unwrap());
            }
        }

        Err([project]Error::MissingOraclePrice.into())
```

## Proof of concept

Test: `programs/[project]/tests/[auditor]_tests/m6_min_agree_silent_downgrade.rs`; function `m2_poc_min_agree_3_accepts_two_provider_consensus`.

The test supplies two matching manipulated backups, a disagreeing primary and min_agree=3. The final report says no existing pool used this configuration. Exploit impact requires the configuration and control or failure of the relevant feeds; the synthetic counterexample is not a demonstrated production pool drain.

The excerpts show the original test assertions. Local funding, position construction, oracle inputs and any privileged setup are fixtures; they do not establish those conditions on a live deployment. The tests require their matching historical source revision, and repaired code may correctly reject the exploit expectation.

## Relevant test excerpts

Source: `programs/[project]/tests/[auditor]_tests/m6_min_agree_silent_downgrade.rs:210-250`

```rust
pub async fn m2_poc_min_agree_3_accepts_two_provider_consensus() {
    let now: i64 = 1_700_000_000;
    let true_price: u64 = 1_500_000; // Switchboard: $1,500 (scaled ×10^3 for clarity)
    let manip_price: u64 = 3_000_000; // ChaosLabs + Autonom: $3,000 — 100% inflation

    let oracle = build_three_provider_eth_oracle(true_price, manip_price, manip_price, now);

    // threshold_bps = 2000 (20%):
    //   $1,500 vs $3,000 → 50% diff > 20% → fails all-3 check
    //   $3,000 vs $3,000 →  0% diff ≤ 20% → passes 2-of-3 fallback
    let cfg = gmx_config(3, 2_000);

    let result = oracle.get_selected_price_by_feed_id_or_name(
        LimitedString::new("eth"),
        ChaosLabsFeedIdEnum::ETH.to_u8(),
        &cfg,
        now,
    );

    // On unpatched code the 2-of-3 fallback fires and returns $3,000.
    assert!(
        result.is_ok(),
        "M-2 PoC: min_agree=3 MUST silently accept secondary+tertiary consensus \
         on unpatched code, returning Ok($3,000). If this is Err, the bug may \
         already be patched."
    );

    let selected = result.unwrap();
    assert_eq!(
        selected.price, manip_price,
        "M-2 PoC: returned price MUST be the manipulated secondary+tertiary \
         consensus ($3,000), NOT the true Switchboard price ($1,500). \
         Got: {}, expected: {}",
        selected.price, manip_price
    );

    assert_ne!(
        selected.price, true_price,
        "M-2 PoC: the Switchboard price ($1,500) must NOT be returned — \
         it was overridden by the 2-of-3 fallback."
    );
```
