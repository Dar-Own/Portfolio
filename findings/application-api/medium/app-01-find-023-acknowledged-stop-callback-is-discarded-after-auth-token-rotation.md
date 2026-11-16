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

# Acknowledged STOP callback is discarded after Auth Token rotation

## Description

An acknowledged signed STOP can be discarded after an Auth Token change, leaving consent granted and allowing another SMS attempt.

Prerequisite and affected workflow: A recipient's valid STOP is stored before the same-number credential change and the worker processes it afterward.

Established observation: A signed STOP callback was accepted over HTTP, then discarded after the same phone number's Auth Token changed. Consent remained granted and a later outbound SMS reached the test adapter.

Limits: The credential switch used a test-only settings helper. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A recipient's valid STOP is stored before the same-number credential change and the worker processes it afterward.

Source: `apps/api/src/features/texting/routes.ts:70-90` (1d5d98db9acf).

```typescript
  publicRoutes: {
    prefix: '/api/public/texting',
    register(app) {
      // Twilio posts form-encoded fields, kept as sent for the worker. Only these routes accept that content type.
      app.addContentTypeParser('application/x-www-form-urlencoded', { parseAs: 'string', bodyLimit: FORM_BODY_LIMIT }, (_request, body, done) => done(null, String(body)));
      const config = { csrf: false, rateLimit: WEBHOOK_RATE_LIMIT };

      /** An incoming text. Answered with empty TwiML: the app never texts back by itself. */
      app.post('/twilio', { config }, async (request, reply) => {
        await receive(app, 'twilio', request);
        return reply.status(200).type('text/xml').send(EMPTY_TWIML);
      });

      /** A delivery receipt for a text the worker sent through Twilio. */
      app.post('/twilio/status', { config }, async (request, reply) => {
        await receive(app, 'twilio_status', request);
        return reply.status(204).send();
      });
    },
  },
};
```

#### Step 2 — Cross the missing state or authority check

The worker rechecks only the current Auth Token and deletes accepted callbacks signed with the old token.

Source: `apps/worker/src/jobs/webhook-inbox.ts:97-181` (1d5d98db9acf).

```typescript
/**
 * The company's settings, read fresh: never this worker's copy of up to thirty seconds ago, because the
 * webhook that arrives the second after the Connections page saved the keys must see them (a webhook is
 * rare, so the read costs nothing worth caching). Null once the row is dropped because they can't be read
 * at all (its Automation member is disabled or missing): while the row is young that is a later try, not
 * a drop, and the company's other jobs fail too, so it is heard of there.
 */
async function settingsFor(ctx: JobContext, row: InboxRow, tenantId: string, detail: string): Promise<CompanyConnections | null> {
  ctx.connections.forget(tenantId);
  try {
    return await ctx.connections.values(tenantId);
  } catch (error) {
    // The database refusing to act for the company (its Automation member disabled or missing, the company
    // suspended: a raised refusal, code P0001) is not keys on their way, and a flood of forged rows for such
    // a company must not wait anywhere: dropped, as before. Anything else (the database unreachable for a
    // moment) is the loop's later try, like any other failure.
    if ((error as { code?: unknown }).code !== 'P0001') throw error;
    await dropRow(ctx, row, tenantId, detail, `its settings can't be read to check the signature: ${errorText(error)}`);
    return null;
  }
}

async function companyFrom(ctx: JobContext, lookup: string, value: string): Promise<string | null> {
  const { rows } = await ctx.pool.query<{ tenant_id: string | null }>(`select ${lookup}($1) as tenant_id`, [value]);
  return rows[0]?.tenant_id ?? null;
}

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
}

// ---------------------------------------------------------------------------------------------
// Twilio: an incoming text, a delivery receipt
// ---------------------------------------------------------------------------------------------

const E164 = /^\+[1-9][0-9]{7,14}$/;
const MESSAGE_SID = /^(SM|MM)[0-9a-f]{32}$/i;

/** A form-encoded body as the flat field map Twilio posts (a repeated field keeps its last value). */
export function formFields(body: string): WebhookParams {
  const fields: WebhookParams = {};
  for (const [key, value] of new URLSearchParams(body)) fields[key] = value;
  return fields;
}

/** The company's number, its webhook address's slug or a Twilio message id, if the value is one: all a log line names. */
function loggable(value: string | undefined): string {
  return value && /^[+A-Za-z0-9._-]{1,60}$/.test(value) ? value : '?';
}

/** The first field Twilio would never send like this, in plain words, or null. */
function badTwilioFields(fields: WebhookParams, receipt: boolean): string | null {
  if (!MESSAGE_SID.test(fields.MessageSid ?? '')) return 'MessageSid must be a Twilio message id.';
  if (!E164.test(fields.From ?? '')) return 'From must be a phone number in international format.';
  if (receipt) {
    const status = fields.MessageStatus ?? '';
    if (status.length === 0) return 'MessageStatus is missing.';
    if (status.length > 40) return 'MessageStatus is too long.';
    if (fields.ErrorCode !== undefined && !/^\d+$/.test(fields.ErrorCode)) return 'ErrorCode must be a number.';
    return null;
  }
  if (!E164.test(fields.To ?? '')) return 'To must be a phone number in international format.';
  if ((fields.Body ?? '').length > 10_000) return 'Body is too long.';
  if (fields.NumMedia !== undefined && !/^\d+$/.test(fields.NumMedia)) return 'NumMedia must be a number.';
  return null;
}

async function twilio(ctx: JobContext, row: InboxRow): Promise<Handled> {
```


#### Step 3 — Observe the durable effect

A signed STOP callback was accepted over HTTP, then discarded after the same phone number's Auth Token changed. Consent remained granted and a later outbound SMS reached the test adapter.

The credential switch used a test-only settings helper.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/campaign-2026-10-08/workers/run-webhook-rotation.mjs:228-248` (synthetic test).

```javascript
      }
      return realConnections.values(tenantId);
    },
  };
  const poisonDrain = await drain();
  ctx.connections = realConnections;
  const poisonState = { inbox: rows('select provider, attempts, next_attempt_at > now() as backed_off from app.inbound_webhooks'),
    peachConsent: consent(), rivalConsent: consent(rival), peachMessage: message(sid(104)), rivalMessage: message(sid(105)), drain: poisonDrain };
  assert.equal(poisonState.peachConsent?.granted, false, 'poisoned tenant effect deferred');
  assert.equal(poisonState.rivalConsent?.granted, false, 'other tenant processed');
  assert.equal(poisonState.rivalMessage.length, 1, 'foreign-tenant positive callback filed');
  assert.equal(poisonState.inbox.length, 1, 'failed row retained');
  assert.equal(poisonState.inbox[0].attempts, 1, 'failed row backed off');
  timeline.push({ event: 'two_tenant_poison', ...poisonState });
  psql('update app.inbound_webhooks set next_attempt_at = now() where attempts = 1');
  const recoveryDrain = await drain();
  const recoveryState = { inbox: inbox(), peachConsent: consent(), peachMessage: message(sid(104)), drain: recoveryDrain };
  assert.equal(recoveryState.inbox, '0');
  assert.equal(recoveryState.peachConsent?.granted, true);
  assert.equal(recoveryState.peachMessage.length, 1);
  timeline.push({ event: 'poison_recovery', ...recoveryState });
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A recipient's valid STOP is stored before the same-number credential change and the worker processes it afterward.

Recorded result: A signed STOP callback was accepted over HTTP, then discarded after the same phone number's Auth Token changed. Consent remained granted and a later outbound SMS reached the test adapter.

Scope: The credential switch used a test-only settings helper. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/campaign-2026-10-08/workers/run-webhook-rotation.mjs:228-248` (synthetic test).

```javascript
      }
      return realConnections.values(tenantId);
    },
  };
  const poisonDrain = await drain();
  ctx.connections = realConnections;
  const poisonState = { inbox: rows('select provider, attempts, next_attempt_at > now() as backed_off from app.inbound_webhooks'),
    peachConsent: consent(), rivalConsent: consent(rival), peachMessage: message(sid(104)), rivalMessage: message(sid(105)), drain: poisonDrain };
  assert.equal(poisonState.peachConsent?.granted, false, 'poisoned tenant effect deferred');
  assert.equal(poisonState.rivalConsent?.granted, false, 'other tenant processed');
  assert.equal(poisonState.rivalMessage.length, 1, 'foreign-tenant positive callback filed');
  assert.equal(poisonState.inbox.length, 1, 'failed row retained');
  assert.equal(poisonState.inbox[0].attempts, 1, 'failed row backed off');
  timeline.push({ event: 'two_tenant_poison', ...poisonState });
  psql('update app.inbound_webhooks set next_attempt_at = now() where attempts = 1');
  const recoveryDrain = await drain();
  const recoveryState = { inbox: inbox(), peachConsent: consent(), peachMessage: message(sid(104)), drain: recoveryDrain };
  assert.equal(recoveryState.inbox, '0');
  assert.equal(recoveryState.peachConsent?.granted, true);
  assert.equal(recoveryState.peachMessage.length, 1);
  timeline.push({ event: 'poison_recovery', ...recoveryState });
```

## Remediation

Preserve verified authority for accepted callbacks across credential rotation and reconcile the queued STOP before later sends.
