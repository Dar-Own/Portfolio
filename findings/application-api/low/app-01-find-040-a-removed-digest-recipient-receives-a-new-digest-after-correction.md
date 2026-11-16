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

# A removed digest recipient receives a new digest after correction

## Description

A removed digest recipient can receive a newly saved in-app digest and alert after the owner corrects the recipient list.

Prerequisite and affected workflow: The worker selects recipient A; the owner removes A before the digest and alert are saved.

Established observation: The owner removed A and selected B before the digest was saved. The worker still saved the new digest and alert for A, and A could read it through the application; changing the list before the worker gathered recipients avoided the effect.

Limits: Natural timing and any external delivery remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

The worker selects recipient A; the owner removes A before the digest and alert are saved.

Source: `apps/worker/src/jobs/mail-digest.ts:120-145` (1d5d98db9acf).

```typescript
  const { rows: settings } = await client.query<{ digest_hour: string; digest_user_ids: string[] | null }>(
    'select digest_hour::text as digest_hour, digest_user_ids from app.mail_settings where tenant_id = $1',
    [tenantId],
  );
  if (localHour(now, hours.timezone) < (settings[0] ? Number(settings[0].digest_hour) : DEFAULT_DIGEST_HOUR)) return null;
  if ((await client.query('select 1 from app.mail_digests where tenant_id = $1 and day = $2', [tenantId, day])).rows.length > 0) return null;
  // The people named who work the shared inbox (who may open office@, exceptions counted: the conversations
  // it names are theirs to read), else the owner.
  const { rows: recipients } = await client.query<{ user_id: string }>(
    `select p.user_id from app.mail_box_people('office') p
      where case when $1::uuid[] is null then p.is_owner else p.user_id = any($1::uuid[]) end
      order by p.full_name, p.user_id limit $2`,
    [settings[0]?.digest_user_ids ?? null, MAX_DIGEST_RECIPIENTS],
  );
  if (recipients.length === 0) return null;

  const org = await buildOrganization(client, tenantId, { address: await simulatedAccount(client, tenantId), provider: 'fake' });
  const from = dayStart(previousBusinessDay(day, days), hours.timezone);
  const to = dayStart(day, hours.timezone);
  const data = await digestData(client, tenantId, org, now, from, to);
  // The daily cap counts the company's day, as mail-ai counts it.
  const today = await companyDay(client, tenantId, now);
  const spent = await client.query<{ cents: string }>('select cents from app.mail_ai_spend($1, $2)', [today.start, today.next]);
  return {
    day,
    recipients: recipients.map((row) => row.user_id),
```

#### Step 2 — Cross the missing state or authority check

The daily digest worker saves the digest and alerts recipients from an older recipient list without checking the current list.

Source: `apps/worker/src/jobs/mail-digest.ts:238-270` (1d5d98db9acf).

```typescript
// 3. Store it, once a day, and tell its recipients

const RUN_WORDS: Readonly<Record<'ok' | 'failed' | 'refused', string>> = Object.freeze({
  ok: 'wrote the morning digest',
  failed: 'failed to write the morning digest',
  refused: 'declined to write the morning digest',
});

async function store(client: Client, tenantId: string, due: Due, answer: Written, now: Date): Promise<boolean> {
  let runId: string | null = null;
  if (answer.run) {
    runId = randomUUID();
    // The digest is about no one conversation: its line is read through app.mail_ai_spend alone.
    await client.query(
      `insert into app.mail_ai_runs (id, tenant_id, thread_id, message_id, kind, model, input_tokens, output_tokens, cache_read_tokens, cache_write_tokens,
                                     estimated_cents, status, outcome, fields_sent, attachments_sent, requested_by, created_at)
       values ($1, $2, null, null, 'digest', $3, $4, $5, $6, $7, $8, $9, $10, $11::text[], false, null, $12)`,
      [
        runId, tenantId, answer.run.model, answer.run.usage.input, answer.run.usage.output, answer.run.usage.cacheRead, answer.run.usage.cacheWrite,
        answer.run.cents.toFixed(4), answer.run.status, RUN_WORDS[answer.run.status], [...DIGEST_VIEW_FIELDS], now,
      ],
    );
  }
  const inserted = await client.query(
    `insert into app.mail_digests (tenant_id, day, body, note, data, recipient_ids, ai_run_id) values ($1, $2, $3, $4, $5::jsonb, $6::uuid[], $7)
     on conflict (tenant_id, day) do nothing`,
    [tenantId, due.day, answer.body, answer.note, JSON.stringify(due.data), due.recipients, runId],
  );
  if ((inserted.rowCount ?? 0) !== 1) return false;
  await postPing(client, tenantId, MAIL_DIGEST_CHANNEL, digestAlertText(digestDayLabel(due.day), due.data), {
    forUsers: due.recipients,
    path: DIGEST_PATH,
    textSummary: digestTextSummary(due.data),
```


#### Step 3 — Observe the durable effect

The owner removed A and selected B before the digest was saved. The worker still saved the new digest and alert for A, and A could read it through the application; changing the list before the worker gathered recipients avoided the effect.

Natural timing and any external delivery remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/worker-authority-tail/run-digest-recipient-switch.mjs:272-292` (synthetic test).

```javascript
  const revokedRead = await digest(maya.headers, '2026-09-22');
  const revokedPings = await app.inject({ method: 'GET', url: '/api/tasks/pings?limit=50', headers: maya.headers });
  const revokedStored = row(`select recipient_ids::text[] as ids, data->'needsReply' as lines from app.mail_digests where tenant_id = '${peach}' and day = '2026-09-22'`);
  const revokedAlert = row(`select recipient_ids::text[] as ids, body, text_summary from app.notifications where tenant_id = '${peach}' and link like '%/inbox/digest' order by created_at desc, id desc limit 1`);
  record.revocation = { happensBefore: ['gather selects A while inbox.work allowed', 'fake AI held',
      'owner password-confirmed HTTP deny inbox.work for A commits', 'fake AI completes', 'J13 stores and alerts', 'A attempts R185'],
    beforeRevocation, revokeStatus: revoked.statusCode, job: revokedJob, stored: revokedStored,
    AReadStatus: revokedRead.statusCode, AReadContainsMarker: revokedRead.body.includes(marker),
    APingsStatus: revokedPings.statusCode,
    APingsDigestItems: revokedPings.statusCode === 200 ? revokedPings.json().items.filter((item) => item.channel === '#inbox' && item.body.includes('Email digest:')) : [],
    alert: revokedAlert, alertContainsMarker: JSON.stringify(revokedAlert).includes(marker),
    foreignMarkerAbsent: !(await digest(foreign.headers, '2026-09-22')).body.includes(marker) };
  assert.equal(revokedJob.errors, 0, JSON.stringify(revokedJob));
  assert.deepEqual(revokedStored.ids, [mayaId], 'expected stale recipient id for the revocation challenge');
  assert.ok([401, 403, 404].includes(revokedRead.statusCode), 'revoked A unexpectedly read digest');
  assert.equal(revokedPings.statusCode, 200, revokedPings.body);
  assert.ok(record.revocation.APingsDigestItems.length > 0, 'revoked A did not see the counts-only digest alert');
  assert.ok(!revokedPings.body.includes(marker), 'A alert list unexpectedly carried the private subject');
  assert.equal(record.revocation.alertContainsMarker, false, 'alert unexpectedly carried the private subject');
  assert.equal(record.revocation.foreignMarkerAbsent, true, 'foreign tenant read Peach subject');
  record.state = 'PROVED-RECIPIENT-SWITCH; PROVED-HIDDEN-RECORD-NUMBER; REVOKED-INBOX-DETAIL-FALSIFIED-IN-SCOPE';
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: The worker selects recipient A; the owner removes A before the digest and alert are saved.

Recorded result: The owner removed A and selected B before the digest was saved. The worker still saved the new digest and alert for A, and A could read it through the application; changing the list before the worker gathered recipients avoided the effect.

Scope: Natural timing and any external delivery remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/worker-authority-tail/run-digest-recipient-switch.mjs:272-292` (synthetic test).

```javascript
  const revokedRead = await digest(maya.headers, '2026-09-22');
  const revokedPings = await app.inject({ method: 'GET', url: '/api/tasks/pings?limit=50', headers: maya.headers });
  const revokedStored = row(`select recipient_ids::text[] as ids, data->'needsReply' as lines from app.mail_digests where tenant_id = '${peach}' and day = '2026-09-22'`);
  const revokedAlert = row(`select recipient_ids::text[] as ids, body, text_summary from app.notifications where tenant_id = '${peach}' and link like '%/inbox/digest' order by created_at desc, id desc limit 1`);
  record.revocation = { happensBefore: ['gather selects A while inbox.work allowed', 'fake AI held',
      'owner password-confirmed HTTP deny inbox.work for A commits', 'fake AI completes', 'J13 stores and alerts', 'A attempts R185'],
    beforeRevocation, revokeStatus: revoked.statusCode, job: revokedJob, stored: revokedStored,
    AReadStatus: revokedRead.statusCode, AReadContainsMarker: revokedRead.body.includes(marker),
    APingsStatus: revokedPings.statusCode,
    APingsDigestItems: revokedPings.statusCode === 200 ? revokedPings.json().items.filter((item) => item.channel === '#inbox' && item.body.includes('Email digest:')) : [],
    alert: revokedAlert, alertContainsMarker: JSON.stringify(revokedAlert).includes(marker),
    foreignMarkerAbsent: !(await digest(foreign.headers, '2026-09-22')).body.includes(marker) };
  assert.equal(revokedJob.errors, 0, JSON.stringify(revokedJob));
  assert.deepEqual(revokedStored.ids, [mayaId], 'expected stale recipient id for the revocation challenge');
  assert.ok([401, 403, 404].includes(revokedRead.statusCode), 'revoked A unexpectedly read digest');
  assert.equal(revokedPings.statusCode, 200, revokedPings.body);
  assert.ok(record.revocation.APingsDigestItems.length > 0, 'revoked A did not see the counts-only digest alert');
  assert.ok(!revokedPings.body.includes(marker), 'A alert list unexpectedly carried the private subject');
  assert.equal(record.revocation.alertContainsMarker, false, 'alert unexpectedly carried the private subject');
  assert.equal(record.revocation.foreignMarkerAbsent, true, 'foreign tenant read Peach subject');
  record.state = 'PROVED-RECIPIENT-SWITCH; PROVED-HIDDEN-RECORD-NUMBER; REVOKED-INBOX-DETAIL-FALSIFIED-IN-SCOPE';
```

## Remediation

Recheck or version the recipient setting at digest save and alert creation, then save only currently authorized recipients.
