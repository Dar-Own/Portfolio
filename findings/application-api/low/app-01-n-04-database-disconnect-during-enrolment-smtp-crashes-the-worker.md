# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy
Evidence: Source inspection and recorded regression tests for corrected behavior; these excerpts are not a reproduction of the original failure.
Remediation: Resolved in reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Database disconnect during enrolment SMTP crashes the worker

## Description

A lost database connection during enrolment mail can terminate the worker and delay unrelated scheduled business work until restart.

Prerequisite: The checked-out enrolment connection fails while SMTP handoff is held.

Recorded result: Native PostgreSQL backend termination reproduced the worker failure. Patched lifecycle tests cover disconnect timing, refusal, rollback failure and subsequent work without changing delivery retry policy.

Evidence limits: A native PostgreSQL connection was terminated under controlled timing with synthetic SMTP. Worker availability is established for that failure mode; provider delivery and ordinary incident frequency are unverified. SMTP acknowledgement ambiguity remains a separate accepted behavior.

### Mechanism step by step

#### Step 1 — The worker checks out a database client across SMTP handoff

The checked-out enrolment connection fails while SMTP handoff is held.

Source: `apps/worker/src/jobs/enrolment-codes.ts:61-86` (be12d54).

```typescript
    // the person still an enabled member. A refusal drops the code in the database and marks the letter. The
    // code is handed over inside the transaction that asked, which holds the company's row (migration 0126), so
    // the person taken out of the company or disabled meanwhile waits for the hand-over instead of racing it.
    const client = await ctx.pool.connect();
    let broken: Error | undefined;
    try {
      await client.query('begin');
      const check = await askAsItLeaves<{ reason: string | null }>(client, 'select app.enrolment_letter_may_leave($1) as reason', [letter.id]);
      if (check.rows[0]?.reason) {
        await client.query('commit');
        stopped++;
        continue;
      }
      const message = enrolmentCodeEmail({ fullName: letter.full_name, companyName: letter.company, code: letter.code, minutes: letter.minutes });
      let accepted = false;
      try {
        // The company's name and its brand's address; the install's own mail service puts its own on instead
        // (SmtpEmailSenderOptions.fromName and from, set by SERVER_SMTP_FROM).
        const result = await sender.send({
          from: { name: letter.company, address: letter.sender ?? 'no-reply@[redacted name].invalid' },
          to: { name: letter.full_name, address: letter.to_address },
          subject: message.subject,
          text: message.text,
        });
        accepted = result.accepted.length > 0;
        if (!accepted) ctx.logger.error("enrolment-codes: the install's own mail service refused a code's recipient");
```

#### Step 2 — Connection failure can escape the awaited transaction error path

The client can emit an unhandled connection error during the handoff transaction.

Source: `apps/worker/src/jobs/enrolment-codes.ts:94-111` (be12d54).

```typescript
    } catch (error) {
      try {
        await client.query('rollback');
      } catch (rollbackError) {
        broken = rollbackError as Error;
      }
      // A change of the person's company held open past LETTER_TURN_WAIT (0126): this code waits for the next run
      // (taken again a minute later), and the other codes go now.
      if (isLockTimeout(error) && !broken) {
        waited++;
        held.add(letter.company);
        ctx.logger.error('enrolment-codes: a code waited too long for a change of its company being saved; it is sent on a later run');
        continue;
      }
      throw error;
    } finally {
      client.release(broken);
    }
```

#### Step 3 — Observe the effect and its correction

Native PostgreSQL backend termination reproduced the worker failure. Patched lifecycle tests cover disconnect timing, refusal, rollback failure and subsequent work without changing delivery retry policy.

Attach the connection-error listener immediately, record failure, reject unsuccessful work and discard a broken connection. Clean up the listener and release the client on every exit.

Regression test for the corrected behavior:

Source: `apps/worker/src/jobs/enrolment-connection.test.ts:18-42` (corrected implementation).

```typescript
}
for (const moment of ['eligibility', 'smtp', 'before-commit'] as const) {
  test(`N-04: native backend loss at ${moment} is controlled and the pool remains usable`, async () => {
    queue();
    const admin = await superClient(database);
    let client: pg.PoolClient;
    let sends = 0;
    let killed = false;
    let installedListeners = -1;
    const terminate = async () => {
      if (killed) return;
      killed = true;
      installedListeners = client.listenerCount('error');
      assert.equal(installedListeners, 1, 'a checked-out client has one error listener (the pool idle listener is removed after acquire)');
      const lost = new Promise<void>(resolve => client.once('error', () => resolve()));
      const pid = (client as pg.PoolClient & { processID: number }).processID;
      assert.equal((await admin.query<{ killed: boolean }>('select pg_terminate_backend($1) as killed', [pid])).rows[0]!.killed, true);
      await lost;
    };
    const ctx = testContext(database, 1, { email: {
      async send() {
        sends++;
        if (moment === 'smtp') await terminate();
        return { providerMessageId: '<native-loss@local>', accepted: ['[redacted email]'], rejected: [] };
      }, close() {},
```

## Proof of concept

The excerpts below test the corrected behavior. They support the remediation within the tested conditions, rather than independently demonstrating the original vulnerability.

Setup: The checked-out enrolment connection fails while SMTP handoff is held.

Recorded result: Native PostgreSQL backend termination reproduced the worker failure. Patched lifecycle tests cover disconnect timing, refusal, rollback failure and subsequent work without changing delivery retry policy.

Scope: A native PostgreSQL connection was terminated under controlled timing with synthetic SMTP. Worker availability is established for that failure mode; provider delivery and ordinary incident frequency are unverified. SMTP acknowledgement ambiguity remains a separate accepted behavior.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Regression test: `apps/worker/src/jobs/enrolment-connection.test.ts`.

Source: `apps/worker/src/jobs/enrolment-connection.test.ts:18-42` (corrected implementation).

```typescript
}
for (const moment of ['eligibility', 'smtp', 'before-commit'] as const) {
  test(`N-04: native backend loss at ${moment} is controlled and the pool remains usable`, async () => {
    queue();
    const admin = await superClient(database);
    let client: pg.PoolClient;
    let sends = 0;
    let killed = false;
    let installedListeners = -1;
    const terminate = async () => {
      if (killed) return;
      killed = true;
      installedListeners = client.listenerCount('error');
      assert.equal(installedListeners, 1, 'a checked-out client has one error listener (the pool idle listener is removed after acquire)');
      const lost = new Promise<void>(resolve => client.once('error', () => resolve()));
      const pid = (client as pg.PoolClient & { processID: number }).processID;
      assert.equal((await admin.query<{ killed: boolean }>('select pg_terminate_backend($1) as killed', [pid])).rows[0]!.killed, true);
      await lost;
    };
    const ctx = testContext(database, 1, { email: {
      async send() {
        sends++;
        if (moment === 'smtp') await terminate();
        return { providerMessageId: '<native-loss@local>', accepted: ['[redacted email]'], rejected: [] };
      }, close() {},
```

Source: `apps/worker/src/jobs/enrolment-connection.test.ts:58-70` (corrected implementation).

```typescript
    try {
      await assert.rejects(sendEnrolmentCodes(ctx), /terminating connection|not queryable|connection.*closed|Connection terminated/i);
      assert.equal(sends, moment === 'eligibility' ? 0 : 1);
      assert.equal(client!.listeners('error').some(listener => listener.name === 'onLost'), false, 'the job removed its own listener');
      assert.equal(ctx.pool.totalCount, 0, 'the broken client was discarded, not returned to the pool');
      assert.equal(sql('select count(*) from aos_ctx.enrolment_letters where code is not null'), '1', 'no false successful handoff commit');
      assert.equal((await ctx.pool.query('select 1 as ok')).rows[0].ok, 1, 'subsequent worker DB work continues');
      // Clear the ambiguous test letter; this test does not add an automatic SMTP retry policy.
      sql('delete from aos_ctx.enrolment_letters');
      queue();
      const recovered = await sendEnrolmentCodes(ctx);
      assert.equal(recovered.processed, 1);
    } finally {
```
