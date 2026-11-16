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

# Post-acceptance rollback can repeat outbound mail

## Description

An outbound email accepted before a worker rollback can be submitted again on retry.

Prerequisite and affected workflow: SMTP accepts a queued message and the worker transaction fails before its sent marker commits.

Established observation: After the test mail adapter accepted a message, a forced database rollback left the row queued; retry handed the same message to the adapter again. This demonstrates duplicate application handoff, without asserting duplicate delivery by a real mail provider.

Limits: Duplicate delivery by a real SMTP provider was not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

SMTP accepts a queued message and the worker transaction fails before its sent marker commits.

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

#### Step 2 — Cross the missing state or authority check

`outbox-delivery` calls `sender.send` before writing `app.messages.status = 'sent'` (`apps/worker/src/jobs/outbox-delivery.ts:665-681,729-747`). `withAutomation` rolls back the transaction on an error (`apps/worker/src/db.ts:45-69`), so provider acceptance cannot be rolled back with it. The next `pickQueued` can select the row again (`apps/worker/src/jobs/outbox-delivery.ts:395-411`). The source explicitly acknowledges at-least-once behavior (`apps/worker/src/jobs/outbox-delivery.ts:73-76`); no provider idempotency guarantee was established for the repeated request.

Source: `apps/worker/src/db.ts:45-69` (1d5d98db9acf).

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
  } finally {
    client.release(broken);
  }
```

Source: `apps/worker/src/jobs/outbox-delivery.ts:395-411` (1d5d98db9acf).

```typescript
async function pickQueued(client: pg.PoolClient, tenantId: string, channel: 'email' | 'sms', tried: string[]): Promise<PickedMessage | null> {
  const { rows } = await client.query<PickedMessage>(
    `select m.id, m.lead_id, m.customer_id, m.to_address, m.from_address, m.template_key, m.body, m.blocked_reason, m.recipient_user_id,
            m.subject, m.in_reply_to, m.references_header
       from app.messages m
      where m.tenant_id = $1 and m.channel = $2::app.channel and m.direction = 'outbound' and m.status = 'queued'
        and m.id <> all($3::uuid[])
        and (m.send_after is null or m.send_after <= now())
        and (m.channel <> 'email' or m.template_key is null or m.template_key <> all($4::text[])
             or m.created_at < now() - make_interval(secs => $5)
             or exists (select 1 from app.message_attachments a where a.tenant_id = m.tenant_id and a.message_id = m.id))
      order by m.created_at, m.id
      limit 1
      for update of m skip locked`,
    [tenantId, channel, tried, [...PDF_TEMPLATE_KEYS], ATTACHMENT_GRACE_SECONDS],
  );
  return rows[0] ?? null;
```


#### Step 3 — Observe the durable effect

After the test mail adapter accepted a message, a forced database rollback left the row queued; retry handed the same message to the adapter again. This demonstrates duplicate application handoff, without asserting duplicate delivery by a real mail provider.

Duplicate delivery by a real SMTP provider was not tested.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/pocs/run-leads-013-015.mjs:205-243` (synthetic test).

```javascript
  // POC-014: one successful control; then a provider acceptance followed by an actual SQL
  // failure in the still-open transaction, rollback, and a second acceptance on retry.
  const outboxControl = id(1401), outboxFault = id(1402);
  psql(`insert into app.messages (id,tenant_id,channel,direction,to_address,from_address,body,status)
    values ('${outboxControl}','${tenant}','email','outbound','[redacted email]','[redacted email]','Synthetic control','queued'),
           ('${outboxFault}','${tenant}','email','outbound','[redacted email]','[redacted email]','Synthetic fault','queued');`);
  const fakeSmtp014 = [];
  async function sendAttempt(messageId, fail) {
    const session = new SqlSession('poc_worker');
    try {
      await session.query(`begin; select app.act_as_automation('${tenant}');`);
      const picked = await session.query(`select id from app.messages where tenant_id='${tenant}' and id='${messageId}'
        and channel='email' and direction='outbound' and status='queued' for update skip locked;`);
      assert.equal(picked, messageId);
      // Fake provider accepts before the app attempts to persist local sent state.
      fakeSmtp014.push({ outbox: messageId, providerMessageId: `fake-${fakeSmtp014.length + 1}` });
      if (fail) {
        await session.query('select 1/0;');
        assert.match(session.error, /division by zero/);
        await session.query('rollback;');
        return 'rolled_back';
      }
      const saved = await session.query(`update app.messages set status='sent', sent_at=now(),
        provider_message_id='${fakeSmtp014.at(-1).providerMessageId}', blocked_reason=null
        where tenant_id='${tenant}' and id='${messageId}' and status='queued' returning status;`);
      assert.equal(saved, 'sent');
      await session.query('commit;');
      return saved;
    } finally { await session.close(); }
  }
  assert.equal(await sendAttempt(outboxControl, false), 'sent');
  assert.equal(await sendAttempt(outboxFault, true), 'rolled_back');
  const afterFault = last(`select status from app.messages where id='${outboxFault}';`);
  assert.equal(afterFault, 'queued');
  assert.equal(await sendAttempt(outboxFault, false), 'sent');
  const afterRetry = last(`select status from app.messages where id='${outboxFault}';`);
  assert.equal(afterRetry, 'sent');
  assert.equal(fakeSmtp014.filter((x) => x.outbox === outboxControl).length, 1);
  assert.equal(fakeSmtp014.filter((x) => x.outbox === outboxFault).length, 2);
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: SMTP accepts a queued message and the worker transaction fails before its sent marker commits.

Recorded result: After the test mail adapter accepted a message, a forced database rollback left the row queued; retry handed the same message to the adapter again. This demonstrates duplicate application handoff, without asserting duplicate delivery by a real mail provider.

Scope: Duplicate delivery by a real SMTP provider was not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/pocs/run-leads-013-015.mjs:205-243` (synthetic test).

```javascript
  // POC-014: one successful control; then a provider acceptance followed by an actual SQL
  // failure in the still-open transaction, rollback, and a second acceptance on retry.
  const outboxControl = id(1401), outboxFault = id(1402);
  psql(`insert into app.messages (id,tenant_id,channel,direction,to_address,from_address,body,status)
    values ('${outboxControl}','${tenant}','email','outbound','[redacted email]','[redacted email]','Synthetic control','queued'),
           ('${outboxFault}','${tenant}','email','outbound','[redacted email]','[redacted email]','Synthetic fault','queued');`);
  const fakeSmtp014 = [];
  async function sendAttempt(messageId, fail) {
    const session = new SqlSession('poc_worker');
    try {
      await session.query(`begin; select app.act_as_automation('${tenant}');`);
      const picked = await session.query(`select id from app.messages where tenant_id='${tenant}' and id='${messageId}'
        and channel='email' and direction='outbound' and status='queued' for update skip locked;`);
      assert.equal(picked, messageId);
      // Fake provider accepts before the app attempts to persist local sent state.
      fakeSmtp014.push({ outbox: messageId, providerMessageId: `fake-${fakeSmtp014.length + 1}` });
      if (fail) {
        await session.query('select 1/0;');
        assert.match(session.error, /division by zero/);
        await session.query('rollback;');
        return 'rolled_back';
      }
      const saved = await session.query(`update app.messages set status='sent', sent_at=now(),
        provider_message_id='${fakeSmtp014.at(-1).providerMessageId}', blocked_reason=null
        where tenant_id='${tenant}' and id='${messageId}' and status='queued' returning status;`);
      assert.equal(saved, 'sent');
      await session.query('commit;');
      return saved;
    } finally { await session.close(); }
  }
  assert.equal(await sendAttempt(outboxControl, false), 'sent');
  assert.equal(await sendAttempt(outboxFault, true), 'rolled_back');
  const afterFault = last(`select status from app.messages where id='${outboxFault}';`);
  assert.equal(afterFault, 'queued');
  assert.equal(await sendAttempt(outboxFault, false), 'sent');
  const afterRetry = last(`select status from app.messages where id='${outboxFault}';`);
  assert.equal(afterRetry, 'sent');
  assert.equal(fakeSmtp014.filter((x) => x.outbox === outboxControl).length, 1);
  assert.equal(fakeSmtp014.filter((x) => x.outbox === outboxFault).length, 2);
```

## Remediation

Use a stable provider idempotency key where supported or reconcile uncertain sends by outbox ID before retry.
