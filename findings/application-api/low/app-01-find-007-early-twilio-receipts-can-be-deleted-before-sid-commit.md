# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Early Twilio receipts can be deleted before SID commit

## Description

An early Twilio receipt can be deleted before its message SID commits, leaving delivery status stale.

Prerequisite and affected workflow: A final status callback is processed while the sender transaction still holds the returned SID uncommitted.

Established observation: When a signed delivery receipt was processed before its outbound message identifier committed, the worker deleted the unmatched receipt. The later committed message kept stale delivery status under the forced ordering.

Limits: Real Twilio callback timing and later callback guarantees were not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A final status callback is processed while the sender transaction still holds the returned SID uncommitted.

Source: `apps/worker/src/jobs/outbox-delivery.ts:855-880` (1d5d98db9acf).

```typescript
/** Sends every queued text of one company (marks each sent, or blocked, or failed). Delivery is marked by markTextsDelivered. */
export async function sendQueuedTexts(ctx: JobContext, tenantId: string, now: Date, tally: Tally, company?: CompanyOutbound): Promise<void> {
  const outbound = company ?? (await companyOutbound(ctx, tenantId));
  const tried: string[] = [];
  for (let i = 0; i < MAX_PER_RUN; i++) {
    const picked = await withAutomation(ctx.pool, tenantId, async (client) => {
      const message = await pickQueued(client, tenantId, 'sms', tried);
      if (!message) return false;
      tried.push(message.id);

      // Marking it sent re-checks consent in the database. Without consent it comes back blocked: don't send.
      const marked = await client.query<{ status: string }>(
        `update app.messages set status = 'sent', sent_at = $3, blocked_reason = null
          where tenant_id = $1 and id = $2 and status = 'queued'
          returning status`,
        [tenantId, message.id, now],
      );
      const status = marked.rows[0]?.status;
      if (status === 'blocked') {
        tally.textsBlocked++;
        return true;
      }
      if (status !== 'sent') {
        throw new Error(`Text ${message.id} could not be marked sent.`);
      }

```

#### Step 2 — Cross the missing state or authority check

The outbox calls the provider and stores the SID within one still-uncommitted transaction: `apps/worker/src/jobs/outbox-delivery.ts:855-880,894-934`. Receipt processing looks up only committed SID state and permanently deletes unknown inbox rows: `apps/worker/src/jobs/webhook-inbox.ts:124-142,198-210,304-327`.

Source: `apps/worker/src/jobs/webhook-inbox.ts:124-142` (1d5d98db9acf).

```typescript
/**
 * Deletes the row inside the company's transaction and runs fn there. fn throws Dropped for a request
 * that is not the company's genuine webhook: the delete stays and the reason is logged. 'taken' when
 * another run already has the row.
 */
async function claimed(ctx: JobContext, row: InboxRow, tenantId: string, detail: string, fn: (client: pg.PoolClient) => Promise<string>): Promise<Handled> {
  const result = await withAutomation(ctx.pool, tenantId, async (client) => {
    const claim = await client.query('delete from app.inbound_webhooks where id = $1', [row.id]);
    if (claim.rowCount !== 1) return { outcome: 'taken' as const };
    try {
      return { outcome: 'handled' as const, said: await fn(client) };
    } catch (error) {
      if (!(error instanceof Dropped)) throw error;
      return { outcome: 'dropped' as const, reason: error.reason };
    }
  });
  if (result.outcome === 'dropped') ctx.logger.info(`webhook-inbox: ${row.provider === 'stripe' ? 'Stripe' : 'Twilio'} webhook dropped (${detail}): ${result.reason}`);
  else if (result.outcome === 'handled') ctx.logger.info(`webhook-inbox: ${row.provider === 'stripe' ? 'Stripe' : 'Twilio'} webhook handled (${detail}): ${result.said}`);
  return { tenantId, outcome: result.outcome };
```


#### Step 3 — Observe the durable effect

When a signed delivery receipt was processed before its outbound message identifier committed, the worker deleted the unmatched receipt. The later committed message kept stale delivery status under the forced ordering.

Real Twilio callback timing and later callback guarantees were not tested.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/pocs/run-leads-003-010.mjs:518-541` (test source).

```javascript
async function poc009TwilioReceiptCommitRace(pg, setup) {
  const tenant = setup.tenantId;
  const tenantSql = sqlLiteral(tenant);
  const victim = '+14705550199';
  const company = '+14705550100';
  const messageId = '00000000-0000-0000-0000-000000000909';
  const sid = `SM${'a'.repeat(32)}`;
  pg.psql(`
    insert into app.consent_events (tenant_id, channel, address, granted, method, evidence)
    values (${tenantSql}::uuid, 'sms', ${sqlLiteral(victim)}, true, 'written', 'PoC receipt race');
    insert into app.messages (id, tenant_id, channel, direction, to_address, from_address, body)
    values (${sqlLiteral(messageId)}::uuid, ${tenantSql}::uuid, 'sms', 'outbound', ${sqlLiteral(victim)}, ${sqlLiteral(company)}, 'PoC. Reply STOP to opt out.');
    select app.receive_webhook('twilio_status', '/api/public/texting/status/poc', 'valid-signature-assumed', 'MessageSid=${sid}');
  `);
  const inboxId = pg.psql(`select max(id) from app.inbound_webhooks where provider = 'twilio_status';`).stdout;
  const sender = pg.psqlAsync(`
    begin;
    update app.messages set status = 'sent', sent_at = now() where id = ${sqlLiteral(messageId)}::uuid and status = 'queued';
    select pg_sleep(3);
    update app.messages set provider_message_id = ${sqlLiteral(sid)} where id = ${sqlLiteral(messageId)}::uuid;
    commit;
  `);
  await wait(500);
  const receipt = await pg.psqlAsync(`
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A final status callback is processed while the sender transaction still holds the returned SID uncommitted.

Recorded result: When a signed delivery receipt was processed before its outbound message identifier committed, the worker deleted the unmatched receipt. The later committed message kept stale delivery status under the forced ordering.

Scope: Real Twilio callback timing and later callback guarantees were not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/pocs/run-leads-003-010.mjs:518-541` (test source).

```javascript
async function poc009TwilioReceiptCommitRace(pg, setup) {
  const tenant = setup.tenantId;
  const tenantSql = sqlLiteral(tenant);
  const victim = '+14705550199';
  const company = '+14705550100';
  const messageId = '00000000-0000-0000-0000-000000000909';
  const sid = `SM${'a'.repeat(32)}`;
  pg.psql(`
    insert into app.consent_events (tenant_id, channel, address, granted, method, evidence)
    values (${tenantSql}::uuid, 'sms', ${sqlLiteral(victim)}, true, 'written', 'PoC receipt race');
    insert into app.messages (id, tenant_id, channel, direction, to_address, from_address, body)
    values (${sqlLiteral(messageId)}::uuid, ${tenantSql}::uuid, 'sms', 'outbound', ${sqlLiteral(victim)}, ${sqlLiteral(company)}, 'PoC. Reply STOP to opt out.');
    select app.receive_webhook('twilio_status', '/api/public/texting/status/poc', 'valid-signature-assumed', 'MessageSid=${sid}');
  `);
  const inboxId = pg.psql(`select max(id) from app.inbound_webhooks where provider = 'twilio_status';`).stdout;
  const sender = pg.psqlAsync(`
    begin;
    update app.messages set status = 'sent', sent_at = now() where id = ${sqlLiteral(messageId)}::uuid and status = 'queued';
    select pg_sleep(3);
    update app.messages set provider_message_id = ${sqlLiteral(sid)} where id = ${sqlLiteral(messageId)}::uuid;
    commit;
  `);
  await wait(500);
  const receipt = await pg.psqlAsync(`
```

Source: `security-audit/pocs/run-leads-003-010.mjs:561-575` (test source).

```javascript
       and channel = 'sms' and direction = 'outbound';
  `).stdout;
  assert.equal(afterCommitLookup, '1');

  const sendSource = readFileSync(path.join(repo, 'apps/worker/src/jobs/outbox-delivery.ts'), 'utf8');
  const receiptSource = readFileSync(path.join(repo, 'apps/worker/src/jobs/webhook-inbox.ts'), 'utf8');
  const sendCall = sendSource.indexOf('await outbound.sms.send');
  const sidWrite = sendSource.indexOf('set provider_message_id = $3', sendCall);
  assert.ok(sendCall >= 0 && sidWrite > sendCall);
  assert.match(receiptSource, /if \(!message\) return 'not_found'/);
  assert.match(receiptSource, /if \(result === 'not_found'\) throw new Dropped/);
  return {
    verdict: 'forced-order-race-proved',
    lookupBeforeSenderCommit: 0,
    afterBothCommit: final,
```

## Remediation

Retain unmatched signed receipts for bounded retry and reconcile them after the SID commits.
