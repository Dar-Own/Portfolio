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

# Schedule-Dependent Interest and Funding Inflation via Ceiling Division Rounding

## Description

Permissionless caller repeatedly touches a custody over the same elapsed interval; each persisted interest/funding update ceil-rounds its subinterval separately, inflating cumulative indices.

Schedule-dependent borrow/funding charge inflation; project says per-call sub-dust effect cannot amplify to material loss at current block times.

The bounded Kani model has borrow_interest_schedule_divergence_exists and funding_schedule_divergence_exists harnesses. formal/[project]-kani/FINDINGS.md:12-32 records divergence and bounded-divergence results. The final response says the per-call effect is sub-dust and not materially amplifiable at current block times. The model is not a production economic exploit.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Round and persist each borrow-interest interval separately

A caller touching the custody causes ceil division over the elapsed subinterval and advances last_update. Repeating updates retains each rounding increment.

Source: `programs/[project]/src/state/custody.rs:471-498`

```rust
    pub fn get_cumulative_interest(&self, current_time: i64) -> Result<u128> {
        if current_time > self.borrow_rate_state.last_update {
            let cumulative_interest = math::checked_ceil_div(
                (current_time - self.borrow_rate_state.last_update) as u128
                    * self.borrow_rate_state.current_rate as u128,
                3_600,
            )?;

            Ok(self.borrow_rate_state.cumulative_interest.to_u128() + cumulative_interest)
        } else {
            Ok(self.borrow_rate_state.cumulative_interest.to_u128())
        }
    }

    pub fn update_borrow_rate(&mut self, current_time: i64) -> Result<()> {
        if self.assets.owned == 0 {
            self.borrow_rate_state.current_rate = 0;
            self.borrow_rate_state.last_update =
                std::cmp::max(current_time, self.borrow_rate_state.last_update);
            return Ok(());
        }

        if current_time > self.borrow_rate_state.last_update {
            // compute interest accumulated since previous update
            self.borrow_rate_state.cumulative_interest =
                self.get_cumulative_interest(current_time)?.into();
            self.borrow_rate_state.last_update = current_time;
        }
```

#### Step 2 — Apply the same schedule dependence to funding

Funding similarly rounds elapsed-time times rate upward. Splitting a fixed total interval can therefore change the accumulated index in both lanes; these are aliases of one final finding.

Source: `programs/[project]/src/state/custody.rs:584-598`

```rust

        // SAFETY: Using unsigned_abs() to handle the theoretical edge case of rate == i64::MIN.
        // In practice, this is impossible with real funding rates (max ~0.1%/hr = ~10M, vs i64::MAX = 9.2 quintillion).
        // The rate is bounded by max_hourly_funding_rate config which is always tiny relative to i64 limits.
        // This fix is defensive - the overflow condition cannot occur with any realistic configuration.
        let abs_rate = rate.unsigned_abs();

        // integrate per second over the elapsed dt, rate is per hour
        let delta = math::checked_ceil_div(dt * abs_rate as u128, 3_600)?;

        if rate > 0 {
            Ok((
                self.virtual_funding_state
                    .cumulative_long_to_short
                    .to_u128()
```

## Proof of concept

Bounded arithmetic harnesses compare different update schedules for the same elapsed interval. They establish schedule-dependent index divergence, not economically material exploitation at production block times.

The bounded Kani model has borrow_interest_schedule_divergence_exists and funding_schedule_divergence_exists harnesses. formal/[project]-kani/FINDINGS.md:12-32 records divergence and bounded-divergence results. The final response says the per-call effect is sub-dust and not materially amplifiable at current block times. The model is not a production economic exploit.

## Relevant test excerpts

Source: `formal/[project]-kani/src/wave1.rs:323-340`

```rust
#[kani::proof]
fn borrow_interest_schedule_divergence_exists() {
    let custody = Custody {
        current_rate: 1,
        last_update: 0,
        cumulative_interest: 0,
        ..Default::default()
    };
    let one_shot = get_cumulative_interest(&custody, 2).unwrap();
    let custody_after_first = Custody {
        cumulative_interest: get_cumulative_interest(&custody, 1).unwrap(),
        last_update: 1,
        ..custody
    };
    let split_total = get_cumulative_interest(&custody_after_first, 2).unwrap();

    assert_ne!(split_total, one_shot);
}
```
