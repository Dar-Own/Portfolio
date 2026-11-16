# Security Research Portfolio

Technical case studies covering smart contracts, financial accounting, application security and blockchain infrastructure. Each case explains the mechanism, prerequisites, demonstrated result and limits of the evidence.

Project names are redacted. Original source and test excerpts are retained, so distinctive code, filenames and mechanisms may still identify a project. Redaction markers are explicit; some displayed excerpts will not compile with those markers in place. Original source comments are retained with the code; editorial work logs and review instructions are omitted.

## Case studies

This collection contains **159 case studies**: **141 vulnerability records**, **one informational observation**, and **17 research/hardening records**. Severity reflects the original assessment and retains conditional, disputed and future-deployment qualifications. Historical remediation does not erase a finding. Team engagement findings are not presented as sole discoveries.

Browse by category and severity in the index below. Each case includes the chain, technology stack and engagement type.

## Proof coverage

This is a **case-study repository, not a runnable test suite**. It includes selected original test code and source references. The original applications, dependencies, fixtures and complete harnesses are not distributed here. Source references identify the original files, not additional files included in this repository.

The presence of a test assertion does not establish a successful execution or prove every claimed consequence. Each case states what the available evidence supports. Regression tests describe corrected behavior; models and partial proofs retain their assumptions. Source-only findings remain labeled as such.

| Evidence available | Cases | Meaning |
| --- | ---: | --- |
| Original test excerpts | 97 | Original finding-related test code is shown; consult the case for execution evidence and scope. |
| Partial proof / model | 27 | The proof establishes a narrower property, uses a model, or leaves a required condition unproved. |
| Regression tests | 7 | The included test exercises corrected behavior, not the original failure. |
| Source / configuration | 25 | Source or configuration supports the claim; no dedicated executable exploit proof is presented. |
| Recorded observation | 3 | The case relies on retained observations or simulation results rather than a distributed harness. |

See the [evidence index](EVIDENCE.md) for each case’s proof category. The counts describe evidence coverage, not accepted bounties or independently reproduced exploits.

## Index

### Application Api

| Severity | Case study | Evidence |
| --- | --- | --- |
| High | [Public first-run setup gives ownership to the first visitor](findings/application-api/high/app-01-find-008-public-first-run-setup-gives-ownership-to-the-first-visitor.md) | Original test excerpts |
| High (same-tenant confidentiality/authority) | [Manager can delegate unheld Controller authority to reveal employee SSNs](findings/application-api/high/app-01-find-056-manager-can-delegate-unheld-controller-authority-to-reveal-employee-ssns.md) | Original test excerpts |
| Medium | [Failed domain-event handlers are permanently acknowledged](findings/application-api/medium/app-01-find-001-failed-domain-event-handlers-are-permanently-acknowledged.md) | Partial proof / model |
| Medium | [Anonymous form submission overrides SMS STOP and relays attacker content](findings/application-api/medium/app-01-find-002-anonymous-form-submission-overrides-sms-stop-and-relays-attacker-content.md) | Original test excerpts |
| Medium | [Web Push redirects permit private-network requests](findings/application-api/medium/app-01-find-003-web-push-redirects-permit-private-network-requests.md) | Original test excerpts |
| Medium | [Worker crashes permanently strand checkout requests](findings/application-api/medium/app-01-find-005-worker-crashes-permanently-strand-checkout-requests.md) | Partial proof / model |
| Medium | [Unsigned webhooks consume durable storage before verification](findings/application-api/medium/app-01-find-006-unsigned-webhooks-consume-durable-storage-before-verification.md) | Original test excerpts |
| Medium (conditional document-confidentiality variant) | [Queued messages can target a former contact and create a new paperwork link](findings/application-api/medium/app-01-find-016-queued-messages-can-target-a-former-contact-and-create-a-new-paperwork-link.md) | Original test excerpts |
| Medium (conditional) | [Queued invoice mail can go to a superseded recipient after corrected resend](findings/application-api/medium/app-01-find-017-queued-invoice-mail-can-go-to-a-superseded-recipient-after-corrected-resend.md) | Original test excerpts |
| Medium (conditional) | [Old payment link stays valid after invoice recipient correction](findings/application-api/medium/app-01-find-018-old-payment-link-stays-valid-after-invoice-recipient-correction.md) | Original test excerpts |
| Medium (conditional) | [Twilio number rotation drops old-number STOP and receipts](findings/application-api/medium/app-01-find-019-twilio-number-rotation-drops-old-number-stop-and-receipts.md) | Original test excerpts |
| Medium (conditional) | [Stripe account switch can orphan an open checkout and lose its late payment event](findings/application-api/medium/app-01-find-020-stripe-account-switch-can-orphan-an-open-checkout-and-lose-its-late-payment-event.md) | Original test excerpts |
| Medium (conditional) | [Acknowledged STOP callback is discarded after Auth Token rotation](findings/application-api/medium/app-01-find-023-acknowledged-stop-callback-is-discarded-after-auth-token-rotation.md) | Original test excerpts |
| Medium (conditional) | [Bill reminder retains sensitive details after viewing rights are revoked](findings/application-api/medium/app-01-find-025-bill-reminder-retains-sensitive-details-after-viewing-rights-are-revoked.md) | Original test excerpts |
| Medium (conditional) | [Encoded or compressed customer link survives shared-inbox removal](findings/application-api/medium/app-01-find-028-encoded-or-compressed-customer-link-survives-shared-inbox-removal.md) | Original test excerpts |
| Medium (conditional) | [AI worker builds provider request after an owner turns attachment reading or summaries off](findings/application-api/medium/app-01-find-029-ai-worker-builds-provider-request-after-an-owner-turns-attachment-reading-or-summaries-off.md) | Original test excerpts |
| Medium (conditional) | [Import commits a lead after import.run is revoked](findings/application-api/medium/app-01-find-031-import-commits-a-lead-after-import-run-is-revoked.md) | Original test excerpts |
| Medium (conditional) | [Revoked staff session completes a pending platform read or write](findings/application-api/medium/app-01-find-033-revoked-staff-session-completes-a-pending-platform-read-or-write.md) | Original test excerpts |
| Medium | [Controller denied an employee's People view can retrieve that employee's full SSN](findings/application-api/medium/app-01-find-035-controller-denied-an-employee-s-people-view-can-retrieve-that-employee-s-full-ssn.md) | Original test excerpts |
| Medium (conditional) | [Customer quote can sign after staff cancellation commits and revive canceled lead](findings/application-api/medium/app-01-find-037-customer-quote-can-sign-after-staff-cancellation-commits-and-revive-canceled-lead.md) | Original test excerpts |
| Medium (conditional) | [Expense approval adds a cost to an already closed job](findings/application-api/medium/app-01-find-038-expense-approval-adds-a-cost-to-an-already-closed-job.md) | Original test excerpts |
| Medium (conditional) | [Fixed worker scans silently skip later due records](findings/application-api/medium/app-01-find-039-fixed-worker-scans-silently-skip-later-due-records.md) | Original test excerpts |
| Medium (conditional integrity) | [Deposit draft edit bypasses estimate cap and queues an over-ceiling invoice](findings/application-api/medium/app-01-find-046-deposit-draft-edit-bypasses-estimate-cap-and-queues-an-over-ceiling-invoice.md) | Original test excerpts |
| Medium (operational integrity) | [Default Foreman can add load tickets to another crew's job](findings/application-api/medium/app-01-find-047-default-foreman-can-add-load-tickets-to-another-crew-s-job.md) | Original test excerpts |
| Medium (same-tenant financial-record integrity) | [Nonfinancial equipment manager or assigned incident solver can enter maintenance cost](findings/application-api/medium/app-01-find-051-nonfinancial-equipment-manager-or-assigned-incident-solver-can-enter-maintenance-cost.md) | Original test excerpts |
| Medium (same-tenant process/pricing integrity) | [Sub-job manager can offer and price a job whose linked lead is hidden](findings/application-api/medium/app-01-find-053-sub-job-manager-can-offer-and-price-a-job-whose-linked-lead-is-hidden.md) | Original test excerpts |
| Medium (same-tenant confidentiality) | [Import-history detail reveals notes from a lead the reader cannot open](findings/application-api/medium/app-01-find-055-import-history-detail-reveals-notes-from-a-lead-the-reader-cannot-open.md) | Original test excerpts |
| Medium (same-tenant confidentiality) | [Own-handler call audio and transcript bypass linked-lead visibility](findings/application-api/medium/app-01-find-057-own-handler-call-audio-and-transcript-bypass-linked-lead-visibility.md) | Original test excerpts |
| Medium (same-tenant integrity) | [Customer merge silently reassigns a hidden lead](findings/application-api/medium/app-01-find-059-customer-merge-silently-reassigns-a-hidden-lead.md) | Original test excerpts |
| Medium (conditional provider-bound confidentiality) | [AI request includes company data after Ask permission is revoked](findings/application-api/medium/app-01-find-060-ai-request-includes-company-data-after-ask-permission-is-revoked.md) | Original test excerpts |
| Medium (conditional provider-bound confidentiality) | [Call audio is handed off after transcription is disabled](findings/application-api/medium/app-01-find-061-call-audio-is-handed-off-after-transcription-is-disabled.md) | Original test excerpts |
| Medium (same-tenant financial confidentiality) | [Sub-job prices and margin disclosed without money permission](findings/application-api/medium/app-01-find-065-sub-job-prices-and-margin-disclosed-without-money-permission.md) | Original test excerpts |
| Medium | [Public setup validation amplifies memory use](findings/application-api/medium/app-01-n-07-public-setup-validation-amplifies-memory-use.md) | Regression tests |
| Low | [Concurrent checkout requests bypass the five-request cap](findings/application-api/low/app-01-find-004-concurrent-checkout-requests-bypass-the-five-request-cap.md) | Original test excerpts |
| Low | [Early Twilio receipts can be deleted before SID commit](findings/application-api/low/app-01-find-007-early-twilio-receipts-can-be-deleted-before-sid-commit.md) | Original test excerpts |
| Low (provisional) | [Forged reply headers attach mail to another connected account's thread](findings/application-api/low/app-01-find-009-forged-reply-headers-attach-mail-to-another-connected-account-s-thread.md) | Original test excerpts |
| Low (conditional) | [Queued AI reply can send after its authorization facts change](findings/application-api/low/app-01-find-011-queued-ai-reply-can-send-after-its-authorization-facts-change.md) | Original test excerpts |
| Low (conditional) | [Post-acceptance rollback can repeat outbound mail](findings/application-api/low/app-01-find-012-post-acceptance-rollback-can-repeat-outbound-mail.md) | Original test excerpts |
| Low | [Old paperwork link serves superseded document after same-type replacement](findings/application-api/low/app-01-find-013-old-paperwork-link-serves-superseded-document-after-same-type-replacement.md) | Original test excerpts |
| Low | [Canceled-job billing link still writes job data and emits an event](findings/application-api/low/app-01-find-014-canceled-job-billing-link-still-writes-job-data-and-emits-an-event.md) | Partial proof / model |
| Low | [Restricted expense or support-ticket details appear in company activity to members denied the source object](findings/application-api/low/app-01-find-021-restricted-expense-or-support-ticket-details-appear-in-company-activity-to-members-denied.md) | Original test excerpts |
| Low | [Alerts and digest identify hidden leads or tickets to members denied those objects](findings/application-api/low/app-01-find-022-alerts-and-digest-identify-hidden-leads-or-tickets-to-members-denied-those-objects.md) | Original test excerpts |
| Low (conditional) | [DB/file backup race restores a file pointer without ciphertext](findings/application-api/low/app-01-find-024-db-file-backup-race-restores-a-file-pointer-without-ciphertext.md) | Partial proof / model |
| Low (conditional) | [Queued internal alert sends after user disable or membership removal](findings/application-api/low/app-01-find-026-queued-internal-alert-sends-after-user-disable-or-membership-removal.md) | Original test excerpts |
| Low (conditional) | [One tenant's owner-configured slow SMTP delays another tenant's due mail](findings/application-api/low/app-01-find-027-one-tenant-s-owner-configured-slow-smtp-delays-another-tenant-s-due-mail.md) | Original test excerpts |
| Low | [Job-comment route bypasses linked-lead visibility for a same-tenant member](findings/application-api/low/app-01-find-030-job-comment-route-bypasses-linked-lead-visibility-for-a-same-tenant-member.md) | Original test excerpts |
| Low (conditional) | [Suspended company messages remain eligible for delivery](findings/application-api/low/app-01-find-032-suspended-company-messages-remain-eligible-for-delivery.md) | Original test excerpts |
| Low (conditional) | [Replacing call audio can leave an unrelated transcript; delayed transcription can overwrite newer text](findings/application-api/low/app-01-find-034-replacing-call-audio-can-leave-an-unrelated-transcript-delayed-transcription-can-overwrite.md) | Partial proof / model |
| Low (conditional) | [Old mailbox results can file after connected account changes or disconnect](findings/application-api/low/app-01-find-036-old-mailbox-results-can-file-after-connected-account-changes-or-disconnect.md) | Original test excerpts |
| Low (conditional) | [A removed digest recipient receives a new digest after correction](findings/application-api/low/app-01-find-040-a-removed-digest-recipient-receives-a-new-digest-after-correction.md) | Original test excerpts |
| Low (conditional) | [Security invite alert is missed after the one-hour scan window](findings/application-api/low/app-01-find-041-security-invite-alert-is-missed-after-the-one-hour-scan-window.md) | Original test excerpts |
| Low (conditional) | [Mail nudge creates an open old-message task after staff completes, snoozes or replies](findings/application-api/low/app-01-find-042-mail-nudge-creates-an-open-old-message-task-after-staff-completes-snoozes-or-replies.md) | Original test excerpts |
| Low (conditional) | [Renewed company COI gains a new obsolete expiry task and alert after cleanup](findings/application-api/low/app-01-find-043-renewed-company-coi-gains-a-new-obsolete-expiry-task-and-alert-after-cleanup.md) | Original test excerpts |
| Low (conditional; provisional) | [Untouched-lead worker creates a false task/alert after staff already touched lead](findings/application-api/low/app-01-find-044-untouched-lead-worker-creates-a-false-task-alert-after-staff-already-touched-lead.md) | Original test excerpts |
| Low (conditional integrity) | [Expense decision can finalize a revised amount/description without current-version review](findings/application-api/low/app-01-find-045-expense-decision-can-finalize-a-revised-amount-description-without-current-version-review.md) | Original test excerpts |
| Low (conditional workflow integrity) | [Queued Intake push creates estimating work after lead becomes Lost](findings/application-api/low/app-01-find-048-queued-intake-push-creates-estimating-work-after-lead-becomes-lost.md) | Original test excerpts |
| Low (conditional financial-offer integrity) | [Queued older quote email carries the revised quote PDF](findings/application-api/low/app-01-find-049-queued-older-quote-email-carries-the-revised-quote-pdf.md) | Original test excerpts |
| Low (conditional communication integrity) | [Scheduled-date SMS can leave after the job is unassigned](findings/application-api/low/app-01-find-050-scheduled-date-sms-can-leave-after-the-job-is-unassigned.md) | Original test excerpts |
| Low (planning integrity; downstream harm unproved) | [Out-of-service unit remains on a future equipment deployment](findings/application-api/low/app-01-find-052-out-of-service-unit-remains-on-a-future-equipment-deployment.md) | Original test excerpts |
| Low (same-tenant financial-workflow integrity) | [Bill-blind sub-job manager can unlink an existing bill](findings/application-api/low/app-01-find-054-bill-blind-sub-job-manager-can-unlink-an-existing-bill.md) | Original test excerpts |
| Low (same-tenant confidentiality) | [Foreman reads another crew's customer and site through Plants and Loads](findings/application-api/low/app-01-find-058-foreman-reads-another-crew-s-customer-and-site-through-plants-and-loads.md) | Original test excerpts |
| Low (internal workflow integrity) | [End-of-day job creates a stale email task after final inbox triage](findings/application-api/low/app-01-find-062-end-of-day-job-creates-a-stale-email-task-after-final-inbox-triage.md) | Original test excerpts |
| Low (financial-record integrity) | [Cost-free repair correction leaves incident and maintenance costs inconsistent](findings/application-api/low/app-01-find-063-cost-free-repair-correction-leaves-incident-and-maintenance-costs-inconsistent.md) | Original test excerpts |
| Low (customer communication integrity) | [Invoice reminder quotes stale balance after partial payment](findings/application-api/low/app-01-find-064-invoice-reminder-quotes-stale-balance-after-partial-payment.md) | Original test excerpts |
| Low (conditional subscription integrity/alert availability) | [Cap-refused push claim deletes another tenant's subscription](findings/application-api/low/app-01-find-066-cap-refused-push-claim-deletes-another-tenant-s-subscription.md) | Original test excerpts |
| Low | [Customer-link redaction removes unrelated reference codes](findings/application-api/low/app-01-n-01-customer-link-redaction-removes-unrelated-reference-codes.md) | Regression tests |
| Low | [Task-owner visibility guard conflicts with routed recipients](findings/application-api/low/app-01-n-02-task-owner-visibility-guard-conflicts-with-routed-recipients.md) | Recorded observation |
| Low | [Logical restore suppresses history-based security alerts](findings/application-api/low/app-01-n-03-logical-restore-suppresses-history-based-security-alerts.md) | Regression tests |
| Low | [Database disconnect during enrolment SMTP crashes the worker](findings/application-api/low/app-01-n-04-database-disconnect-during-enrolment-smtp-crashes-the-worker.md) | Regression tests |
| Low | [Cost-restricted sub-job manager loses the Notes editor](findings/application-api/low/app-01-n-05-cost-restricted-sub-job-manager-loses-the-notes-editor.md) | Regression tests |
| Low | [Explicit payment date shifts to the previous company day](findings/application-api/low/app-01-n-06-explicit-payment-date-shifts-to-the-previous-company-day.md) | Regression tests |
| Low | [Customer timeline reads spend seconds compiling query plans](findings/application-api/low/app-01-n-08-customer-timeline-reads-spend-seconds-compiling-query-plans.md) | Regression tests |

### Asset Management

| Severity | Case study | Evidence |
| --- | --- | --- |
| High — report proposal; proxy scope, fee-as-yield classification and novelty require program ruling | [Delisted tracked assets make ordinary redemptions skip accrued fund fees](findings/asset-management/high/am-02-fee-hook.md) | Original test excerpts |
| High (future severity) | [Prototype AMM adapters violate withdrawal and balance interfaces](findings/asset-management/high-future/am-01-f-01-amm-adapters-violate-project-strategy-deposit-and-withdrawal-semantics.md) | Partial proof / model |
| High (future severity) | [Spot-reserve valuation permits modeled extraction of idle liquidity](findings/asset-management/high-future/am-01-f-02-manipulable-amm-reserves-can-inflate-xycloans-nav-and-drain-idle-liquidity.md) | Partial proof / model |
| Medium | [Stale strategy NAV shifts an upstream loss to remaining vault holders](findings/asset-management/medium/am-01-m-01-stale-strategy-nav-shifts-an-upstream-loss-to-remaining-vault-holders.md) | Original test excerpts |
| Medium | [Public invest-on-deposit deletes negative strategy loss carry](findings/asset-management/medium/am-01-m-02-public-invest-on-deposit-deletes-negative-strategy-loss-carry.md) | Original test excerpts |
| Medium | [Incompatible fee receiver blocks management and incident response](findings/asset-management/medium/am-01-m-03-incompatible-fee-receiver-blocks-management-and-incident-response.md) | Recorded observation |
| Medium | [Refresh tokens remain reusable after refresh and concurrent replay](findings/asset-management/medium/am-01-m-04-refresh-tokens-remain-reusable-after-refresh-and-concurrent-replay.md) | Recorded observation |
| Low | [Unauthenticated quote/checkpoint mutates report and fee state](findings/asset-management/low/am-01-l-01-unauthenticated-quote-checkpoint-mutates-report-and-fee-state.md) | Original test excerpts |
| Low | [Persistent API-origin override forwards bearer credentials](findings/asset-management/low/am-01-l-02-frameable-console-persists-an-arbitrary-api-origin-across-sessions.md) | Source / configuration |

### Blockchain Infrastructure

| Severity | Case study | Evidence |
| --- | --- | --- |
| Medium — conservative source-assigned upgrade-safety rating; default-off scope boundary unresolved | [Legacy zero commit timestamps bypass the configured restart halt deadline](findings/blockchain-infrastructure/medium/chain-01-halt-time.md) | Original test excerpts |

### Collateralized Lending

| Severity | Case study | Evidence |
| --- | --- | --- |
| Medium — report-assigned, unadjudicated | [A third-party opener installs withdrawal authority on a victim-owned position](findings/collateralized-lending/medium/lend-02-af-02.md) | Original test excerpts |

### Cross Chain Bridges

| Severity | Case study | Evidence |
| --- | --- | --- |
| Unrated — no assigned severity established in the reviewed report | [Failed-message refunds bypass ordinary execution limits and accounting](findings/cross-chain-bridges/unrated/bridge-01-failed-message-refund.md) | Original test excerpts |

### Epoch Lending

| Severity | Case study | Evidence |
| --- | --- | --- |
| Critical author recommendation; conflicting High/potential Critical source descriptions | [The allowKick state suppresses public epoch closure after debt repayment](findings/epoch-lending/critical-conflicted/lend-01-p1.md) | Original test excerpts |
| Critical author recommendation; conflicting High/potential Critical source descriptions | [Stale repayment credit permits an underfunded later epoch to settle](findings/epoch-lending/critical-conflicted/lend-01-p2.md) | Original test excerpts |

### Insurance Vaults

| Severity | Case study | Evidence |
| --- | --- | --- |
| Medium — submitted rating; later High/Critical interpretations remain unadjudicated | [Period-start checkpoint replacement suppresses a valid coverage payout](findings/insurance-vaults/medium/ins-01-rl-001.md) | Original test excerpts |

### Nft Rewards

These are research candidates and hardening observations; severity values are ceilings, not confirmed exploit ratings.

| Severity | Case study | Evidence |
| --- | --- | --- |
| Medium ceiling; audit candidate, privileged/admin and external-NFT prerequisites | [Failed `ownerOf` is accepted as proof that a live token burned](findings/nft-rewards/medium/nft-01-lhf-01-failed-ownerof-is-accepted-as-proof-that-a-live-token-burned.md) | Original test excerpts |
| Medium to Low ceiling; accounting/liveness candidate with privileged recovery | [Positive compound proceeds can be impossible to book honestly with zero principal](findings/nft-rewards/medium-low/nft-01-lhf-03-positive-compound-proceeds-can-be-impossible-to-book-honestly-with-zero-principal.md) | Original test excerpts |
| Low to Medium operational ceiling; wrong-deployment configuration risk | [The indexer does not bind its configured Lens to its configured Core](findings/nft-rewards/medium-low/nft-01-lhf-09-the-indexer-does-not-bind-its-configured-lens-to-its-configured-core.md) | Source / configuration |
| Low ceiling; confirmed can-check/write inconsistency | [`canReportCompound()` approves a write that later reverts](findings/nft-rewards/low/nft-01-lhf-02-canreportcompound-approves-a-write-that-later-reverts.md) | Original test excerpts |
| Low ceiling; accounting-reporting drift candidate | [Corrections after deactivation understate dead liability](findings/nft-rewards/low/nft-01-lhf-04-corrections-after-deactivation-understate-dead-liability.md) | Source / configuration |
| Low ceiling; privileged configuration denial of service | [Activation price accepts a wider type than NFT payment storage](findings/nft-rewards/low/nft-01-lhf-05-activation-price-accepts-a-wider-type-than-nft-payment-storage.md) | Source / configuration |
| Low ceiling; privileged lifecycle denial at an extreme integer range | [Valid principal can exceed the signed lifecycle-event domain](findings/nft-rewards/low/nft-01-lhf-06-valid-principal-can-exceed-the-signed-lifecycle-event-domain.md) | Partial proof / model |
| Low ceiling; lens/operator guidance inconsistency | [`pendingWork()` can recommend an impossible batch report](findings/nft-rewards/low/nft-01-lhf-07-pendingwork-can-recommend-an-impossible-batch-report.md) | Source / configuration |
| Low ceiling; operational false-green health signal | [Reconciliation failure does not make the indexer unhealthy](findings/nft-rewards/low/nft-01-lhf-08-reconciliation-failure-does-not-make-the-indexer-unhealthy.md) | Source / configuration |
| Low ceiling; wrong-backend detection gap | [The app ignores the indexer’s direct Core identity](findings/nft-rewards/low/nft-01-lhf-10-the-app-ignores-the-indexer-s-direct-core-identity.md) | Source / configuration |
| Low ceiling; conditional can-check/write parity gap | [`canRetireCollection()` omits the membership-list check](findings/nft-rewards/low/nft-01-p2-01-canretirecollection-omits-the-membership-list-check.md) | Source / configuration |
| Low ceiling; trusted-operator boundary hardening | [`reportBatch()` forgives activation-wallet shortfall silently](findings/nft-rewards/low/nft-01-p2-02-reportbatch-forgives-activation-wallet-shortfall-silently.md) | Source / configuration |
| Low ceiling; conditional accounting-drift hardening, related to LHF-04 | [Saturating dead counters allow cross-level drift](findings/nft-rewards/low/nft-01-p2-03-saturating-dead-counters-allow-cross-level-drift.md) | Source / configuration |
| Informational to Low ceiling; upgrade/hotfix configuration footgun | [Configured selector routes can be silently shadowed by Core dispatch](findings/nft-rewards/informational-low/nft-01-lhf-11-configured-selector-routes-can-be-silently-shadowed-by-core-dispatch.md) | Source / configuration |
| Informational to Low ceiling; pause-matrix design decision | [`deactivateBurned` and `sweepExcess` are outside every pause domain](findings/nft-rewards/informational-low/nft-01-p2-04-deactivateburned-and-sweepexcess-are-outside-every-pause-domain.md) | Source / configuration |
| Informational ceiling; privileged lifecycle sequencing note | [Move or revive inside a correction window forfeits the correction](findings/nft-rewards/informational/nft-01-p2-05-move-or-revive-inside-a-correction-window-forfeits-the-correction.md) | Source / configuration |
| Trust-dependent High ceiling only with untrusted collection upgrade authority; no High vulnerability established | [Collection code-change detection remains advisory](findings/nft-rewards/trust-dependent-high/nft-01-lhf-12-collection-code-change-detection-remains-advisory.md) | Source / configuration |

### Perpetuals

| Severity | Case study | Evidence |
| --- | --- | --- |
| Critical — final client-report assignment; historical finding | [Missing Timelock on Admin Operations with single step transfer](findings/perpetuals/critical/perp-01-c1-missing-timelock-on-admin-operations-with-single-step-transfer.md) | Original test excerpts |
| High — final client-report assignment; historical finding | [VFR Funding Obligations Invisible to AUM](findings/perpetuals/high/perp-01-h1-vfr-funding-obligations-invisible-to-aum.md) | Partial proof / model |
| High — final client-report assignment; historical finding | [Oracle guarantees degrade under staleness](findings/perpetuals/high/perp-01-h2-oracle-guarantees-degrade-under-staleness.md) | Partial proof / model |
| High — final client-report assignment; historical finding | [Switchboard Oracle Staleness Laundering](findings/perpetuals/high/perp-01-h3-switchboard-oracle-staleness-laundering.md) | Partial proof / model |
| Medium — final client-report assignment; historical finding | [Governance Inflation: Resolved Locked Stakes Recount on Sync](findings/perpetuals/medium/perp-01-m1-governance-inflation-resolved-locked-stakes-recount-on-sync.md) | Partial proof / model |
| Medium — final client-report assignment; historical finding | [Pixel-Perfect Liquidation via Stale Oracle Price](findings/perpetuals/medium/perp-01-m2-pixel-perfect-liquidation-via-stale-oracle-price.md) | Original test excerpts |
| Medium — final client-report assignment; historical finding | [PnL Uses Midpoint Instead of Protective Confidence Bands](findings/perpetuals/medium/perp-01-m3-pnl-uses-midpoint-instead-of-protective-confidence-bands.md) | Original test excerpts |
| Medium — final client-report assignment; historical finding | [Account revival: closed position PDAs can be zombified](findings/perpetuals/medium/perp-01-m4-account-revival-closed-position-pdas-can-be-zombified.md) | Original test excerpts |
| Medium — final client-report assignment; historical finding | [Autonom Market-Opening Signatures Use Ambiguous Encoding](findings/perpetuals/medium/perp-01-m5-autonom-market-opening-signatures-use-ambiguous-encoding.md) | Partial proof / model |
| Medium — final client-report assignment; historical finding | [Underweight Oracle Consensus](findings/perpetuals/medium/perp-01-m6-underweight-oracle-consensus.md) | Partial proof / model |
| Medium — final client-report assignment; historical finding | [Incorrect exponent handling](findings/perpetuals/medium/perp-01-m7-incorrect-exponent-handling.md) | Partial proof / model |
| Medium — final client-report assignment; historical finding | [Missing Slippage Enforcement on Open Position](findings/perpetuals/medium/perp-01-m8-missing-slippage-enforcement-on-open-position.md) | Original test excerpts |
| Low — final client-report assignment; historical finding | [Schedule-Dependent Interest and Funding Inflation via Ceiling Division Rounding](findings/perpetuals/low/perp-01-l1-schedule-dependent-interest-and-funding-inflation-via-ceiling-division-rounding.md) | Partial proof / model |
| Low — final client-report assignment; historical finding | [Griefing: Deprecated Permissionless IDL](findings/perpetuals/low/perp-01-l10-griefing-deprecated-permissionless-idl.md) | Partial proof / model |
| Low — final client-report assignment; historical finding | [LP Sandwich: No Structural Hold Period](findings/perpetuals/low/perp-01-l11-lp-sandwich-no-structural-hold-period.md) | Partial proof / model |
| Low — final client-report assignment; historical finding | [Wrapping Overflow in Financial Calculation](findings/perpetuals/low/perp-01-l12-wrapping-overflow-in-financial-calculation.md) | Partial proof / model |
| Low — final client-report assignment; historical finding | [Hardcoded RPC Authentication Tokens in Source](findings/perpetuals/low/perp-01-l2-hardcoded-rpc-authentication-tokens-in-source.md) | Source / configuration |
| Low — final client-report assignment; historical finding | [DOS: Residual Unchecked Oracle Math](findings/perpetuals/low/perp-01-l3-dos-residual-unchecked-oracle-math.md) | Partial proof / model |
| Low — final client-report assignment; historical finding | [Griefing: Pool-Wide Write-Lock](findings/perpetuals/low/perp-01-l4-griefing-pool-wide-write-lock.md) | Partial proof / model |
| Low — final client-report assignment; historical finding | [Griefing: Liquidation Capture](findings/perpetuals/low/perp-01-l5-griefing-liquidation-capture.md) | Partial proof / model |
| Low — final client-report assignment; historical finding | [Update-Time Lock Traps User After add_collateral](findings/perpetuals/low/perp-01-l6-update-time-lock-traps-user-after-add-collateral.md) | Original test excerpts |
| Low — final client-report assignment; historical finding | [Governance Lacks Participation Quorum and Execution Timelock](findings/perpetuals/low/perp-01-l7-governance-lacks-participation-quorum-and-execution-timelock.md) | Source / configuration |
| Low — final client-report assignment; historical finding | [Fee Distribution: Off-Peg Stablecoin Creates Phantom Pool Value](findings/perpetuals/low/perp-01-l8-fee-distribution-off-peg-stablecoin-creates-phantom-pool-value.md) | Original test excerpts |
| Low — final client-report assignment; historical finding | [Feeless Swap Skips Borrow Rate Update](findings/perpetuals/low/perp-01-l9-feeless-swap-skips-borrow-rate-update.md) | Original test excerpts |

### Prediction Market Lending

| Severity | Case study | Evidence |
| --- | --- | --- |
| High — final source assessment; capital and modeled-settlement assumptions apply | [Pre-loss deposits capture recovery allocations and recycle capital](findings/prediction-market-lending/high/pm-01-n4-pre-loss-deposit-recovery-capture.md) | Original test excerpts |
| High — mainnet-posture / privileged-control risk; testnet-fork demonstration | [Single-controller custody and self-reported accounting](findings/prediction-market-lending/high/pm-01-v1-single-controller-custody-and-accounting.md) | Original test excerpts |
| High — confirmed mechanism with signed-input and market prerequisites | [Superseded signed prices remain actionable inside the freshness window](findings/prediction-market-lending/high/pm-01-v3-superseded-signed-marks.md) | Original test excerpts |
| Medium–High when chained with the expiry clock — qualified source severity | [Public quote consumption can block renewal until collateral forfeiture](findings/prediction-market-lending/medium-high/pm-01-v4-public-quote-consumption-forced-expiry.md) | Original test excerpts |
| Medium — liveness and privileged operational dependency | [Executor outage blocks leverage-position exits](findings/prediction-market-lending/medium/pm-01-l1-executor-outage-blocks-exits.md) | Original test excerpts |
| Medium — confirmed with capital, timing, recovery and rescue bounds | [Large redemption queues block renewal and participate in expiry recovery](findings/prediction-market-lending/medium/pm-01-n2-queue-blocks-renewal-and-captures-recovery.md) | Original test excerpts |
| Medium — governance/mainnet posture; authorized configuration risk | [Five-minute governance delay for critical policy changes](findings/prediction-market-lending/medium/pm-01-v2-short-governance-timelock.md) | Source / configuration |
| Low — conditional production-ramp hardening; current fixed-rate slippage claim downgraded | [Zap conversion lacks a caller minimum-output bound](findings/prediction-market-lending/low/pm-01-v5-l4-zap-minimum-output-gap.md) | Source / configuration |
| Low — voluntary liquidity/UX constraint; operator-lock hypothesis refuted | [Queued redemption cannot be canceled while awaiting liquidity](findings/prediction-market-lending/low/pm-01-v6-noncancelable-redemption-queue.md) | Partial proof / model |
| Informational — bounded residual; zero-supply scenario structurally refuted | [Pro-rata recovery rounding leaves protected reserved dust](findings/prediction-market-lending/informational/pm-01-l3-recovery-rounding-dust.md) | Source / configuration |
| Unrated — no standalone severity assigned in the source assessment; signer-dependent mechanism | [Future-dated signed marks bypass the intended freshness horizon](findings/prediction-market-lending/unrated/pm-01-n1-future-dated-marks.md) | Original test excerpts |

### Rewards Governance

| Severity | Case study | Evidence |
| --- | --- | --- |
| Critical (final report assignment) | [earnedId replay drains the shared reward bucket](findings/rewards-governance/critical/rg-01-c1-earnedid-replay-drains-the-shared-reward-bucket.md) | Original test excerpts |
| High (final report assignment) | [Trustline gate ships fail-open / never enabled](findings/rewards-governance/high/rg-01-h1-trustline-gate-ships-fail-open-never-enabled.md) | Source / configuration |
| High (final report assignment) | [Captured pool can mint full poolCap without entitlement checks](findings/rewards-governance/high/rg-01-h2-captured-pool-can-mint-full-poolcap-without-entitlement-checks.md) | Original test excerpts |
| High (final report assignment) | [Council vote burning can permanently disable governance](findings/rewards-governance/high/rg-01-h3-council-vote-burning-can-permanently-disable-governance.md) | Source / configuration |
| High (final report assignment) | [Council initialization can fail open below governance quorum](findings/rewards-governance/high/rg-01-h4-council-initialization-can-fail-open-below-governance-quorum.md) | Source / configuration |
| Medium (final report assignment) | [poolCap unenforced at allocation strands rewards](findings/rewards-governance/medium/rg-01-m1-poolcap-unenforced-at-allocation-strands-rewards.md) | Original test excerpts |
| Medium (final report assignment) | [Governance capture is cheap](findings/rewards-governance/medium/rg-01-m2-governance-capture-is-cheap.md) | Partial proof / model |
| Medium (final report assignment) | [Deployer retains [project]CouncilToken.DEFAULT_ADMIN_ROLE](findings/rewards-governance/medium/rg-01-m3-deployer-retains-project-counciltoken-default-admin-role.md) | Partial proof / model |
| Medium (final report assignment) | [TIMELOCK_DELAY_SECONDS=0 is accepted](findings/rewards-governance/medium/rg-01-m4-timelock-delay-seconds-0-is-accepted.md) | Partial proof / model |
| Medium (final report assignment) | [Deployer EOA owns the Trustline engines](findings/rewards-governance/medium/rg-01-m5-deployer-eoa-owns-the-trustline-engines.md) | Source / configuration |
| Medium (final report assignment) | [updateAndAdjust missing self-scope](findings/rewards-governance/medium/rg-01-m6-updateandadjust-missing-self-scope.md) | Original test excerpts |
| Medium (final report assignment) | [One pool reserves global headroom](findings/rewards-governance/medium/rg-01-m7-one-pool-reserves-global-headroom.md) | Original test excerpts |
| Medium (final report assignment) | [Lifetime poolCap permanently bricks pools](findings/rewards-governance/medium/rg-01-m8-lifetime-poolcap-permanently-bricks-pools.md) | Original test excerpts |
| Medium (final report assignment) | [Reverting Trustline engine bricks claims](findings/rewards-governance/medium/rg-01-m9-reverting-trustline-engine-bricks-claims.md) | Original test excerpts |
| Low (final report assignment) | [Timelock lacks AUTHORIZED_PARTY_ROLE on minter](findings/rewards-governance/low/rg-01-l1-timelock-lacks-authorized-party-role-on-minter.md) | Partial proof / model |
| Low (final report assignment) | [Unbounded reward arrays](findings/rewards-governance/low/rg-01-l2-unbounded-reward-arrays.md) | Source / configuration |
| Low (final report assignment) | [Pools have no independent pause](findings/rewards-governance/low/rg-01-l3-pools-have-no-independent-pause.md) | Source / configuration |
| Low (final report assignment) | [Delegated backend is screened](findings/rewards-governance/low/rg-01-m10-delegated-backend-is-screened.md) | Original test excerpts |
| Low (final report assignment) | [Merkle root reusable across uuidHash](findings/rewards-governance/low/rg-01-m11-merkle-root-reusable-across-uuidhash.md) | Original test excerpts |

### Tokenized Assets

| Severity | Case study | Evidence |
| --- | --- | --- |
| Medium — report-assigned; admin-recoverable availability loss | [Swap redemptions exhaust an outer-vault quota without consuming its liquidity](findings/tokenized-assets/medium/rwa-01-m02.md) | Original test excerpts |

