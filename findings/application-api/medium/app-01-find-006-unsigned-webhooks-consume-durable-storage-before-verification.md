# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Medium
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below. A narrow private-lab observation is described below; it does not establish public deployment exposure.
Remediation: Acknowledged, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Unsigned webhooks consume durable storage before verification

## Description

Unsigned webhook bodies reach durable PostgreSQL storage before authentication and can consume queue, WAL, backup, and worker capacity.

Prerequisite and affected workflow: An unauthenticated sender submits repeated accepted-size bodies to a reachable provider webhook.

Established observation: Unsigned callback bodies were written to durable PostgreSQL storage before signature verification. A bounded private-deployment test confirmed the same write path; it did not attempt to exhaust the service.

Limits: The bounded tests did not produce an outage. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

An unauthenticated sender submits repeated accepted-size bodies to a reachable provider webhook.

Source: `apps/api/src/features/invoices/routes.ts:79-82` (1d5d98db9acf).

```typescript
/** The company's slug at the end of its Stripe webhook address. */
const companySlug = z.string().min(1).max(100).regex(/^[A-Za-z0-9._-]+$/);
/** Stripe's events are a few kilobytes; anything bigger than this is not one. */
const STRIPE_BODY_LIMIT = 256 * 1024;
```

#### Step 2 — Cross the missing state or authority check

The public Stripe route accepts up to 256 KiB and persists the body before verification: `apps/api/src/features/invoices/routes.ts:79-82,333-353`. `app.receive_webhook` inserts unconditionally with no global row/byte quota: `packages/db/migrations/0100_verified_identity.sql:1303-1331`. Worker verification happens only after durable admission: `apps/worker/src/jobs/webhook-inbox.ts:49-95,437-530`.

Source: `packages/db/migrations/0100_verified_identity.sql:1303-1331` (1d5d98db9acf).

```sql
create table app.inbound_webhooks (
  id          bigint generated always as identity primary key,
  provider    text not null check (provider in ('stripe', 'twilio', 'twilio_status')),
  route       text not null check (length(route) <= 2000),          -- path + query, for Twilio's signature
  signature   text check (length(signature) <= 2000),
  body        text not null check (length(body) <= 262144),
  received_at timestamptz not null default now(),
  -- A genuine webhook whose handling failed (Stripe not answering) is tried again later, each time
  -- later still, so rows that keep failing fall behind new ones instead of holding up the inbox.
  attempts        int not null default 0,
  next_attempt_at timestamptz not null default now()
);
create index inbound_webhooks_due on app.inbound_webhooks (next_attempt_at, id);
comment on table app.inbound_webhooks is
  'Webhooks as Stripe and Twilio posted them, not yet checked: the API adds them (app.receive_webhook), the worker checks each with its company''s own secret, acts and deletes it (migration 0100).';
revoke all on table app.inbound_webhooks from public;
grant select, delete, update (attempts, next_attempt_at) on table app.inbound_webhooks to app_worker;
-- Every app table has row-level security, enabled and forced; this one belongs to no company, so its one
-- policy is the worker's (receive_webhook runs as the owner, which row-level security does not stop).
alter table app.inbound_webhooks enable row level security;
alter table app.inbound_webhooks force row level security;
create policy worker_only on app.inbound_webhooks to app_worker using (true);

create function app.receive_webhook(p_provider text, p_route text, p_signature text, p_body text) returns void
language sql security definer set search_path = app, pg_temp as $$
  insert into app.inbound_webhooks (provider, route, signature, body) values (p_provider, p_route, p_signature, p_body)
$$;
revoke all on function app.receive_webhook(text, text, text, text) from public;
grant execute on function app.receive_webhook(text, text, text, text) to app_auth;
```

Source: `apps/worker/src/jobs/webhook-inbox.ts:49-95` (1d5d98db9acf).

```typescript
/** How many rows are read at a time. */
const BATCH = 100;
/** How long one run keeps taking rows; the next run takes the rest. */
const BUDGET_MS = 8_000;
/** A failed row's next try: 10 seconds after its first failure, doubling, at most an hour later. */
function retryDelaySeconds(attempts: number): number {
  return Math.min(3600, 10 * 2 ** attempts);
}
/** Stripe retries a webhook for three days; a row still failing after that is given up. */
const GIVE_UP_AFTER_MS = 3 * 24 * 60 * 60 * 1000;

/** What became of one row. */
type Handled = { tenantId: string | null; outcome: 'handled' | 'dropped' | 'taken' };

/** Thrown inside a company's transaction for a request that is not that company's genuine webhook. */
class Dropped {
  readonly reason: string;
  constructor(reason: string) {
    this.reason = reason;
  }
}

/** Deletes a row that is not a company's genuine webhook, before any transaction of the company, and logs why. */
async function dropRow(ctx: JobContext, row: InboxRow, tenantId: string | null, detail: string, reason: string): Promise<Handled> {
  await ctx.pool.query('delete from app.inbound_webhooks where id = $1', [row.id]);
  ctx.logger.info(`webhook-inbox: ${row.provider === 'stripe' ? 'Stripe' : 'Twilio'} webhook dropped (${detail}): ${reason}`);
  return { tenantId, outcome: 'dropped' };
}

/**
 * How long a webhook whose company's keys are missing or unreadable is kept for a later try, counted from
 * when it arrived. The Connections page saves a company's keys in the API's process; this worker reads
 * them fresh (settingsFor), but Stripe's first event or a customer's text can still land before that save
 * commits, and neither must be thrown away for it: Stripe says it was received, and Twilio never sends a
 * text twice. Older than this, the keys are really not there, and the row is dropped as a forgery would be.
 */
const KEYS_SETTLE_MS = 10 * 60_000;

/**
 * A company's webhook that can't be checked because its keys are missing or unreadable: while it is young,
 * kept for a later try (the loop's backoff; its error line says why), and dropped once it is old. Rows a
 * forger sends for a keyless company therefore wait ten minutes at most, behind newer rows, never for days.
 */
async function keysNotThere(ctx: JobContext, row: InboxRow, tenantId: string, detail: string, reason: string): Promise<Handled> {
  if (Date.now() - row.received_at.getTime() < KEYS_SETTLE_MS) throw new Error(`${reason}; kept for a later try, in case the keys were just saved`);
  return dropRow(ctx, row, tenantId, detail, reason);
}
```


#### Step 3 — Observe the durable effect

Unsigned callback bodies were written to durable PostgreSQL storage before signature verification. A bounded private-deployment test confirmed the same write path; it did not attempt to exhaust the service.

The bounded tests did not produce an outage.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/pocs/run-leads-003-010.mjs:476-499` (test source).

```javascript
function poc008WebhookPreverificationStorage(pg) {
  pg.psql('truncate app.inbound_webhooks;');
  const before = Number(pg.psql(`select pg_total_relation_size('app.inbound_webhooks');`).stdout);
  pg.psql(`
    do $$
    begin
      for i in 1..16 loop
        perform app.receive_webhook(
          'stripe',
          '/api/public/payments/webhook/poc-company',
          null,
          (select string_agg(md5(i::text || ':' || g::text), '' order by g) from generate_series(1, 8192) g)
        );
      end loop;
    end $$;
  `);
  const [rows, bodyBytes, relationBytes] = pg.psql(`
    select count(*) || '|' || sum(octet_length(body)) || '|' || pg_total_relation_size('app.inbound_webhooks')
      from app.inbound_webhooks;
  `).stdout.split('|').map(Number);
  assert.equal(rows, 16);
  assert.equal(bodyBytes, 16 * 262144);
  assert.ok(relationBytes - before > 3 * 1024 * 1024);
  const oversized = pg.psql(`select app.receive_webhook('stripe', '/poc', null, repeat('x', 262145));`, { allowFailure: true });
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: An unauthenticated sender submits repeated accepted-size bodies to a reachable provider webhook.

Recorded result: Unsigned callback bodies were written to durable PostgreSQL storage before signature verification. A bounded private-deployment test confirmed the same write path; it did not attempt to exhaust the service.

Scope: The bounded tests did not produce an outage. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/pocs/run-leads-003-010.mjs:476-499` (test source).

```javascript
function poc008WebhookPreverificationStorage(pg) {
  pg.psql('truncate app.inbound_webhooks;');
  const before = Number(pg.psql(`select pg_total_relation_size('app.inbound_webhooks');`).stdout);
  pg.psql(`
    do $$
    begin
      for i in 1..16 loop
        perform app.receive_webhook(
          'stripe',
          '/api/public/payments/webhook/poc-company',
          null,
          (select string_agg(md5(i::text || ':' || g::text), '' order by g) from generate_series(1, 8192) g)
        );
      end loop;
    end $$;
  `);
  const [rows, bodyBytes, relationBytes] = pg.psql(`
    select count(*) || '|' || sum(octet_length(body)) || '|' || pg_total_relation_size('app.inbound_webhooks')
      from app.inbound_webhooks;
  `).stdout.split('|').map(Number);
  assert.equal(rows, 16);
  assert.equal(bodyBytes, 16 * 262144);
  assert.ok(relationBytes - before > 3 * 1024 * 1024);
  const oversized = pg.psql(`select app.receive_webhook('stripe', '/poc', null, repeat('x', 262145));`, { allowFailure: true });
```

Source: `security-audit/pocs/run-leads-003-010.mjs:497-511` (test source).

```javascript
  assert.equal(bodyBytes, 16 * 262144);
  assert.ok(relationBytes - before > 3 * 1024 * 1024);
  const oversized = pg.psql(`select app.receive_webhook('stripe', '/poc', null, repeat('x', 262145));`, { allowFailure: true });
  assert.notEqual(oversized.status, 0);
  assert.match(oversized.stderr, /inbound_webhooks_body_check/);

  const route = readFileSync(path.join(repo, 'apps/api/src/features/invoices/routes.ts'), 'utf8');
  const worker = readFileSync(path.join(repo, 'apps/worker/src/jobs/webhook-inbox.ts'), 'utf8');
  assert.ok(route.indexOf("receive_webhook('stripe'") >= 0 || route.indexOf('app.receive_webhook') >= 0);
  assert.match(worker, /row\.provider === 'stripe' \? await stripe\(ctx, row\)/);
  assert.match(worker, /provider\.verifyStripeWebhook\(row\.body, row\.signature\)/);
  return {
    verdict: 'storage-primitive-proved',
    unsignedRowsStored: rows,
    rawBodyBytes: bodyBytes,
```

## Remediation

Verify signatures before durable admission where possible and enforce global and per-tenant byte, row, and retention budgets.
