# Source-code-based

Project: Name-redacted perpetuals protocol (PERP-01)
Category: Perpetuals
Severity: Critical — final client-report assignment; historical finding
Engagement: Private engagement
Chain: Solana
Tech stack: Rust, Anchor, Solana runtime, SPL Token/SPL Governance, oracle integrations; Rust local tests and configured Surfpool evidence where stated
Evidence: Original local test source and historical report; mechanism assertions shown below
Remediation: Resolved — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Missing Timelock on Admin Operations with single step transfer

## Description

An actor controlling the admin key calls set_admin or parameter setters; immediate single-signature authority transfer has no acceptance delay, allowing instant takeover and hostile fee/exit settings.

Source-established unrestricted control after key compromise; report models freeze/fee diversion, not a demonstrated live theft.

The integration test signs as the original administrator to simulate key compromise and supplies a fresh attacker key. The historical test predates the final Critical severity assignment. The final report does not establish that every privileged parameter setter received the same 48-hour delay.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Compromised administrator replaces the authority

The historical PoC comment preserves the old set_admin assignment. Its caller already controls the administrator key; this is not an unprivileged authorization bypass. The assignment commits in the same transaction.

Source: `programs/[project]/tests/[auditor]_tests/c1_no_admin_timelock.rs:15-17`

```rust
//!   // set_admin.rs:31-33
//!   let mut cortex = ctx.accounts.cortex.load_mut()?;
//!   cortex.admin = params.new_admin;  // instant, no delay, no multisig
```

#### Step 2 — Distinguish the historical path from the remediation

The source snapshot now rejects the old instruction. The final report records a two-step administrator transfer and fixed 48-hour delay. This excerpt is remediation evidence, not a reproduction of the historical vulnerability.

Source: `programs/[project]/src/instructions/admin/cortex/set_admin.rs:27-33`

```rust
pub fn set_admin<'info>(
    _ctx: Context<'_, '_, '_, 'info, SetAdmin<'info>>,
    _params: &SetAdminParams,
) -> Result<()> {
    // Deprecated — use propose_admin + accept_admin for two-step transfer with timelock.
    Err(solana_program::program_error::ProgramError::InvalidInstructionData.into())
}
```

## Proof of concept

Test: `programs/[project]/tests/[auditor]_tests/c1_no_admin_timelock.rs`; function `h2_poc_instant_admin_takeover`.

The integration test signs as the original administrator to simulate key compromise and supplies a fresh attacker key. The historical test predates the final Critical severity assignment. The final report does not establish that every privileged parameter setter received the same 48-hour delay.

The excerpts show the original test assertions. Local funding, position construction, oracle inputs and any privileged setup are fixtures; they do not establish those conditions on a live deployment. The tests require their matching historical source revision, and repaired code may correctly reject the exploit expectation.

## Relevant test excerpts

Source: `programs/[project]/tests/[auditor]_tests/c1_no_admin_timelock.rs:194-240`

```rust
pub async fn h2_poc_instant_admin_takeover() {
    let test_setup = timelock_test_setup().await;

    let original_admin = &test_setup.admin_keypair;
    let attacker = Keypair::new(); // simulates attacker's freshly generated key

    eprintln!("=== H-2a PoC: Instant Admin Takeover ===");
    eprintln!("Original admin: {}", original_admin.pubkey());
    eprintln!("Attacker key:   {}", attacker.pubkey());

    // ---- Step 1: Verify original admin controls the protocol ----------------
    let cortex_pda = utils::pda::get_cortex_pda().0;
    let cortex_before =
        utils::get_zero_copy_account::<Cortex>(&test_setup.program_test_ctx, cortex_pda).await;

    assert_eq!(
        cortex_before.admin,
        original_admin.pubkey(),
        "Pre-condition: original admin must control cortex"
    );
    eprintln!("[Step 1] Confirmed: original admin controls cortex");

    // ---- Step 2: Attacker calls set_admin with ONE stolen signature ----------
    // In a real attack the attacker has the private key. Here we call it
    // as the original admin to simulate the key having been compromised.
    // There is no pending state, no delay, no second confirmation required.
    test_instructions::set_admin(
        &test_setup.program_test_ctx,
        &test_setup.payer_keypair,
        original_admin,
        &attacker.pubkey(),
    )
    .await
    .expect("set_admin must succeed — single signature, no timelock");

    // ---- Step 3: Verify takeover is immediate and complete ------------------
    let cortex_after =
        utils::get_zero_copy_account::<Cortex>(&test_setup.program_test_ctx, cortex_pda).await;

    assert_eq!(
        cortex_after.admin,
        attacker.pubkey(),
        "cortex.admin must equal attacker key immediately after set_admin"
    );
    // Crucially: no pending_admin field exists — the transfer is unconditional
    eprintln!("[Step 2] cortex.admin transferred to attacker in one transaction");
    eprintln!("         No pending state. No timelock. No multisig. Takeover complete.");
```
