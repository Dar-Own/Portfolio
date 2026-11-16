# Evidence Index

The evidence category describes what is included or referenced in each case. Original harnesses are not distributed. A passing regression test is not a reproduction of the historical vulnerability; a model is not a deployed exploit.

| Case | Evidence category |
| --- | --- |
| [Public first-run setup gives ownership to the first visitor](findings/application-api/high/app-01-find-008-public-first-run-setup-gives-ownership-to-the-first-visitor.md) | Original test excerpts |
| [Manager can delegate unheld Controller authority to reveal employee SSNs](findings/application-api/high/app-01-find-056-manager-can-delegate-unheld-controller-authority-to-reveal-employee-ssns.md) | Original test excerpts |
| [Failed domain-event handlers are permanently acknowledged](findings/application-api/medium/app-01-find-001-failed-domain-event-handlers-are-permanently-acknowledged.md) | Partial proof / model |
| [Anonymous form submission overrides SMS STOP and relays attacker content](findings/application-api/medium/app-01-find-002-anonymous-form-submission-overrides-sms-stop-and-relays-attacker-content.md) | Original test excerpts |
| [Web Push redirects permit private-network requests](findings/application-api/medium/app-01-find-003-web-push-redirects-permit-private-network-requests.md) | Original test excerpts |
| [Worker crashes permanently strand checkout requests](findings/application-api/medium/app-01-find-005-worker-crashes-permanently-strand-checkout-requests.md) | Partial proof / model |
| [Unsigned webhooks consume durable storage before verification](findings/application-api/medium/app-01-find-006-unsigned-webhooks-consume-durable-storage-before-verification.md) | Original test excerpts |
| [Queued messages can target a former contact and create a new paperwork link](findings/application-api/medium/app-01-find-016-queued-messages-can-target-a-former-contact-and-create-a-new-paperwork-link.md) | Original test excerpts |
| [Queued invoice mail can go to a superseded recipient after corrected resend](findings/application-api/medium/app-01-find-017-queued-invoice-mail-can-go-to-a-superseded-recipient-after-corrected-resend.md) | Original test excerpts |
| [Old payment link stays valid after invoice recipient correction](findings/application-api/medium/app-01-find-018-old-payment-link-stays-valid-after-invoice-recipient-correction.md) | Original test excerpts |
| [Twilio number rotation drops old-number STOP and receipts](findings/application-api/medium/app-01-find-019-twilio-number-rotation-drops-old-number-stop-and-receipts.md) | Original test excerpts |
| [Stripe account switch can orphan an open checkout and lose its late payment event](findings/application-api/medium/app-01-find-020-stripe-account-switch-can-orphan-an-open-checkout-and-lose-its-late-payment-event.md) | Original test excerpts |
| [Acknowledged STOP callback is discarded after Auth Token rotation](findings/application-api/medium/app-01-find-023-acknowledged-stop-callback-is-discarded-after-auth-token-rotation.md) | Original test excerpts |
| [Bill reminder retains sensitive details after viewing rights are revoked](findings/application-api/medium/app-01-find-025-bill-reminder-retains-sensitive-details-after-viewing-rights-are-revoked.md) | Original test excerpts |
| [Encoded or compressed customer link survives shared-inbox removal](findings/application-api/medium/app-01-find-028-encoded-or-compressed-customer-link-survives-shared-inbox-removal.md) | Original test excerpts |
| [AI worker builds provider request after an owner turns attachment reading or summaries off](findings/application-api/medium/app-01-find-029-ai-worker-builds-provider-request-after-an-owner-turns-attachment-reading-or-summaries-off.md) | Original test excerpts |
| [Import commits a lead after import.run is revoked](findings/application-api/medium/app-01-find-031-import-commits-a-lead-after-import-run-is-revoked.md) | Original test excerpts |
| [Revoked staff session completes a pending platform read or write](findings/application-api/medium/app-01-find-033-revoked-staff-session-completes-a-pending-platform-read-or-write.md) | Original test excerpts |
| [Controller denied an employee's People view can retrieve that employee's full SSN](findings/application-api/medium/app-01-find-035-controller-denied-an-employee-s-people-view-can-retrieve-that-employee-s-full-ssn.md) | Original test excerpts |
| [Customer quote can sign after staff cancellation commits and revive canceled lead](findings/application-api/medium/app-01-find-037-customer-quote-can-sign-after-staff-cancellation-commits-and-revive-canceled-lead.md) | Original test excerpts |
| [Expense approval adds a cost to an already closed job](findings/application-api/medium/app-01-find-038-expense-approval-adds-a-cost-to-an-already-closed-job.md) | Original test excerpts |
| [Fixed worker scans silently skip later due records](findings/application-api/medium/app-01-find-039-fixed-worker-scans-silently-skip-later-due-records.md) | Original test excerpts |
| [Deposit draft edit bypasses estimate cap and queues an over-ceiling invoice](findings/application-api/medium/app-01-find-046-deposit-draft-edit-bypasses-estimate-cap-and-queues-an-over-ceiling-invoice.md) | Original test excerpts |
| [Default Foreman can add load tickets to another crew's job](findings/application-api/medium/app-01-find-047-default-foreman-can-add-load-tickets-to-another-crew-s-job.md) | Original test excerpts |
| [Nonfinancial equipment manager or assigned incident solver can enter maintenance cost](findings/application-api/medium/app-01-find-051-nonfinancial-equipment-manager-or-assigned-incident-solver-can-enter-maintenance-cost.md) | Original test excerpts |
| [Sub-job manager can offer and price a job whose linked lead is hidden](findings/application-api/medium/app-01-find-053-sub-job-manager-can-offer-and-price-a-job-whose-linked-lead-is-hidden.md) | Original test excerpts |
| [Import-history detail reveals notes from a lead the reader cannot open](findings/application-api/medium/app-01-find-055-import-history-detail-reveals-notes-from-a-lead-the-reader-cannot-open.md) | Original test excerpts |
| [Own-handler call audio and transcript bypass linked-lead visibility](findings/application-api/medium/app-01-find-057-own-handler-call-audio-and-transcript-bypass-linked-lead-visibility.md) | Original test excerpts |
| [Customer merge silently reassigns a hidden lead](findings/application-api/medium/app-01-find-059-customer-merge-silently-reassigns-a-hidden-lead.md) | Original test excerpts |
| [AI request includes company data after Ask permission is revoked](findings/application-api/medium/app-01-find-060-ai-request-includes-company-data-after-ask-permission-is-revoked.md) | Original test excerpts |
| [Call audio is handed off after transcription is disabled](findings/application-api/medium/app-01-find-061-call-audio-is-handed-off-after-transcription-is-disabled.md) | Original test excerpts |
| [Sub-job prices and margin disclosed without money permission](findings/application-api/medium/app-01-find-065-sub-job-prices-and-margin-disclosed-without-money-permission.md) | Original test excerpts |
| [Public setup validation amplifies memory use](findings/application-api/medium/app-01-n-07-public-setup-validation-amplifies-memory-use.md) | Regression tests |
| [Concurrent checkout requests bypass the five-request cap](findings/application-api/low/app-01-find-004-concurrent-checkout-requests-bypass-the-five-request-cap.md) | Original test excerpts |
| [Early Twilio receipts can be deleted before SID commit](findings/application-api/low/app-01-find-007-early-twilio-receipts-can-be-deleted-before-sid-commit.md) | Original test excerpts |
| [Forged reply headers attach mail to another connected account's thread](findings/application-api/low/app-01-find-009-forged-reply-headers-attach-mail-to-another-connected-account-s-thread.md) | Original test excerpts |
| [Queued AI reply can send after its authorization facts change](findings/application-api/low/app-01-find-011-queued-ai-reply-can-send-after-its-authorization-facts-change.md) | Original test excerpts |
| [Post-acceptance rollback can repeat outbound mail](findings/application-api/low/app-01-find-012-post-acceptance-rollback-can-repeat-outbound-mail.md) | Original test excerpts |
| [Old paperwork link serves superseded document after same-type replacement](findings/application-api/low/app-01-find-013-old-paperwork-link-serves-superseded-document-after-same-type-replacement.md) | Original test excerpts |
| [Canceled-job billing link still writes job data and emits an event](findings/application-api/low/app-01-find-014-canceled-job-billing-link-still-writes-job-data-and-emits-an-event.md) | Partial proof / model |
| [Restricted expense or support-ticket details appear in company activity to members denied the source object](findings/application-api/low/app-01-find-021-restricted-expense-or-support-ticket-details-appear-in-company-activity-to-members-denied.md) | Original test excerpts |
| [Alerts and digest identify hidden leads or tickets to members denied those objects](findings/application-api/low/app-01-find-022-alerts-and-digest-identify-hidden-leads-or-tickets-to-members-denied-those-objects.md) | Original test excerpts |
| [DB/file backup race restores a file pointer without ciphertext](findings/application-api/low/app-01-find-024-db-file-backup-race-restores-a-file-pointer-without-ciphertext.md) | Partial proof / model |
| [Queued internal alert sends after user disable or membership removal](findings/application-api/low/app-01-find-026-queued-internal-alert-sends-after-user-disable-or-membership-removal.md) | Original test excerpts |
| [One tenant's owner-configured slow SMTP delays another tenant's due mail](findings/application-api/low/app-01-find-027-one-tenant-s-owner-configured-slow-smtp-delays-another-tenant-s-due-mail.md) | Original test excerpts |
| [Job-comment route bypasses linked-lead visibility for a same-tenant member](findings/application-api/low/app-01-find-030-job-comment-route-bypasses-linked-lead-visibility-for-a-same-tenant-member.md) | Original test excerpts |
| [Suspended company messages remain eligible for delivery](findings/application-api/low/app-01-find-032-suspended-company-messages-remain-eligible-for-delivery.md) | Original test excerpts |
| [Replacing call audio can leave an unrelated transcript; delayed transcription can overwrite newer text](findings/application-api/low/app-01-find-034-replacing-call-audio-can-leave-an-unrelated-transcript-delayed-transcription-can-overwrite.md) | Partial proof / model |
| [Old mailbox results can file after connected account changes or disconnect](findings/application-api/low/app-01-find-036-old-mailbox-results-can-file-after-connected-account-changes-or-disconnect.md) | Original test excerpts |
| [A removed digest recipient receives a new digest after correction](findings/application-api/low/app-01-find-040-a-removed-digest-recipient-receives-a-new-digest-after-correction.md) | Original test excerpts |
| [Security invite alert is missed after the one-hour scan window](findings/application-api/low/app-01-find-041-security-invite-alert-is-missed-after-the-one-hour-scan-window.md) | Original test excerpts |
| [Mail nudge creates an open old-message task after staff completes, snoozes or replies](findings/application-api/low/app-01-find-042-mail-nudge-creates-an-open-old-message-task-after-staff-completes-snoozes-or-replies.md) | Original test excerpts |
| [Renewed company COI gains a new obsolete expiry task and alert after cleanup](findings/application-api/low/app-01-find-043-renewed-company-coi-gains-a-new-obsolete-expiry-task-and-alert-after-cleanup.md) | Original test excerpts |
| [Untouched-lead worker creates a false task/alert after staff already touched lead](findings/application-api/low/app-01-find-044-untouched-lead-worker-creates-a-false-task-alert-after-staff-already-touched-lead.md) | Original test excerpts |
| [Expense decision can finalize a revised amount/description without current-version review](findings/application-api/low/app-01-find-045-expense-decision-can-finalize-a-revised-amount-description-without-current-version-review.md) | Original test excerpts |
| [Queued Intake push creates estimating work after lead becomes Lost](findings/application-api/low/app-01-find-048-queued-intake-push-creates-estimating-work-after-lead-becomes-lost.md) | Original test excerpts |
| [Queued older quote email carries the revised quote PDF](findings/application-api/low/app-01-find-049-queued-older-quote-email-carries-the-revised-quote-pdf.md) | Original test excerpts |
| [Scheduled-date SMS can leave after the job is unassigned](findings/application-api/low/app-01-find-050-scheduled-date-sms-can-leave-after-the-job-is-unassigned.md) | Original test excerpts |
| [Out-of-service unit remains on a future equipment deployment](findings/application-api/low/app-01-find-052-out-of-service-unit-remains-on-a-future-equipment-deployment.md) | Original test excerpts |
| [Bill-blind sub-job manager can unlink an existing bill](findings/application-api/low/app-01-find-054-bill-blind-sub-job-manager-can-unlink-an-existing-bill.md) | Original test excerpts |
| [Foreman reads another crew's customer and site through Plants and Loads](findings/application-api/low/app-01-find-058-foreman-reads-another-crew-s-customer-and-site-through-plants-and-loads.md) | Original test excerpts |
| [End-of-day job creates a stale email task after final inbox triage](findings/application-api/low/app-01-find-062-end-of-day-job-creates-a-stale-email-task-after-final-inbox-triage.md) | Original test excerpts |
| [Cost-free repair correction leaves incident and maintenance costs inconsistent](findings/application-api/low/app-01-find-063-cost-free-repair-correction-leaves-incident-and-maintenance-costs-inconsistent.md) | Original test excerpts |
| [Invoice reminder quotes stale balance after partial payment](findings/application-api/low/app-01-find-064-invoice-reminder-quotes-stale-balance-after-partial-payment.md) | Original test excerpts |
| [Cap-refused push claim deletes another tenant's subscription](findings/application-api/low/app-01-find-066-cap-refused-push-claim-deletes-another-tenant-s-subscription.md) | Original test excerpts |
| [Customer-link redaction removes unrelated reference codes](findings/application-api/low/app-01-n-01-customer-link-redaction-removes-unrelated-reference-codes.md) | Regression tests |
| [Task-owner visibility guard conflicts with routed recipients](findings/application-api/low/app-01-n-02-task-owner-visibility-guard-conflicts-with-routed-recipients.md) | Recorded observation |
| [Logical restore suppresses history-based security alerts](findings/application-api/low/app-01-n-03-logical-restore-suppresses-history-based-security-alerts.md) | Regression tests |
| [Database disconnect during enrolment SMTP crashes the worker](findings/application-api/low/app-01-n-04-database-disconnect-during-enrolment-smtp-crashes-the-worker.md) | Regression tests |
| [Cost-restricted sub-job manager loses the Notes editor](findings/application-api/low/app-01-n-05-cost-restricted-sub-job-manager-loses-the-notes-editor.md) | Regression tests |
| [Explicit payment date shifts to the previous company day](findings/application-api/low/app-01-n-06-explicit-payment-date-shifts-to-the-previous-company-day.md) | Regression tests |
| [Customer timeline reads spend seconds compiling query plans](findings/application-api/low/app-01-n-08-customer-timeline-reads-spend-seconds-compiling-query-plans.md) | Regression tests |
| [Delisted tracked assets make ordinary redemptions skip accrued fund fees](findings/asset-management/high/am-02-fee-hook.md) | Original test excerpts |
| [Prototype AMM adapters violate withdrawal and balance interfaces](findings/asset-management/high-future/am-01-f-01-amm-adapters-violate-project-strategy-deposit-and-withdrawal-semantics.md) | Partial proof / model |
| [Spot-reserve valuation permits modeled extraction of idle liquidity](findings/asset-management/high-future/am-01-f-02-manipulable-amm-reserves-can-inflate-xycloans-nav-and-drain-idle-liquidity.md) | Partial proof / model |
| [Stale strategy NAV shifts an upstream loss to remaining vault holders](findings/asset-management/medium/am-01-m-01-stale-strategy-nav-shifts-an-upstream-loss-to-remaining-vault-holders.md) | Original test excerpts |
| [Public invest-on-deposit deletes negative strategy loss carry](findings/asset-management/medium/am-01-m-02-public-invest-on-deposit-deletes-negative-strategy-loss-carry.md) | Original test excerpts |
| [Incompatible fee receiver blocks management and incident response](findings/asset-management/medium/am-01-m-03-incompatible-fee-receiver-blocks-management-and-incident-response.md) | Recorded observation |
| [Refresh tokens remain reusable after refresh and concurrent replay](findings/asset-management/medium/am-01-m-04-refresh-tokens-remain-reusable-after-refresh-and-concurrent-replay.md) | Recorded observation |
| [Unauthenticated quote/checkpoint mutates report and fee state](findings/asset-management/low/am-01-l-01-unauthenticated-quote-checkpoint-mutates-report-and-fee-state.md) | Original test excerpts |
| [Persistent API-origin override forwards bearer credentials](findings/asset-management/low/am-01-l-02-frameable-console-persists-an-arbitrary-api-origin-across-sessions.md) | Source / configuration |
| [Legacy zero commit timestamps bypass the configured restart halt deadline](findings/blockchain-infrastructure/medium/chain-01-halt-time.md) | Original test excerpts |
| [A third-party opener installs withdrawal authority on a victim-owned position](findings/collateralized-lending/medium/lend-02-af-02.md) | Original test excerpts |
| [Failed-message refunds bypass ordinary execution limits and accounting](findings/cross-chain-bridges/unrated/bridge-01-failed-message-refund.md) | Original test excerpts |
| [The allowKick state suppresses public epoch closure after debt repayment](findings/epoch-lending/critical-conflicted/lend-01-p1.md) | Original test excerpts |
| [Stale repayment credit permits an underfunded later epoch to settle](findings/epoch-lending/critical-conflicted/lend-01-p2.md) | Original test excerpts |
| [Period-start checkpoint replacement suppresses a valid coverage payout](findings/insurance-vaults/medium/ins-01-rl-001.md) | Original test excerpts |
| [Failed `ownerOf` is accepted as proof that a live token burned](findings/nft-rewards/medium/nft-01-lhf-01-failed-ownerof-is-accepted-as-proof-that-a-live-token-burned.md) | Original test excerpts |
| [Positive compound proceeds can be impossible to book honestly with zero principal](findings/nft-rewards/medium-low/nft-01-lhf-03-positive-compound-proceeds-can-be-impossible-to-book-honestly-with-zero-principal.md) | Original test excerpts |
| [The indexer does not bind its configured Lens to its configured Core](findings/nft-rewards/medium-low/nft-01-lhf-09-the-indexer-does-not-bind-its-configured-lens-to-its-configured-core.md) | Source / configuration |
| [`canReportCompound()` approves a write that later reverts](findings/nft-rewards/low/nft-01-lhf-02-canreportcompound-approves-a-write-that-later-reverts.md) | Original test excerpts |
| [Corrections after deactivation understate dead liability](findings/nft-rewards/low/nft-01-lhf-04-corrections-after-deactivation-understate-dead-liability.md) | Source / configuration |
| [Activation price accepts a wider type than NFT payment storage](findings/nft-rewards/low/nft-01-lhf-05-activation-price-accepts-a-wider-type-than-nft-payment-storage.md) | Source / configuration |
| [Valid principal can exceed the signed lifecycle-event domain](findings/nft-rewards/low/nft-01-lhf-06-valid-principal-can-exceed-the-signed-lifecycle-event-domain.md) | Partial proof / model |
| [`pendingWork()` can recommend an impossible batch report](findings/nft-rewards/low/nft-01-lhf-07-pendingwork-can-recommend-an-impossible-batch-report.md) | Source / configuration |
| [Reconciliation failure does not make the indexer unhealthy](findings/nft-rewards/low/nft-01-lhf-08-reconciliation-failure-does-not-make-the-indexer-unhealthy.md) | Source / configuration |
| [The app ignores the indexer’s direct Core identity](findings/nft-rewards/low/nft-01-lhf-10-the-app-ignores-the-indexer-s-direct-core-identity.md) | Source / configuration |
| [`canRetireCollection()` omits the membership-list check](findings/nft-rewards/low/nft-01-p2-01-canretirecollection-omits-the-membership-list-check.md) | Source / configuration |
| [`reportBatch()` forgives activation-wallet shortfall silently](findings/nft-rewards/low/nft-01-p2-02-reportbatch-forgives-activation-wallet-shortfall-silently.md) | Source / configuration |
| [Saturating dead counters allow cross-level drift](findings/nft-rewards/low/nft-01-p2-03-saturating-dead-counters-allow-cross-level-drift.md) | Source / configuration |
| [Configured selector routes can be silently shadowed by Core dispatch](findings/nft-rewards/informational-low/nft-01-lhf-11-configured-selector-routes-can-be-silently-shadowed-by-core-dispatch.md) | Source / configuration |
| [`deactivateBurned` and `sweepExcess` are outside every pause domain](findings/nft-rewards/informational-low/nft-01-p2-04-deactivateburned-and-sweepexcess-are-outside-every-pause-domain.md) | Source / configuration |
| [Move or revive inside a correction window forfeits the correction](findings/nft-rewards/informational/nft-01-p2-05-move-or-revive-inside-a-correction-window-forfeits-the-correction.md) | Source / configuration |
| [Collection code-change detection remains advisory](findings/nft-rewards/trust-dependent-high/nft-01-lhf-12-collection-code-change-detection-remains-advisory.md) | Source / configuration |
| [Missing Timelock on Admin Operations with single step transfer](findings/perpetuals/critical/perp-01-c1-missing-timelock-on-admin-operations-with-single-step-transfer.md) | Original test excerpts |
| [VFR Funding Obligations Invisible to AUM](findings/perpetuals/high/perp-01-h1-vfr-funding-obligations-invisible-to-aum.md) | Partial proof / model |
| [Oracle guarantees degrade under staleness](findings/perpetuals/high/perp-01-h2-oracle-guarantees-degrade-under-staleness.md) | Partial proof / model |
| [Switchboard Oracle Staleness Laundering](findings/perpetuals/high/perp-01-h3-switchboard-oracle-staleness-laundering.md) | Partial proof / model |
| [Governance Inflation: Resolved Locked Stakes Recount on Sync](findings/perpetuals/medium/perp-01-m1-governance-inflation-resolved-locked-stakes-recount-on-sync.md) | Partial proof / model |
| [Pixel-Perfect Liquidation via Stale Oracle Price](findings/perpetuals/medium/perp-01-m2-pixel-perfect-liquidation-via-stale-oracle-price.md) | Original test excerpts |
| [PnL Uses Midpoint Instead of Protective Confidence Bands](findings/perpetuals/medium/perp-01-m3-pnl-uses-midpoint-instead-of-protective-confidence-bands.md) | Original test excerpts |
| [Account revival: closed position PDAs can be zombified](findings/perpetuals/medium/perp-01-m4-account-revival-closed-position-pdas-can-be-zombified.md) | Original test excerpts |
| [Autonom Market-Opening Signatures Use Ambiguous Encoding](findings/perpetuals/medium/perp-01-m5-autonom-market-opening-signatures-use-ambiguous-encoding.md) | Partial proof / model |
| [Underweight Oracle Consensus](findings/perpetuals/medium/perp-01-m6-underweight-oracle-consensus.md) | Partial proof / model |
| [Incorrect exponent handling](findings/perpetuals/medium/perp-01-m7-incorrect-exponent-handling.md) | Partial proof / model |
| [Missing Slippage Enforcement on Open Position](findings/perpetuals/medium/perp-01-m8-missing-slippage-enforcement-on-open-position.md) | Original test excerpts |
| [Schedule-Dependent Interest and Funding Inflation via Ceiling Division Rounding](findings/perpetuals/low/perp-01-l1-schedule-dependent-interest-and-funding-inflation-via-ceiling-division-rounding.md) | Partial proof / model |
| [Griefing: Deprecated Permissionless IDL](findings/perpetuals/low/perp-01-l10-griefing-deprecated-permissionless-idl.md) | Partial proof / model |
| [LP Sandwich: No Structural Hold Period](findings/perpetuals/low/perp-01-l11-lp-sandwich-no-structural-hold-period.md) | Partial proof / model |
| [Wrapping Overflow in Financial Calculation](findings/perpetuals/low/perp-01-l12-wrapping-overflow-in-financial-calculation.md) | Partial proof / model |
| [Hardcoded RPC Authentication Tokens in Source](findings/perpetuals/low/perp-01-l2-hardcoded-rpc-authentication-tokens-in-source.md) | Source / configuration |
| [DOS: Residual Unchecked Oracle Math](findings/perpetuals/low/perp-01-l3-dos-residual-unchecked-oracle-math.md) | Partial proof / model |
| [Griefing: Pool-Wide Write-Lock](findings/perpetuals/low/perp-01-l4-griefing-pool-wide-write-lock.md) | Partial proof / model |
| [Griefing: Liquidation Capture](findings/perpetuals/low/perp-01-l5-griefing-liquidation-capture.md) | Partial proof / model |
| [Update-Time Lock Traps User After add_collateral](findings/perpetuals/low/perp-01-l6-update-time-lock-traps-user-after-add-collateral.md) | Original test excerpts |
| [Governance Lacks Participation Quorum and Execution Timelock](findings/perpetuals/low/perp-01-l7-governance-lacks-participation-quorum-and-execution-timelock.md) | Source / configuration |
| [Fee Distribution: Off-Peg Stablecoin Creates Phantom Pool Value](findings/perpetuals/low/perp-01-l8-fee-distribution-off-peg-stablecoin-creates-phantom-pool-value.md) | Original test excerpts |
| [Feeless Swap Skips Borrow Rate Update](findings/perpetuals/low/perp-01-l9-feeless-swap-skips-borrow-rate-update.md) | Original test excerpts |
| [Pre-loss deposits capture recovery allocations and recycle capital](findings/prediction-market-lending/high/pm-01-n4-pre-loss-deposit-recovery-capture.md) | Original test excerpts |
| [Single-controller custody and self-reported accounting](findings/prediction-market-lending/high/pm-01-v1-single-controller-custody-and-accounting.md) | Original test excerpts |
| [Superseded signed prices remain actionable inside the freshness window](findings/prediction-market-lending/high/pm-01-v3-superseded-signed-marks.md) | Original test excerpts |
| [Public quote consumption can block renewal until collateral forfeiture](findings/prediction-market-lending/medium-high/pm-01-v4-public-quote-consumption-forced-expiry.md) | Original test excerpts |
| [Executor outage blocks leverage-position exits](findings/prediction-market-lending/medium/pm-01-l1-executor-outage-blocks-exits.md) | Original test excerpts |
| [Large redemption queues block renewal and participate in expiry recovery](findings/prediction-market-lending/medium/pm-01-n2-queue-blocks-renewal-and-captures-recovery.md) | Original test excerpts |
| [Five-minute governance delay for critical policy changes](findings/prediction-market-lending/medium/pm-01-v2-short-governance-timelock.md) | Source / configuration |
| [Zap conversion lacks a caller minimum-output bound](findings/prediction-market-lending/low/pm-01-v5-l4-zap-minimum-output-gap.md) | Source / configuration |
| [Queued redemption cannot be canceled while awaiting liquidity](findings/prediction-market-lending/low/pm-01-v6-noncancelable-redemption-queue.md) | Partial proof / model |
| [Pro-rata recovery rounding leaves protected reserved dust](findings/prediction-market-lending/informational/pm-01-l3-recovery-rounding-dust.md) | Source / configuration |
| [Future-dated signed marks bypass the intended freshness horizon](findings/prediction-market-lending/unrated/pm-01-n1-future-dated-marks.md) | Original test excerpts |
| [earnedId replay drains the shared reward bucket](findings/rewards-governance/critical/rg-01-c1-earnedid-replay-drains-the-shared-reward-bucket.md) | Original test excerpts |
| [Trustline gate ships fail-open / never enabled](findings/rewards-governance/high/rg-01-h1-trustline-gate-ships-fail-open-never-enabled.md) | Source / configuration |
| [Captured pool can mint full poolCap without entitlement checks](findings/rewards-governance/high/rg-01-h2-captured-pool-can-mint-full-poolcap-without-entitlement-checks.md) | Original test excerpts |
| [Council vote burning can permanently disable governance](findings/rewards-governance/high/rg-01-h3-council-vote-burning-can-permanently-disable-governance.md) | Source / configuration |
| [Council initialization can fail open below governance quorum](findings/rewards-governance/high/rg-01-h4-council-initialization-can-fail-open-below-governance-quorum.md) | Source / configuration |
| [poolCap unenforced at allocation strands rewards](findings/rewards-governance/medium/rg-01-m1-poolcap-unenforced-at-allocation-strands-rewards.md) | Original test excerpts |
| [Governance capture is cheap](findings/rewards-governance/medium/rg-01-m2-governance-capture-is-cheap.md) | Partial proof / model |
| [Deployer retains [project]CouncilToken.DEFAULT_ADMIN_ROLE](findings/rewards-governance/medium/rg-01-m3-deployer-retains-project-counciltoken-default-admin-role.md) | Partial proof / model |
| [TIMELOCK_DELAY_SECONDS=0 is accepted](findings/rewards-governance/medium/rg-01-m4-timelock-delay-seconds-0-is-accepted.md) | Partial proof / model |
| [Deployer EOA owns the Trustline engines](findings/rewards-governance/medium/rg-01-m5-deployer-eoa-owns-the-trustline-engines.md) | Source / configuration |
| [updateAndAdjust missing self-scope](findings/rewards-governance/medium/rg-01-m6-updateandadjust-missing-self-scope.md) | Original test excerpts |
| [One pool reserves global headroom](findings/rewards-governance/medium/rg-01-m7-one-pool-reserves-global-headroom.md) | Original test excerpts |
| [Lifetime poolCap permanently bricks pools](findings/rewards-governance/medium/rg-01-m8-lifetime-poolcap-permanently-bricks-pools.md) | Original test excerpts |
| [Reverting Trustline engine bricks claims](findings/rewards-governance/medium/rg-01-m9-reverting-trustline-engine-bricks-claims.md) | Original test excerpts |
| [Timelock lacks AUTHORIZED_PARTY_ROLE on minter](findings/rewards-governance/low/rg-01-l1-timelock-lacks-authorized-party-role-on-minter.md) | Partial proof / model |
| [Unbounded reward arrays](findings/rewards-governance/low/rg-01-l2-unbounded-reward-arrays.md) | Source / configuration |
| [Pools have no independent pause](findings/rewards-governance/low/rg-01-l3-pools-have-no-independent-pause.md) | Source / configuration |
| [Delegated backend is screened](findings/rewards-governance/low/rg-01-m10-delegated-backend-is-screened.md) | Original test excerpts |
| [Merkle root reusable across uuidHash](findings/rewards-governance/low/rg-01-m11-merkle-root-reusable-across-uuidhash.md) | Original test excerpts |
| [Swap redemptions exhaust an outer-vault quota without consuming its liquidity](findings/tokenized-assets/medium/rwa-01-m02.md) | Original test excerpts |
