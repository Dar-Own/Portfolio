# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low (provisional)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Forged reply headers attach mail to another connected account's thread

## Description

Forged reply headers can file inbound mail from one connected account into another account's thread within the same tenant and mailbox box, changing its status and participants.

Prerequisite and affected workflow: An outside sender references a known Message-ID from account A in mail delivered to account B's shared mailbox box.

Established observation: A forged reply header filed mail delivered to connected account B into an existing account A thread in the same company and mailbox box. The filing changed the selected thread's participants and status despite the account mismatch.

Limits: Ordinary live-mail delivery of this forged-header sequence was not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

An outside sender references a known Message-ID from account A in mail delivered to account B's shared mailbox box.

Source: `packages/mail-manager/src/threading.ts:74-89` (1d5d98db9acf).

```typescript
  if (message.threadId) {
    const thread = await store.threadByProviderThreadId(account, box, message.threadId);
    if (thread) return { rule: 'provider', thread };
  }
  const ids = headerIds(message);
  if (ids.length > 0) {
    const thread = await store.threadByMessageIds(box, ids);
    if (thread) return { rule: 'headers', thread };
  }
  const key = normalizeSubject(message.subject);
  if (key !== '') {
    const at = Date.parse(message.receivedAt);
    const since = new Date((Number.isNaN(at) ? Date.now() : at) - SUBJECT_MATCH_DAYS * 86_400_000);
    const outside = addressesOf(message, [account, ...(options.ownAddresses ?? [])]);
    const thread = outside.length > 0 ? await store.threadBySubjectAndParticipant(account, box, key, outside, since) : null;
    if (thread) return { rule: 'subject', thread };
```

#### Step 2 — Cross the missing state or authority check

The provider-thread-ID and subject/participant branches bind the lookup to `account`, but the reply-header branch calls `threadByMessageIds(box, ids)` without it: `packages/mail-manager/src/threading.ts:74-89`. The SQL implementation checks tenant and box but not `t.account`: `apps/api/src/features/inbox/sql-store.ts:314-345`. `fileMessage` then uses the selected existing thread, stores the new-account message under its ID, and updates status and participants: `packages/mail-manager/src/filing.ts:164-185,196-271`.

Source: `apps/api/src/features/inbox/sql-store.ts:314-345` (1d5d98db9acf).

```typescript
    // The three threading lookups are of one box: the box is the read rule (an estimator reads bids@ only),
    // and a message joins only a thread of the box its own recipients chose.
    async threadByProviderThreadId(account, box, providerThreadId) {
      const { rows } = await client.query<ThreadRefRow>(
        `${THREAD_REF_SELECT} where t.tenant_id = $1 and t.account = $2 and t.mailbox = $3::app.mailbox and t.provider_thread_id = $4 order by t.last_message_at desc limit 1`,
        [tenantId, account, box, providerThreadId],
      );
      return rows[0] ? remember(rows[0]) : null;
    },

    async threadByMessageIds(box, messageIdHeaders) {
      if (messageIdHeaders.length === 0) return null;
      const { rows } = await client.query<ThreadRefRow>(
        `select t.id, t.account, t.mailbox::text as box, t.status::text as status, t.category, t.category_set_by, t.assigned_to, t.message_count
           from app.email_messages em
           join app.mail_threads t on t.tenant_id = em.tenant_id and t.id = em.thread_id
          where em.tenant_id = $1 and t.mailbox = $2::app.mailbox and em.message_id_header = any($3::text[])
          order by array_position($3::text[], em.message_id_header), em.received_at
          limit 1`,
        [tenantId, box, [...messageIdHeaders]],
      );
      return rows[0] ? remember(rows[0]) : null;
    },

    async threadBySubjectAndParticipant(account, box, subjectKey, addresses, since) {
      const wanted = addresses.map((address) => address.trim().toLowerCase()).filter((address) => address !== '');
      if (wanted.length === 0 || subjectKey === '') return null;
      const { rows } = await client.query<ThreadRefRow>(
        `${THREAD_REF_SELECT}
          where t.tenant_id = $1 and t.account = $2 and t.mailbox = $3::app.mailbox and t.subject_key = $4 and t.last_message_at >= $5 and t.participants && $6::citext[]
          order by t.last_message_at desc limit 1`,
        [tenantId, account, box, subjectKey, since, wanted],
```

Source: `packages/mail-manager/src/filing.ts:164-185` (1d5d98db9acf).

```typescript
export async function fileMessage(store: MailStore, org: Organization, account: string, handedOver: MailboxMessage, options: FilingOptions = {}): Promise<Filed> {
  const mailAccount = accountOf(org, account);
  // Cleaned again here, where the rows are decided: no caller can put a NUL byte in the database by mistake.
  const message = cleanMailboxMessage(handedOver);
  const existing = await store.messageByProviderId(account, message.id);
  if (existing) return { outcome: 'already_filed', message: existing };

  const recipients = [...message.to, ...message.cc];
  // The box is the read rule: the message's own addresses choose it, and it joins only a thread of that box
  // (threading.ts); it never takes the box of a thread it would have joined. The recipients count, and for a
  // letter from the company's own Sent folder its From too: only the company writes its Sent folder, and a
  // reply sent from Gmail as bids@ leaves bids@ out of its recipients (Reply all drops the account's own
  // aliases). An incoming email's From never counts: anybody can write "From: bids@".
  const box = boxFor(mailAccount, message.folder === 'sent' ? [message.fromAddress, ...recipients] : recipients);
  const threading = await findThreadFor(store, account, box, message, { ownAddresses: ownAddresses(org) });
  if (threading.rule === 'skip') return { outcome: 'skipped', reason: 'a letter that is never threaded' };
  if (threading.rule === 'app_header') {
    await store.noteSentCopy(threading.message, { providerMessageId: message.id, providerThreadId: message.threadId, messageIdHeader: message.messageId });
    return { outcome: 'sent_copy', message: threading.message, thread: threading.thread };
  }

  const direction: Direction = message.folder === 'sent' ? 'outbound' : 'inbound';
```


#### Step 3 — Observe the durable effect

A forged reply header filed mail delivered to connected account B into an existing account A thread in the same company and mailbox box. The filing changed the selected thread's participants and status despite the account mismatch.

Ordinary live-mail delivery of this forged-header sequence was not tested.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/pocs/run-lead-011-worker.mjs:265-286` (synthetic test).

```javascript
    stateOf(backfill.oldThreadId).ai_next_pass], ['ai', true, null]);
  controls.backfill = 'thread joined; no unattended draft or reply';

  psql(`alter table app.mail_settings disable trigger mail_settings_guard;
    update app.mail_settings set ai_send_categories = '{}' where tenant_id = '${tenant}';
    alter table app.mail_settings enable trigger mail_settings_guard;`);
  const off = await probe('off');
  mustJoin('auto-send off', off);
  const sameAccount = await probe('same-account', { seedAccount: newAccount });
  mustJoin('same-account positive control', sameAccount);
  const noHeader = await probe('no-header', { header: false });
  assert.equal(noHeader.filed.threading, 'new');
  assert.notEqual(noHeader.filed.thread.id, noHeader.oldThreadId);
  const wrongBox = await probe('wrong-box', { box: 'bids' });
  assert.equal(wrongBox.filed.threading, 'new');
  assert.notEqual(wrongBox.filed.thread.id, wrongBox.oldThreadId);
  draftsBefore = unattendedCount();
  await runTwoPasses();
  assert.equal(unattendedCount(), draftsBefore, 'auto-send off must prevent all unattended drafts');
  for (const result of [off, sameAccount, noHeader, wrongBox]) assert.equal(queuedTo(result.recipient), 0, result.name);
  assert.equal(stateOf(off.oldThreadId).category_set_by, 'ai');
  assert.equal(Number(psql(`select cardinality(ai_send_categories) from app.mail_settings where tenant_id = '${tenant}'`)), 0);
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: An outside sender references a known Message-ID from account A in mail delivered to account B's shared mailbox box.

Recorded result: A forged reply header filed mail delivered to connected account B into an existing account A thread in the same company and mailbox box. The filing changed the selected thread's participants and status despite the account mismatch.

Scope: Ordinary live-mail delivery of this forged-header sequence was not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/pocs/run-lead-011-worker.mjs:265-286` (synthetic test).

```javascript
    stateOf(backfill.oldThreadId).ai_next_pass], ['ai', true, null]);
  controls.backfill = 'thread joined; no unattended draft or reply';

  psql(`alter table app.mail_settings disable trigger mail_settings_guard;
    update app.mail_settings set ai_send_categories = '{}' where tenant_id = '${tenant}';
    alter table app.mail_settings enable trigger mail_settings_guard;`);
  const off = await probe('off');
  mustJoin('auto-send off', off);
  const sameAccount = await probe('same-account', { seedAccount: newAccount });
  mustJoin('same-account positive control', sameAccount);
  const noHeader = await probe('no-header', { header: false });
  assert.equal(noHeader.filed.threading, 'new');
  assert.notEqual(noHeader.filed.thread.id, noHeader.oldThreadId);
  const wrongBox = await probe('wrong-box', { box: 'bids' });
  assert.equal(wrongBox.filed.threading, 'new');
  assert.notEqual(wrongBox.filed.thread.id, wrongBox.oldThreadId);
  draftsBefore = unattendedCount();
  await runTwoPasses();
  assert.equal(unattendedCount(), draftsBefore, 'auto-send off must prevent all unattended drafts');
  for (const result of [off, sameAccount, noHeader, wrongBox]) assert.equal(queuedTo(result.recipient), 0, result.name);
  assert.equal(stateOf(off.oldThreadId).category_set_by, 'ai');
  assert.equal(Number(psql(`select cardinality(ai_send_categories) from app.mail_settings where tenant_id = '${tenant}'`)), 0);
```

## Remediation

Bind reply-header thread lookup to the incoming connected account and require explicit authority for any account migration.
