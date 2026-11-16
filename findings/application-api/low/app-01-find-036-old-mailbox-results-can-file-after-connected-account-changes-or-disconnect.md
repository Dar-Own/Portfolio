# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low (conditional)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Acknowledged, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Old mailbox results can file after connected account changes or disconnect

## Description

A stale fetch from a former mailbox can file its message after the owner connects another account or disconnects the mailbox.

Prerequisite and affected workflow: A mailbox fetch begins under account A; the owner connects account B or disconnects before the fetched messages are saved.

Established observation: An owner changed or disconnected a mailbox while an earlier fetch was in flight. The worker still filed the old account's returned message after the change committed; account and connection state were exercised through the owner API and final database path.

Limits: Real Gmail or Graph account transitions and any loss of new-account mail remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A mailbox fetch begins under account A; the owner connects account B or disconnects before the fetched messages are saved.

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

Mailbox synchronization saves fetched messages without rechecking the connected account after the fetch completes.

Source: `apps/worker/src/jobs/mailbox-sync.ts:133-151` (1d5d98db9acf).

```typescript
export async function syncTenantMailbox(pool: pg.Pool, tenantId: string, provider: MailConnector, files: WorkerFileStore, now: Date): Promise<MailboxSyncOutcome> {
  const outcome: MailboxSyncOutcome = { imported: 0, alreadyImported: 0, sentCopies: 0, appLetters: 0, unreadable: 0, note: null, refused: [], busy: false, error: null };

  const state = await withAutomation(pool, tenantId, async (client) => {
    await client.query(`insert into app.mailbox_sync_state (tenant_id, provider) values ($1, $2) on conflict (tenant_id) do nothing`, [tenantId, provider.name]);
    const { rows } = await client.query<SyncStateRow>(`select provider, cursor from app.mailbox_sync_state where tenant_id = $1`, [tenantId]);
    return rows[0]!;
  });
  // Another provider than last time (the server's settings changed): its cursor means nothing here.
  const cursor = state.provider === provider.name ? state.cursor : null;

  let batch: MailboxBatch;
  try {
    batch = await provider.listNew(cursor, SYNC_FOLDERS);
  } catch (error) {
    outcome.error = reason(error);
    await withAutomation(pool, tenantId, (client) => recordError(client, tenantId, outcome.error!, now));
    return outcome;
  }
```

Source: `apps/worker/src/db.ts:37-95` (1d5d98db9acf).

```typescript
/**
 * Runs `fn` in one transaction as the company's Automation member, proven by the worker's own door
 * (app.act_as_automation, migration 0100: only a connection logged in as the worker may use it; the
 * database believes no setting from it). Commits when `fn` resolves, rolls back when it throws.
 * Never from inside another withAutomation (what runs inside one reads on the client it was given): the
 * second act_as_automation waits for the first transaction's aos_ctx rows while the first waits for it,
 * for good (the worker hung that way once in a preview).
 */
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
  } finally {
    client.release(broken);
  }
}

/** What happened when a job went through every active company. */
export interface CompanyScan {
  /** Companies the job went through. */
  companies: number;
  /** Companies where `fn` threw. The other companies still ran. */
  failures: { tenantId: string; error: unknown }[];
}

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


#### Step 3 — Observe the durable effect

An owner changed or disconnected a mailbox while an earlier fetch was in flight. The worker still filed the old account's returned message after the change committed; account and connection state were exercised through the owner API and final database path.

Real Gmail or Graph account transitions and any loss of new-account mail remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/attack-led/workers-bulk/run-remaining-campaign-mailbox-stop.mjs:244-268` (synthetic test).

```javascript
  assert.equal(afterHttp.connection.status, 'revoked');
  assert.equal(afterHttp.pendingMessage.count, 0, 'the stop itself filed the pending message');
  releaseFetch();
  const finished = await bounded(pendingSync, 30_000, 'mailbox worker finalization');
  const afterWorker = { peach: mailboxRows(peach), rival: mailboxRows(rival), state: state(peach), connection: connection(peach), pendingMessage: exactMessage(peach, 'synthetic-a-after-stop') };
  const staleFiled = afterWorker.pendingMessage.count === 1 && afterWorker.pendingMessage.subject === 'A pending at stop'
    && afterWorker.pendingMessage.body?.includes('Harmless synthetic mailbox marker synthetic-a-after-stop.');
  output.race = {
    order: ['connected A credential selected and cursor read', 'A fake provider fetch paused', 'owner password step-up and R194 stop committed', 'A provider response released', 'worker finalization completed'],
    routeStatus: stopped.statusCode, before, afterHttp, workerOutcome: finished, afterWorker, staleFiled, providerCalls,
  };
  assert.equal(finished.errors, 0, JSON.stringify(finished));
  assert.deepEqual(afterWorker.rival, before.rival, 'foreign tenant changed during stop race');
  assert.equal(afterWorker.connection.status, 'revoked');

  // A new scan after the stop is a separate negative control: no new provider A call or row.
  const callsBeforeRetry = aCalls;
  const retry = await runMailboxSync(ctx, options);
  assert.equal(retry.errors, 0, JSON.stringify(retry));
  assert.equal(aCalls, callsBeforeRetry, 'a fresh disconnected scan called A');
  assert.equal(exactMessage(peach, 'synthetic-a-after-stop').count, afterWorker.pendingMessage.count, 'retry changed the pending message');
  const rivalCard = await app.inject({ method: 'GET', url: '/api/inbox/mailbox', headers: rivalOwner.headers });
  assert.equal(rivalCard.statusCode, 200, rivalCard.body);
  assert.equal(rivalCard.json().connected, false, 'foreign tenant connection changed');
  output.controls.afterStopRetry = { outcome: retry, aCalls, peach: mailboxRows(peach), state: state(peach), connection: connection(peach), rivalCardConnected: rivalCard.json().connected };
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A mailbox fetch begins under account A; the owner connects account B or disconnects before the fetched messages are saved.

Recorded result: An owner changed or disconnected a mailbox while an earlier fetch was in flight. The worker still filed the old account's returned message after the change committed; account and connection state were exercised through the owner API and final database path.

Scope: Real Gmail or Graph account transitions and any loss of new-account mail remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/attack-led/workers-bulk/run-remaining-campaign-mailbox-stop.mjs:244-268` (synthetic test).

```javascript
  assert.equal(afterHttp.connection.status, 'revoked');
  assert.equal(afterHttp.pendingMessage.count, 0, 'the stop itself filed the pending message');
  releaseFetch();
  const finished = await bounded(pendingSync, 30_000, 'mailbox worker finalization');
  const afterWorker = { peach: mailboxRows(peach), rival: mailboxRows(rival), state: state(peach), connection: connection(peach), pendingMessage: exactMessage(peach, 'synthetic-a-after-stop') };
  const staleFiled = afterWorker.pendingMessage.count === 1 && afterWorker.pendingMessage.subject === 'A pending at stop'
    && afterWorker.pendingMessage.body?.includes('Harmless synthetic mailbox marker synthetic-a-after-stop.');
  output.race = {
    order: ['connected A credential selected and cursor read', 'A fake provider fetch paused', 'owner password step-up and R194 stop committed', 'A provider response released', 'worker finalization completed'],
    routeStatus: stopped.statusCode, before, afterHttp, workerOutcome: finished, afterWorker, staleFiled, providerCalls,
  };
  assert.equal(finished.errors, 0, JSON.stringify(finished));
  assert.deepEqual(afterWorker.rival, before.rival, 'foreign tenant changed during stop race');
  assert.equal(afterWorker.connection.status, 'revoked');

  // A new scan after the stop is a separate negative control: no new provider A call or row.
  const callsBeforeRetry = aCalls;
  const retry = await runMailboxSync(ctx, options);
  assert.equal(retry.errors, 0, JSON.stringify(retry));
  assert.equal(aCalls, callsBeforeRetry, 'a fresh disconnected scan called A');
  assert.equal(exactMessage(peach, 'synthetic-a-after-stop').count, afterWorker.pendingMessage.count, 'retry changed the pending message');
  const rivalCard = await app.inject({ method: 'GET', url: '/api/inbox/mailbox', headers: rivalOwner.headers });
  assert.equal(rivalCard.statusCode, 200, rivalCard.body);
  assert.equal(rivalCard.json().connected, false, 'foreign tenant connection changed');
  output.controls.afterStopRetry = { outcome: retry, aCalls, peach: mailboxRows(peach), state: state(peach), connection: connection(peach), rivalCardConnected: rivalCard.json().connected };
```

## Remediation

Bind each fetch to a connection generation and verified account identity, then discard obsolete results at the final filing transaction.
