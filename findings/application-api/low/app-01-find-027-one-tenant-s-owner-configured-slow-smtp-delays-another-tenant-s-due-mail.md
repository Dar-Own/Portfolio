# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low (conditional)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# One tenant's owner-configured slow SMTP delays another tenant's due mail

## Description

A slow SMTP endpoint configured by tenant A delays tenant B's due mail in the sequential outbox worker.

Prerequisite and affected workflow: A is visited before B, both have due mail, and A's reachable SMTP server stalls during handoff.

Established observation: A tenant-controlled slow SMTP endpoint held the sequential outbox worker while another tenant's due mail waited. Fast-provider and single-tenant controls showed that the cross-tenant delay came from the shared dispatch order.

Limits: Production egress to owner-configured SMTP was not verified. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A is visited before B, both have due mail, and A's reachable SMTP server stalls during handoff.

Source: `apps/api/src/features/connections/routes.ts:89-120` (1d5d98db9acf).

```typescript
    app.put('/:key', { schema: { params: keyParams, body: saveBody } }, async (request) => {
      requirePermission(request.session, 'connections.manage');
      const session = request.session;
      const key = request.params.key;
      if (!isConnectionKey(key) || !(CONNECTION_KEYS as readonly string[]).includes(key)) throw notFound("That setting doesn't exist.");
      const value = request.body.value === null ? null : request.body.value.trim();
      if (value !== null) {
        const problem = connectionValueProblem(key, value);
        if (problem) throw invalid(problem);
        if (isSecretConnectionKey(key) && !app.config.secretsKey) throw new HttpError(409, 'no_master_key', MASTER_KEY_MISSING_MESSAGE);
      }

      // A fresh password check, the same as signing in (guesses count toward the lockout).
      const check = await confirmPassword(app.db, session, request.body.password, { ip: request.ip, userAgent: request.headers['user-agent'] });
      if (check === 'locked') throw locked();
      if (check !== 'ok') throw new HttpError(422, 'wrong_password', WRONG_PASSWORD_MESSAGE);

      // The owner's settings (0080, 0089): anyone else asks the owner, who gets a task with a one-click
      // approve (one request waits per setting: asking again replaces it, so nothing piles up). Which
      // settings those are is the database's answer, asked here, never a list kept in the API.
      const ownerOnly = await app.db.withUser(session, (ctx) => isOwnerOnlyChange(ctx, key, value));
      if (ownerOnly && !session.role.isOwner) {
        const id = await app.db.withUser(session, (ctx) => requestConnectionChange(ctx, key as ConnectionKey, value, app.config.secretsKey));
        request.log.info({ setting: key, removed: value === null, requestId: id }, "A company connection change was sent for the owner's approval");
        // waitingForOwner: the answer says what happened, so the page doesn't have to guess from the list of
        // waiting changes — which told the owner his own save was "sent to the owner" whenever somebody
        // else had already asked for that same setting (the review of this fix, finding 8).
        return { ...(await view(app, session)), waitingForOwner: true };
      }
      await app.db.withUser(session, (ctx) => saveConnection(ctx, key as ConnectionKey, value, app.config.secretsKey));
      request.log.info({ setting: key, removed: value === null }, 'A company connection setting was changed');
      return { ...(await view(app, session)), waitingForOwner: false };
```

#### Step 2 — Cross the missing state or authority check

The outbox processes tenants sequentially and awaits each tenant's SMTP attempt without a per-tenant budget.

Source: `apps/worker/src/db.ts:80-95` (1d5d98db9acf).

```typescript
/**
 * Calls `fn` once for every company that isn't suspended (app.active_tenant_ids(), worker only), one
 * company at a time. A company whose `fn` throws is reported in `failures`; the others still run.
 */
export async function forEachActiveTenant(pool: Queryable, fn: (tenantId: string) => Promise<void>): Promise<CompanyScan> {
  const { rows } = await pool.query<{ id: string }>('select t.id::text as id from app.active_tenant_ids() as t(id)');
  const scan: CompanyScan = { companies: rows.length, failures: [] };
  for (const { id } of rows) {
    try {
      await fn(id);
    } catch (error) {
      scan.failures.push({ tenantId: id, error });
    }
  }
  return scan;
}
```

Source: `apps/worker/src/jobs/outbox-delivery.ts:279-292` (1d5d98db9acf).

```typescript
export async function deliverOutbox(ctx: JobContext, options: OutboxDeliveryOptions = {}): Promise<JobResult> {
  const tally = emptyTally();
  const run: RunState = { serversDown: new Set(), install: installMail(ctx) };
  const mailbox = options.mailbox ?? companyMailbox(ctx.config.production);

  const scan = await forEachActiveTenant(ctx.pool, async (tenantId) => {
    const now = await ctx.now();
    // This company's own way out (its Connections page), read once per run and used for its messages only.
    const outbound = await companyOutbound(ctx, tenantId);
    await retryFailedMessages(ctx, tenantId, tally);
    await sendQueuedEmails(ctx, tenantId, now, tally, run, outbound, await sentCopies(ctx, tenantId, outbound, mailbox, now));
    await sendQueuedTexts(ctx, tenantId, now, tally, outbound);
    await markTextsDelivered(ctx, tenantId, tally);
  });
```

Source: `apps/worker/src/jobs/outbox-delivery.ts:665-681` (1d5d98db9acf).

```typescript
        const result = await sender.send({
          from: { name: invitation ? brand.company_name : brand.name, address: fromAddress },
          to: message.to_address,
          cc,
          subject,
          text: letter.text,
          attachments,
          // The threading headers of a reply, and on every letter the app's own Message-ID and its mark: the
          // mailbox sync files a customer's answer into the conversation by the one, and knows the Sent
          // folder's copy is the app's own by the other (docs/email-manager.md section 3.3).
          inReplyTo: message.in_reply_to,
          references,
          messageId,
          headers: { [APP_MESSAGE_HEADER]: message.id },
          keepRaw: copied && copies.mailbox !== null,
        });
        if (result.accepted.length === 0) throw new Error('The mail server refused the recipient.');
```


#### Step 3 — Observe the durable effect

A tenant-controlled slow SMTP endpoint held the sequential outbox worker while another tenant's due mail waited. Fast-provider and single-tenant controls showed that the cross-tenant delay came from the shared dispatch order.

Production egress to owner-configured SMTP was not verified.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/campaign-2026-10-08/whiteboard-worker/run-public-form-automation-fairness.mjs:360-383` (synthetic test).

```javascript
  const aSlow1 = await post(peach, 'WB A Slow One', '[redacted email]');
  const aSlow2 = await post(peach, 'WB A Slow Two', '[redacted email]');
  const bAfterSlow = await post(rival, 'WB B After Slow', '[redacted email]');
  assert.deepEqual([aSlow1.status, aSlow2.status, bAfterSlow.status], [200, 200, 200]);
  const slowWork = await fileAndRoute();
  const slowBefore = { messages: queues(), attempts: attempts.length, smtpEvents: smtpEvents.length };
  smtpSlowSuccessDelayMs = 150;
  const slowStartedAt = Date.now();
  const slowDelivered = await waitFor(deliverOutbox(ctx), 15_000, 'two slow SMTP successes then B');
  const slowElapsedMs = Date.now() - slowStartedAt;
  smtpSlowSuccessDelayMs = 0;
  const slowAfter = { messages: queues(), smtpEvents: smtpEvents.slice(slowBefore.smtpEvents),
    slowAcceptedAt, otherTenantProviderAttempts: attempts.slice(slowBefore.attempts).map((item) => ({ ...item })) };
  assert.equal(slowAfter.smtpEvents.filter((event) => event === 'DATA_END_SLOW').length, 2);
  assert.equal(slowAfter.smtpEvents.filter((event) => event === 'DATA_SLOW_ACCEPTED').length, 2);
  assert.ok(slowAfter.messages.some((row) => row.to_address === '[redacted email]' && row.status === 'sent'));
  assert.ok(slowAfter.messages.some((row) => row.to_address === '[redacted email]' && row.status === 'sent'));
  assert.ok(slowAfter.messages.some((row) => row.to_address === '[redacted email]' && row.status === 'sent'));
  assert.ok(slowAfter.otherTenantProviderAttempts.some((item) => item.to === '[redacted email]' && item.accepted));
  assert.ok(slowElapsedMs >= 300, 'two slow successes accumulated before B');
  assert.ok(slowAfter.otherTenantProviderAttempts.find((item) => item.to === '[redacted email]').at >= slowAcceptedAt.at(-1), 'B handoff followed both A accepts');
  result.fairness.twoSlowSuccesses = { posted: [aSlow1.status, aSlow2.status, bAfterSlow.status], work: slowWork,
    delayPerMessageMs: 150, elapsedMs: slowElapsedMs, before: slowBefore,
    delivery: { processed: slowDelivered.processed, errors: slowDelivered.errors }, after: slowAfter };
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A is visited before B, both have due mail, and A's reachable SMTP server stalls during handoff.

Recorded result: A tenant-controlled slow SMTP endpoint held the sequential outbox worker while another tenant's due mail waited. Fast-provider and single-tenant controls showed that the cross-tenant delay came from the shared dispatch order.

Scope: Production egress to owner-configured SMTP was not verified. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/campaign-2026-10-08/whiteboard-worker/run-public-form-automation-fairness.mjs:360-383` (synthetic test).

```javascript
  const aSlow1 = await post(peach, 'WB A Slow One', '[redacted email]');
  const aSlow2 = await post(peach, 'WB A Slow Two', '[redacted email]');
  const bAfterSlow = await post(rival, 'WB B After Slow', '[redacted email]');
  assert.deepEqual([aSlow1.status, aSlow2.status, bAfterSlow.status], [200, 200, 200]);
  const slowWork = await fileAndRoute();
  const slowBefore = { messages: queues(), attempts: attempts.length, smtpEvents: smtpEvents.length };
  smtpSlowSuccessDelayMs = 150;
  const slowStartedAt = Date.now();
  const slowDelivered = await waitFor(deliverOutbox(ctx), 15_000, 'two slow SMTP successes then B');
  const slowElapsedMs = Date.now() - slowStartedAt;
  smtpSlowSuccessDelayMs = 0;
  const slowAfter = { messages: queues(), smtpEvents: smtpEvents.slice(slowBefore.smtpEvents),
    slowAcceptedAt, otherTenantProviderAttempts: attempts.slice(slowBefore.attempts).map((item) => ({ ...item })) };
  assert.equal(slowAfter.smtpEvents.filter((event) => event === 'DATA_END_SLOW').length, 2);
  assert.equal(slowAfter.smtpEvents.filter((event) => event === 'DATA_SLOW_ACCEPTED').length, 2);
  assert.ok(slowAfter.messages.some((row) => row.to_address === '[redacted email]' && row.status === 'sent'));
  assert.ok(slowAfter.messages.some((row) => row.to_address === '[redacted email]' && row.status === 'sent'));
  assert.ok(slowAfter.messages.some((row) => row.to_address === '[redacted email]' && row.status === 'sent'));
  assert.ok(slowAfter.otherTenantProviderAttempts.some((item) => item.to === '[redacted email]' && item.accepted));
  assert.ok(slowElapsedMs >= 300, 'two slow successes accumulated before B');
  assert.ok(slowAfter.otherTenantProviderAttempts.find((item) => item.to === '[redacted email]').at >= slowAcceptedAt.at(-1), 'B handoff followed both A accepts');
  result.fairness.twoSlowSuccesses = { posted: [aSlow1.status, aSlow2.status, bAfterSlow.status], work: slowWork,
    delayPerMessageMs: 150, elapsedMs: slowElapsedMs, before: slowBefore,
    delivery: { processed: slowDelivered.processed, errors: slowDelivered.errors }, after: slowAfter };
```

## Remediation

Give each tenant an independent dispatch lane or bounded send-time budget so later tenants can progress.
