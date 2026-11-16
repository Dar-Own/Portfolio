# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Medium (conditional)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Customer quote can sign after staff cancellation commits and revive canceled lead

## Description

A customer quote can be signed after staff cancellation commits, and its event can revive the canceled lead as won.

Prerequisite and affected workflow: A valid quote holder starts signing before cancellation while the final signature write waits until after cancellation commits.

Established observation: Staff cancellation committed while a valid customer quote-signing request waited at the final write. The signing request then succeeded, recorded a signature, and its event changed the canceled lead to won.

Limits: Natural overlap frequency, deployed behavior, and the legal effect of a real signature remain untested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A valid quote holder starts signing before cancellation while the final signature write waits until after cancellation commits.

Source: `apps/api/src/features/estimates/routes.ts:373-394` (1d5d98db9acf).

```typescript
  app.post('/:token/sign', { schema: { params: tokenParams, body: signBody } }, async (request): Promise<QuoteAnswer> => {
    const { token } = request.params;
    const body = request.body;
    const problem = signatureProblem({ fullName: body.fullName, agreed: body.agreed });
    if (problem) throw invalid(problem);
    const view = await live(token);
    // Accepted exactly as when the office drags it there: the worker's handler of quote.signed moves the lead.
    const signed = await signQuote(app.db, {
      token,
      versionHash: body.versionHash,
      fullName: body.fullName,
      ip: request.ip || null,
      userAgent: userAgentOf(request.headers['user-agent']),
    });
    return {
      status: 'signed',
      name: signed.signedName,
      at: signed.signedAt.toISOString(),
      message: `Thank you, ${signed.signedName}. Estimate ${view.document.number} is signed, and ${view.document.brandName} will be in touch about scheduling.`,
    };
  });

```

#### Step 2 — Cross the missing state or authority check

Quote signing checks the lead stage without coordinating with cancellation. Later signature processing can promote an already canceled lead.

Source: `packages/db/migrations/0106_automation_leaves_the_api.sql:244-256` (1d5d98db9acf).

```sql
  select x.tenant_id, x.id, x.estimate_id, x.version_hash, x.revoked_at, x.used_at, x.expires_at into k
    from app.estimate_links x
    join app.tenants t on t.id = x.tenant_id and t.suspended_at is null
   where x.token_hash = p_token_hash
   for update of x;
  if not found or k.revoked_at is not null or k.used_at is not null or k.expires_at <= now() then
    raise exception 'This link isn''t valid anymore.';
  end if;
  select e.status::text, l.stage::text, e.lead_id into v_status, v_stage, lead_id
    from app.estimates e
    join app.leads l on l.tenant_id = e.tenant_id and l.id = e.lead_id
   where e.tenant_id = k.tenant_id and e.id = k.estimate_id
   for update of e;
```

Source: `apps/api/src/features/estimates/quote-links.ts:264-284` (1d5d98db9acf).

```typescript
export async function acceptSignedQuote(ctx: DbContext, estimateId: string): Promise<void> {
  const estimate = await answeredEstimate(ctx, estimateId);
  if (!estimate || estimate.status !== 'approved' || estimate.decision !== 'signed') return;
  await ctx.client.query('savepoint accept_signed_lead');
  try {
    await setStage(ctx, estimate.lead_id, 'won');
    await ctx.client.query('release savepoint accept_signed_lead');
  } catch (error) {
    await ctx.client.query('rollback to savepoint accept_signed_lead');
    const reason = error instanceof HttpError ? error.message : 'the lead could not move to Accepted';
    await answerTask(
      ctx,
      estimate,
      `${estimate.signed_name ?? 'The customer'} signed ${estimate.number} (${formatCents(estimate.total_cents, { decimals: 0 })}). Move ${estimate.customer_name} to Accepted: ${reason}`,
      `quote-signed:${estimateId}`,
    );
  }
}

/** The customer said no (quote.declined): the lead stays where it is, and its owner is asked to follow up. */
export async function followUpDeclinedQuote(ctx: DbContext, estimateId: string): Promise<void> {
```


#### Step 3 — Observe the durable effect

Staff cancellation committed while a valid customer quote-signing request waited at the final write. The signing request then succeeded, recorded a signature, and its event changed the canceled lead to won.

Natural overlap frequency, deployed behavior, and the legal effect of a real signature remain untested.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/http-poc/run-quote-cancel-race.mjs:310-337` (synthetic test).

```javascript
    advisoryGateHeld = false;
    mark('public-page-gate-released-after-cancel-commit');
    const heldPage = await pagePromise;
    mark('public-page-http-response', { status: heldPage.statusCode });
    expectStatus(heldPage, 200, 'in-flight public page');
  } else {
    await blocker.query('commit');
    blockerTransaction = false;
    mark('estimate-row-lock-released-after-cancel-commit');
  }
  const sign = await signPromise;
  mark('public-sign-http-response', { status: sign.statusCode });
  result.snapshots.afterSignBeforeWorker = snapshot(estimateId, leadId, tokenHash);
  result.orderedObservation = {
    cancelCommittedBeforeBarrierRelease: true,
    signDoorBlockedAcrossCancelCommit: true,
    signResponseStatus: sign.statusCode,
    signedAfterCancel: result.snapshots.afterSignBeforeWorker.signature_count === 1 && result.snapshots.afterSignBeforeWorker.signed_events === 1,
  };
  if (sign.statusCode === 200 && result.orderedObservation.signedAfterCancel) {
    result.verdict = 'POST_CANCEL_SIGNATURE_COMMITTED_IN_LOCAL_RACE';
  } else if (sign.statusCode === 404 && result.snapshots.afterSignBeforeWorker.signature_count === 0) {
    result.verdict = 'CANCEL_BLOCKED_SIGN_IN_THIS_ORDER';
  } else {
    result.verdict = 'UNRESOLVED_ORDERED_RESULT';
  }
  // The real worker handles committed quote.signed and lead.stage_changed events in the synthetic cluster.
  await waitForEvents(app);
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A valid quote holder starts signing before cancellation while the final signature write waits until after cancellation commits.

Recorded result: Staff cancellation committed while a valid customer quote-signing request waited at the final write. The signing request then succeeded, recorded a signature, and its event changed the canceled lead to won.

Scope: Natural overlap frequency, deployed behavior, and the legal effect of a real signature remain untested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/http-poc/run-quote-cancel-race.mjs:310-337` (synthetic test).

```javascript
    advisoryGateHeld = false;
    mark('public-page-gate-released-after-cancel-commit');
    const heldPage = await pagePromise;
    mark('public-page-http-response', { status: heldPage.statusCode });
    expectStatus(heldPage, 200, 'in-flight public page');
  } else {
    await blocker.query('commit');
    blockerTransaction = false;
    mark('estimate-row-lock-released-after-cancel-commit');
  }
  const sign = await signPromise;
  mark('public-sign-http-response', { status: sign.statusCode });
  result.snapshots.afterSignBeforeWorker = snapshot(estimateId, leadId, tokenHash);
  result.orderedObservation = {
    cancelCommittedBeforeBarrierRelease: true,
    signDoorBlockedAcrossCancelCommit: true,
    signResponseStatus: sign.statusCode,
    signedAfterCancel: result.snapshots.afterSignBeforeWorker.signature_count === 1 && result.snapshots.afterSignBeforeWorker.signed_events === 1,
  };
  if (sign.statusCode === 200 && result.orderedObservation.signedAfterCancel) {
    result.verdict = 'POST_CANCEL_SIGNATURE_COMMITTED_IN_LOCAL_RACE';
  } else if (sign.statusCode === 404 && result.snapshots.afterSignBeforeWorker.signature_count === 0) {
    result.verdict = 'CANCEL_BLOCKED_SIGN_IN_THIS_ORDER';
  } else {
    result.verdict = 'UNRESOLVED_ORDERED_RESULT';
  }
  // The real worker handles committed quote.signed and lead.stage_changed events in the synthetic cluster.
  await waitForEvents(app);
```

## Remediation

Serialize final quote signing with lead cancellation and prevent the signature event handler from reversing a committed cancellation.
