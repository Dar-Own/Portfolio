# Source-code-based

Project: Anonymous blockchain infrastructure project (CHAIN-01)  
Category: Blockchain infrastructure  
Severity: Medium — conservative source-assigned upgrade-safety rating; default-off scope boundary unresolved  
Engagement: Bug bounty hunting (scope determination/private upgrade-safety disclosure)  
Chain: Berachain — local four-validator consensus/execution devnet; production deployment not established  
Tech stack: Go consensus client, CometBFT commit metadata, Rust execution client, shell/jq local-devnet harness  
Evidence: Historical local or fork test excerpts; scope and fixture assumptions are described below  
Remediation: Not independently verified  
Contribution: Research and disclosure case study; exclusive discovery credit is not asserted  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# Legacy zero commit timestamps bypass the configured restart halt deadline

## Description

An operator restarts validators with an expired configured halt-time after a rolling upgrade. Validators at the same committed height and app hash can carry different local commit timestamps: current binaries populate them, older binaries leave the zero time. The startup predicate therefore allows only the legacy-metadata side to participate after the deadline.

The all-zero control commits exactly one H+1 then refuses the next restart; this is not indefinite post-halt production. Equal-power 2/2 mixed metadata stalls before H+1 in three recorded clean runs. No external attacker, funds loss, app-hash fork, slashable double-signing or mainnet effect was established. The option is disabled by default and the program excludes default-off DoS; eligibility is not self-certified. The 67% weighted result was not reproduced cleanly and is excluded.

### Mechanism step by step

#### Step 1 — Seed the halt decision from local commit metadata

The service reads CommitInfo at the last block and copies its timestamp verbatim. Zero is not rejected for a non-genesis height.

Source: `consensus/cometbft/service/service.go:212-230`

```go
func (s *Service) seedFinalizedBlock(lastBlockHeight int64) {
	s.finalizedHeight = lastBlockHeight
	if lastBlockHeight <= 0 || (s.haltHeight == 0 && s.haltTime == 0) {
		return
	}

	rms, ok := s.sm.GetCommitMultiStore().(*rootmulti.Store)
	if !ok {
		panic("failed loading last committed block time: unexpected commit multi-store type")
	}

	ci, ciErr := rms.GetCommitInfo(lastBlockHeight)
	if ciErr != nil {
		panic(fmt.Errorf("failed loading commit info at height %d: %w", lastBlockHeight, ciErr))
	}
	if ci == nil {
		panic(fmt.Errorf("failed loading commit info at height %d: empty commit info", lastBlockHeight))
	}
	s.finalizedTime = ci.Timestamp
```

#### Step 2 — Treat zero time as not reaching the deadline

Go time.Time{} has negative Unix time, so unixTime>=0 is false and the configured halt-time case is skipped.

Source: `consensus/cometbft/service/commit.go:70-80`

```go
func haltPointReached(haltHeight, haltTime uint64, height int64, blockTime time.Time) bool {
	unixTime := blockTime.Unix()
	switch {
	case haltHeight > 0 && height >= 0 && uint64(height) >= haltHeight:
		return true
	case haltTime > 0 && unixTime >= 0 && uint64(unixTime) >= haltTime:
		return true
	default:
		return false
	}
}
```

#### Step 3 — Permit startup only on the legacy-metadata validators

ensureNotHalted returns nil for that side while peers with real post-deadline timestamps refuse. The same predicate gates block processing.

Source: `consensus/cometbft/service/commit.go:85-92`

```go
func (s *Service) ensureNotHalted() error {
	if !haltPointReached(s.haltHeight, s.haltTime, s.finalizedHeight, s.finalizedTime) {
		return nil
	}
	return fmt.Errorf(
		"chain reached the configured halt point (halt-height %d, halt-time %d) at committed height %d, unset the halt flags to resume",
		s.haltHeight, s.haltTime, s.finalizedHeight,
	)
```

## Proof of concept

The scripts `reproduce_post_halt_block.sh` and `reproduce.sh` build old/current binaries, create a four-validator devnet and restart validators with an expired halt-time. The natural-metadata scenario uses no database mutation. Its predicate requires equal committed height and app hash, mixed local timestamps, and no H+1 commit. The historical equal-power 2/2 scenario stalled in three clean runs. The all-zero control commits exactly one additional block, then rejects the next restart. This is a local upgrade/configuration failure; no external attacker, mainnet fork or slashing result was established.

**Proof limits:** Default-off halt configuration and mixed binary history; local devnet liveness only.

The excerpts below preserve historical assertions and recorded outcomes; they are not a fresh execution result.

## Relevant test excerpts

Source: `../[project]/report-preparation/immunefi-halt-time-poc/reproduce.sh:418-432`

```bash
			quorum_loss: (($mixed_reached == true) and ($same_height == true) and ($same_hash == true) and ($hplus1 == 0) and (($reject_logs > 0) or ($startup_refusal_logs > 0))),
			divergence: false,
			repeatable_bypass: false,
			control_passed: $control_passed,
			halt_time: $halt_time,
			zero_timestamp_validators: $zero_count,
			nonzero_timestamp_validators: $nonzero_count,
			same_height: $same_height,
			same_app_hash: $same_hash,
			mixed_timestamp_reached: $mixed_reached,
			hplus1_persisted_validators: $hplus1,
			hplus2_persisted_validators: $hplus2,
			hplus1_commit_log_count: $hplus1_logs,
			expired_halt_reject_log_count: $reject_logs,
			expired_halt_startup_refusal_log_count: $startup_refusal_logs,
```

Source: `../[project]/recon/progress/halt-time-escalation-results.md:48-54`

```text
## 4. Mixed-Metadata Results

| Run | Mixed reached | Same height | Same app hash | Zero / Non-zero | H+1 persisted | H+2 persisted | Startup refusals | Round timeouts | Control | Verdict |
| --- | --- | --- | --- | ---: | ---: | ---: | ---: | ---: | --- | --- |
| run-01 | true | true | true | 2/2 | 0 | 0 | 6 | 4 | true | quorum loss before H+1 |
| run-02 | true | true | true | 2/2 | 0 | 0 | 6 | 4 | true | quorum loss before H+1 |
| run-03 | true | true | true | 2/2 | 0 | 0 | 6 | 4 | true | quorum loss before H+1 |
```
