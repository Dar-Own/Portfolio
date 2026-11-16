# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Medium
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and a partial or modeled proof of the original behavior; the complete application or provider flow is not established by these excerpts.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Failed domain-event handlers are permanently acknowledged

## Description

A failed domain-event handler is permanently acknowledged. A voided invoice can retain an open checkout, leaving local and provider payment state out of sync.

Prerequisite and affected workflow: An authorized user voids an invoice with an open checkout while its expiration handler fails transiently.

Established observation: A throwing domain-event handler was recorded as a returned failure while the worker acknowledged the event. The invoice-void path uses that handler to expire an open checkout, so a transient failure can leave the expiration without a retry.

Limits: A provider charge after failed expiration was not exercised. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

An authorized user voids an invoice with an open checkout while its expiration handler fails transiently.

Source: `apps/worker/src/jobs/domain-events.ts:74-102` (1d5d98db9acf).

```typescript
/** One event, in its company's Automation transaction. False when another run already had it. */
async function handleEvent(options: DrainOptions, row: EventRow): Promise<{ taken: boolean; failures: number }> {
  return withAutomation(options.pool, row.tenant_id, async (client) => {
    const claim = await client.query('delete from app.domain_events where id = $1', [row.id]);
    if (claim.rowCount !== 1) return { taken: false, failures: 0 };
    const { rows } = await client.query<{ user_id: string }>('select app.current_user_id() as user_id');
    const ctx: DbContext = {
      client,
      tenantId: row.tenant_id,
      userId: rows[0]!.user_id,
      actor: 'automation',
      devTools: options.devTools,
      emit: () => {
        throw new Error('runEventHandlers gives each handler its own emit');
      },
    };
    const emitted: { name: DomainEventName; subject: string }[] = [];
    const event = { tenantId: row.tenant_id, name: row.name, subjectKey: row.subject_key ?? 'subjectId', subject: row.subject, actorId: row.actor_id };
    const failures = await runEventHandlers(event, options.registry, ctx, emitted);
    for (const failure of failures) {
      options.logger.error(`domain-events: event ${row.id} (${row.name}): the ${failure.source} handler failed and was undone: ${errorText(failure.error)}`);
    }
    if (emitted.length > 0 && row.depth + 1 > MAX_EVENT_DEPTH) {
      options.logger.error(`domain-events: event ${row.id} (${row.name}): its handlers kept emitting events; stopped at depth ${MAX_EVENT_DEPTH} to avoid a loop.`);
    } else {
      for (const next of emitted) await client.query('select app.emit_event($1, $2::uuid, $3)', [next.name, next.subject, row.depth + 1]);
    }
    return { taken: true, failures: failures.length };
  });
```

#### Step 2 — Cross the missing state or authority check

The worker deletes the durable event before invoking handlers. `runEventHandlers` catches handler exceptions and returns them instead of rejecting the worker attempt, so the surrounding retry path never retains the event: `apps/worker/src/jobs/domain-events.ts:74-102,129-145`; `apps/api/src/core/events.ts:121-146`. The invoice handler's relevant sink is hosted-checkout expiration: `apps/api/src/features/invoices/events.ts:13-24`; `apps/api/src/features/invoices/checkout.ts:480-505`.

Source: `apps/api/src/core/events.ts:121-146` (1d5d98db9acf).

```typescript
/**
 * Runs every handler of one event on ctx, each in a savepoint of its own, handing it ONLY the ids the
 * database recorded. The events a handler emits are kept only when it succeeds. Returns the failures.
 */
export async function runEventHandlers(
  event: StoredEvent,
  registry: EventRegistry,
  ctx: DbContext,
  emitted: { name: DomainEventName; subject: string }[],
): Promise<HandlerFailure[]> {
  const failures: HandlerFailure[] = [];
  for (const { source, handler } of registry.handlersFor(event.name)) {
    const payload: DomainEventPayload = { tenantId: event.tenantId, actorUserId: event.actorId, [event.subjectKey]: event.subject };
    const own: { name: DomainEventName; subject: string }[] = [];
    const handlerCtx: DbContext = { ...ctx, emit: (name, subject) => void own.push({ name, subject }) };
    await ctx.client.query('savepoint event_handler');
    try {
      await handler(payload, handlerCtx);
      await ctx.client.query('release savepoint event_handler');
      emitted.push(...own);
    } catch (error) {
      await ctx.client.query('rollback to savepoint event_handler');
      failures.push({ source, error });
    }
  }
  return failures;
```

Source: `apps/api/src/features/invoices/events.ts:13-24` (1d5d98db9acf).

```typescript
const handlers: EventHandlers = {
  'invoice.voided': async (payload, ctx) => {
    const invoiceId = typeof payload.invoiceId === 'string' && UUID.test(payload.invoiceId) ? payload.invoiceId : null;
    if (!invoiceId) return;
    const { rows } = await ctx.client.query<{ status: string }>('select status::text as status from app.invoices where tenant_id = $1 and id = $2', [ctx.tenantId, invoiceId]);
    if (rows[0]?.status !== 'void') return;
    const provider = await paymentProviderFor(ctx);
    if (!provider) return;
    await lockInvoiceCheckouts(ctx, invoiceId);
    // The void closed our records ('canceled'); at the provider those pages may still be open.
    await closePendingCheckoutsIn(ctx, { info: () => {}, error: () => {} }, provider, invoiceId, ['pending', 'canceled']);
  },
```


#### Step 3 — Observe the durable effect

A throwing domain-event handler was recorded as a returned failure while the worker acknowledged the event. The invoice-void path uses that handler to expire an open checkout, so a transient failure can leave the expiration without a retry.

A provider charge after failed expiration was not exercised.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/pocs/run-leads-003-010.mjs:150-173` (test source).

```javascript
async function poc003DomainEventAcknowledgement() {
  const eventsUrl = pathToFileURL(path.join(repo, 'apps/api/src/core/events.ts')).href;
  const { createEventRegistry, runEventHandlers } = await import(eventsUrl);
  const registry = createEventRegistry();
  registry.register({
    'invoice.voided': async () => {
      throw new Error('POC transient provider expiration failure');
    },
  }, 'poc-expiration');
  const queries = [];
  const client = {
    async query(text) {
      queries.push(text);
      return { rows: [], rowCount: 1 };
    },
  };
  const failures = await runEventHandlers(
    {
      tenantId: '00000000-0000-0000-0000-000000000001',
      name: 'invoice.voided',
      subjectKey: 'invoiceId',
      subject: '00000000-0000-0000-0000-000000000002',
      actorId: null,
    },
```

## Proof of concept

The original proof isolates a narrower component or modeled transition. It does not demonstrate every step of the complete external workflow.

Setup: An authorized user voids an invoice with an open checkout while its expiration handler fails transiently.

Recorded result: A throwing domain-event handler was recorded as a returned failure while the worker acknowledged the event. The invoice-void path uses that handler to expire an open checkout, so a transient failure can leave the expiration without a retry.

Scope: A provider charge after failed expiration was not exercised. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/pocs/run-leads-003-010.mjs:150-173` (test source).

```javascript
async function poc003DomainEventAcknowledgement() {
  const eventsUrl = pathToFileURL(path.join(repo, 'apps/api/src/core/events.ts')).href;
  const { createEventRegistry, runEventHandlers } = await import(eventsUrl);
  const registry = createEventRegistry();
  registry.register({
    'invoice.voided': async () => {
      throw new Error('POC transient provider expiration failure');
    },
  }, 'poc-expiration');
  const queries = [];
  const client = {
    async query(text) {
      queries.push(text);
      return { rows: [], rowCount: 1 };
    },
  };
  const failures = await runEventHandlers(
    {
      tenantId: '00000000-0000-0000-0000-000000000001',
      name: 'invoice.voided',
      subjectKey: 'invoiceId',
      subject: '00000000-0000-0000-0000-000000000002',
      actorId: null,
    },
```

Source: `security-audit/pocs/run-leads-003-010.mjs:180-194` (test source).

```javascript
  assert.deepEqual(queries, ['savepoint event_handler', 'rollback to savepoint event_handler']);

  const worker = readFileSync(path.join(repo, 'apps/worker/src/jobs/domain-events.ts'), 'utf8');
  const invoiceHandler = readFileSync(path.join(repo, 'apps/api/src/features/invoices/events.ts'), 'utf8');
  const checkout = readFileSync(path.join(repo, 'apps/api/src/features/invoices/checkout.ts'), 'utf8');
  const deletion = worker.indexOf("delete from app.domain_events where id = $1");
  const handling = worker.indexOf('const failures = await runEventHandlers', deletion);
  const successfulReturn = worker.indexOf('return { taken: true, failures: failures.length }', handling);
  assert.ok(deletion >= 0 && handling > deletion && successfulReturn > handling);
  assert.match(invoiceHandler, /await closePendingCheckoutsIn\([^;]+\['pending', 'canceled'\]\);/s);
  assert.match(checkout, /await provider\.expireCheckout\(id\)/);

  return {
    verdict: 'mechanism-proved',
    handlerRejected: true,
```

## Remediation

Keep each required handler attempt durable and retryable until it succeeds, and make checkout expiration idempotent.
