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

# Suspended company messages remain eligible for delivery

## Description

A suspended tenant's pending email or SMS can still be handed to its delivery adapter. If the handoff succeeds but the database transaction rolls back, the message can be attempted again after reactivation.

Prerequisite and affected workflow: Staff suspend the tenant after the outbox worker selects its message but before delivery handoff.

Established observation: Staff suspended a company after its outbound message was selected but before provider handoff; both email and SMS still reached test adapters. Their local transactions rolled back, and reactivation caused another acceptance of the same queued work.

Limits: Real SMTP or Twilio acceptance was not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

Staff suspend the tenant after the outbox worker selects its message but before delivery handoff.

Source: `apps/worker/src/jobs/outbox-delivery.ts:279-291` (1d5d98db9acf).

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
```

#### Step 2 — Cross the missing state or authority check

The outbox worker checks tenant activity at selection but sends from cached work after suspension commits.

Source: `apps/worker/src/db.ts:45-66` (1d5d98db9acf).

```typescript
export async function withAutomation<T>(pool: pg.Pool, tenantId: string, fn: (client: pg.PoolClient) => Promise<T>): Promise<T> {
  const client = await pool.connect();
  let broken: Error | undefined;
  try {
    await client.query('begin');
    // Refuses, in the worker's own words, a company with no Automation member, a suspended company and a disabled member.
    await client.query('select app.act_as_automation($1::uuid)', [tenantId]);
    const member = await client.query<{ ok: boolean }>('select app.is_member() as ok');
    if (!member.rows[0]?.ok) {
      throw new CannotActError(`The Automation member of company ${tenantId} can't act: its account is disabled or the company is suspended.`);
    }
    const result = await fn(client);
    await client.query('commit');
    return result;
  } catch (error) {
    try {
      await client.query('rollback');
    } catch (rollbackError) {
      // The connection is unusable: throw it away instead of returning it to the pool.
      broken = rollbackError as Error;
    }
    throw error;
```


#### Step 3 — Observe the durable effect

Staff suspended a company after its outbound message was selected but before provider handoff; both email and SMS still reached test adapters. Their local transactions rolled back, and reactivation caused another acceptance of the same queued work.

Real SMTP or Twilio acceptance was not tested.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/pocs/run-after-door-suspension.mjs:148-171` (test source).

```javascript
  assert.deepEqual(positive.accepted.map((x) => x.channel).sort(), ['email', 'sms']);
  result.scenarios.push({ name: 'unsuspended-positive', ...positive });

  await suspend();
  const preEmail = enqueue(peach, 'email', 'pre-suspended email');
  const preSms = enqueue(peach, 'sms', 'pre-suspended sms');
  const foreignEmail = enqueue(rival, 'email', 'foreign independent email');
  const preStart = accepted.length;
  await deliverOutbox(ctx);
  const pre = { email: state(preEmail), sms: state(preSms), foreign: state(foreignEmail), accepted: accepted.slice(preStart) };
  assert.equal(pre.email.status, 'queued'); assert.equal(pre.sms.status, 'queued');
  assert.equal(pre.foreign.status, 'sent');
  assert.equal(pre.accepted.length, 1);
  result.scenarios.push({ name: 'suspended-before-fresh-scan-foreign-continues', ...pre });
  cleanQueued(); await reactivate();

  async function race(channel) {
    cleanQueued();
    const id = enqueue(peach, channel, `paused ${channel}`);
    const before = state(id);
    const acceptedBefore = accepted.length;
    const logBefore = logLines.length;
    armed = barrier(channel);
    const pending = deliverOutbox(ctx);
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: Staff suspend the tenant after the outbox worker selects its message but before delivery handoff.

Recorded result: Staff suspended a company after its outbound message was selected but before provider handoff; both email and SMS still reached test adapters. Their local transactions rolled back, and reactivation caused another acceptance of the same queued work.

Scope: Real SMTP or Twilio acceptance was not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/pocs/run-after-door-suspension.mjs:148-171` (test source).

```javascript
  assert.deepEqual(positive.accepted.map((x) => x.channel).sort(), ['email', 'sms']);
  result.scenarios.push({ name: 'unsuspended-positive', ...positive });

  await suspend();
  const preEmail = enqueue(peach, 'email', 'pre-suspended email');
  const preSms = enqueue(peach, 'sms', 'pre-suspended sms');
  const foreignEmail = enqueue(rival, 'email', 'foreign independent email');
  const preStart = accepted.length;
  await deliverOutbox(ctx);
  const pre = { email: state(preEmail), sms: state(preSms), foreign: state(foreignEmail), accepted: accepted.slice(preStart) };
  assert.equal(pre.email.status, 'queued'); assert.equal(pre.sms.status, 'queued');
  assert.equal(pre.foreign.status, 'sent');
  assert.equal(pre.accepted.length, 1);
  result.scenarios.push({ name: 'suspended-before-fresh-scan-foreign-continues', ...pre });
  cleanQueued(); await reactivate();

  async function race(channel) {
    cleanQueued();
    const id = enqueue(peach, channel, `paused ${channel}`);
    const before = state(id);
    const acceptedBefore = accepted.length;
    const logBefore = logLines.length;
    armed = barrier(channel);
    const pending = deliverOutbox(ctx);
```

## Remediation

Recheck suspension at final handoff and reconcile accepted sends without replay after a concurrent suspension.
