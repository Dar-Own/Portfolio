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

# Mail nudge creates an open old-message task after staff completes, snoozes or replies

## Description

Staff can receive an open old-message task and subject-bearing ping after completing, snoozing, or replying to the thread.

Prerequisite and affected workflow: The worker selects a thread, then staff complete, snooze, or answer it before the task is saved.

Established observation: After the worker selected an inbox thread, staff marked it Done, snoozed it, or sent a reply. Each action committed before the worker wrote its effect, yet it created an open old-message task and a subject-bearing staff alert; changing state before selection suppressed the nudge.

Limits: Natural overlap and any customer-facing message effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

The worker selects a thread, then staff complete, snooze, or answer it before the task is saved.

Source: `apps/worker/src/jobs/mail-nudges.ts:63-136` (1d5d98db9acf).

```typescript
  const woken = await client.query(`update app.mail_threads set status = 'needs_reply' where tenant_id = $1 and status = 'snoozed' and snoozed_until <= $2`, [tenantId, now]);
  tally.woken += woken.rowCount ?? 0;

  const setting = await client.query<{ nudge_minutes: number }>('select nudge_minutes from app.mail_settings where tenant_id = $1', [tenantId]);
  const minutes = setting.rows[0]?.nudge_minutes ?? DEFAULT_NUDGE_MINUTES;
  if (minutes <= 0) return;
  // The wall-clock test first (business minutes never exceed wall-clock ones), then the company's own hours.
  const { rows } = await client.query<Waiting>(
    `select t.id, t.subject, t.mailbox::text as mailbox, t.assigned_to, t.lead_id, t.last_inbound_at
       from app.mail_threads t
      where t.tenant_id = $1 and t.status = 'needs_reply' and t.last_inbound_at is not null
        and t.nudged_for is distinct from t.last_inbound_at
        and t.last_inbound_at <= $2::timestamptz - make_interval(mins => $3)
        and app.business_minutes($1, t.last_inbound_at, $2::timestamptz) >= $3
      order by t.last_inbound_at, t.id
      limit $4`,
    [tenantId, now, minutes, MAX_PER_RUN],
  );
  if (rows.length === 0) return;
  const settings = await loadAutomationSettings(client, tenantId);
  const openers = new Map<Mailbox, Opener[]>();
  for (const thread of rows) {
    if (!openers.has(thread.mailbox)) openers.set(thread.mailbox, await boxOpeners(client, thread.mailbox));
    const owner = ownerFor(openers.get(thread.mailbox)!, [thread.assigned_to, settings.routing.office]);
    const done = await inSavepoint(client, () => nudge(client, tenantId, thread, minutes, owner, now));
    if (done.ok) {
      if (done.value) tally.nudged++;
    } else {
      tally.errors++;
      ctx.logger.error(`mail-nudges: conversation ${thread.id}: ${errorText(done.error)}`);
    }
  }
}

export interface Opener extends TaskOwner {
  isOwner: boolean;
}

/** Who may open a box, by its own read rule with every exception, the company's owners among them. */
export async function boxOpeners(client: Client, box: Mailbox): Promise<Opener[]> {
  const { rows } = await client.query<{ user_id: string; full_name: string; is_owner: boolean }>('select user_id, full_name, is_owner from app.mail_box_people($1)', [box]);
  return rows.map((row) => ({ id: row.user_id, fullName: row.full_name, isOwner: row.is_owner }));
}

/** The first of the people named who may open the box, else the company's owner (the first, by rank and name). */
export function ownerFor(openers: readonly Opener[], named: readonly (string | null | undefined)[]): TaskOwner | null {
  for (const id of named) {
    const person = id ? openers.find((opener) => opener.id === id) : undefined;
    if (person) return person;
  }
  return openers.find((opener) => opener.isOwner) ?? null;
}

/** The task and the alert (once per incoming email), then the thread's mark. True when this run nudged. */
async function nudge(client: Client, tenantId: string, thread: Waiting, minutes: number, owner: TaskOwner | null, now: Date): Promise<boolean> {
  if (!owner) throw new Error('no one who may open this box to nudge');
  const created = await createAutomationTask(client, {
    tenantId,
    title: nudgeTaskTitle(thread.subject),
    kind: 'inbox',
    ownerId: owner.id,
    leadId: thread.lead_id,
    dueAt: now,
    dedupeKey: nudgeDedupeKey(thread.id, thread.last_inbound_at),
  });
  if (created) {
    await postPing(client, tenantId, MAIL_NUDGE_CHANNEL, nudgeAlertText({ subject: thread.subject, box: thread.mailbox, minutes, ownerName: owner.fullName }), {
      forUsers: [owner.id],
      path: `/inbox/t/${thread.id}`,
      textSummary: nudgeTextSummary({ box: thread.mailbox, minutes, ownerName: owner.fullName }),
    });
  }
  await client.query('update app.mail_threads set nudged_for = $3 where tenant_id = $1 and id = $2', [tenantId, thread.id, thread.last_inbound_at]);
  return created;
```

#### Step 2 — Cross the missing state or authority check

The inbox nudge worker creates a task from an older thread snapshot without checking its current status or latest message.

Source: `apps/api/src/features/inbox/routes.ts:343-367` (1d5d98db9acf).

```typescript
    app.patch('/threads/:id', { schema: { params: threadParams, body: threadPatch } }, async (request): Promise<ThreadView> => {
      requireInboxAccess(request.session);
      return app.db.withUser(request.session, async (ctx) => updateThread(ctx, request.session, request.params.id, request.body, await loadCompanySetup(ctx)));
    });

    app.post('/threads/:id/actions', { schema: { params: threadParams, body: actionBody } }, async (request): Promise<ThreadActionResult> => {
      requireInboxAccess(request.session);
      return app.db.withUser(request.session, async (ctx) => actOnThread(ctx, request.session, request.params.id, request.body, await loadCompanySetup(ctx)));
    });

    app.post('/threads/:id/notes', { schema: { params: threadParams, body: noteBody } }, async (request, reply) => {
      requireInboxAccess(request.session);
      requirePermission(request.session, 'comments.write');
      const comment = await app.db.withUser(request.session, (ctx) => addThreadNote(ctx, request.params.id, request.body.body));
      reply.status(201);
      return { comment };
    });

    app.post('/threads/:id/reply', { schema: { params: threadParams, body: replyBody }, config: { rateLimit: replyLimit } }, async (request, reply): Promise<ThreadReplyResult> => {
      requireInboxAccess(request.session);
      requirePermission(request.session, 'messages.send');
      const result = await app.db.withUser(request.session, async (ctx) => replyToThread(ctx, request.session, request.params.id, request.body, await loadCompanySetup(ctx)));
      reply.status(201);
      return result;
    });
```


#### Step 3 — Observe the durable effect

After the worker selected an inbox thread, staff marked it Done, snoozed it, or sent a reply. Each action committed before the worker wrote its effect, yet it created an open old-message task and a subject-bearing staff alert; changing state before selection suppressed the nudge.

Natural overlap and any customer-facing message effect remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/worker-authority-tail/run-mail-nudge-stale-selection.mjs:176-205` (synthetic test).

```javascript
  const atBarrier = Object.fromEntries(['resolved', 'snoozed', 'replied', 'unchanged'].map((name) => [name, markerState(peach, threadIds[name])]));
  const resolved = await patchStatus('resolved', { status: 'done' });
  const snoozed = await patchStatus('snoozed', { status: 'snoozed', snoozedUntil: future });
  const replied = await reply('replied');
  assert.deepEqual([resolved.statusCode, snoozed.statusCode, replied.statusCode], [200, 200, 201], `${resolved.body}\n${snoozed.body}\n${replied.body}`);
  const beforeRelease = Object.fromEntries(['resolved', 'snoozed', 'replied', 'unchanged'].map((name) => [name, markerState(peach, threadIds[name])]));
  assert.deepEqual([beforeRelease.resolved.status, beforeRelease.snoozed.status, beforeRelease.replied.status, beforeRelease.unchanged.status], ['done', 'snoozed', 'waiting', 'needs_reply']);
  for (const state of Object.values(beforeRelease)) assert.equal(state.tasks + state.pings, 0, 'J14 effect happened before release');
  releaseResolve();
  const result = await jobPromise;
  assert.equal(result.errors, 0, ctx.logger.errors.join(' / '));
  const after = Object.fromEntries(cases.map((name) => [name, markerState(peach, threadIds[name])]));
  for (const name of ['resolved', 'snoozed', 'replied', 'unchanged']) {
    assert.equal(after[name].tasks, 1, `${name} missing nudge task`);
    assert.equal(after[name].open_tasks, 1, `${name} nudge task was not open`);
    assert.equal(after[name].pings, 1, `${name} missing nudge ping`);
  }
  for (const name of ['pre-resolved', 'pre-snoozed', 'pre-replied']) {
    assert.equal(after[name].tasks + after[name].pings, 0, `${name} got nudge despite changed pre-SELECT`);
  }
  assert.deepEqual([after.resolved.status, after.snoozed.status, after.replied.status], ['done', 'snoozed', 'waiting']);
  const foreignThread = await app.inject({ method: 'GET', url: `/api/inbox/threads/${threadIds.resolved}`, headers: foreign.headers });
  assert.equal(foreignThread.statusCode, 404);
  const foreignEffects = idsOf(rival);
  assert.deepEqual(foreignEffects, { tasks: 0, pings: 0 });
  const ownerPings = await app.inject({ method: 'GET', url: '/api/tasks/pings?limit=50', headers: owner.headers });
  assert.equal(ownerPings.statusCode, 200, ownerPings.body);
  const pings = ownerPings.json().items.filter((item) => item.body.includes('J14 boundary'));
  assert.ok(pings.length >= 4, 'owner-visible pings omitted a selected J14 marker');
  record.race = { happensBefore: ['J14 SELECT captured eligible thread rows', 'owner HTTP done/snooze/reply committed', 'J14 resumed createAutomationTask/postPing/nudged_for'],
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: The worker selects a thread, then staff complete, snooze, or answer it before the task is saved.

Recorded result: After the worker selected an inbox thread, staff marked it Done, snoozed it, or sent a reply. Each action committed before the worker wrote its effect, yet it created an open old-message task and a subject-bearing staff alert; changing state before selection suppressed the nudge.

Scope: Natural overlap and any customer-facing message effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/worker-authority-tail/run-mail-nudge-stale-selection.mjs:176-205` (synthetic test).

```javascript
  const atBarrier = Object.fromEntries(['resolved', 'snoozed', 'replied', 'unchanged'].map((name) => [name, markerState(peach, threadIds[name])]));
  const resolved = await patchStatus('resolved', { status: 'done' });
  const snoozed = await patchStatus('snoozed', { status: 'snoozed', snoozedUntil: future });
  const replied = await reply('replied');
  assert.deepEqual([resolved.statusCode, snoozed.statusCode, replied.statusCode], [200, 200, 201], `${resolved.body}\n${snoozed.body}\n${replied.body}`);
  const beforeRelease = Object.fromEntries(['resolved', 'snoozed', 'replied', 'unchanged'].map((name) => [name, markerState(peach, threadIds[name])]));
  assert.deepEqual([beforeRelease.resolved.status, beforeRelease.snoozed.status, beforeRelease.replied.status, beforeRelease.unchanged.status], ['done', 'snoozed', 'waiting', 'needs_reply']);
  for (const state of Object.values(beforeRelease)) assert.equal(state.tasks + state.pings, 0, 'J14 effect happened before release');
  releaseResolve();
  const result = await jobPromise;
  assert.equal(result.errors, 0, ctx.logger.errors.join(' / '));
  const after = Object.fromEntries(cases.map((name) => [name, markerState(peach, threadIds[name])]));
  for (const name of ['resolved', 'snoozed', 'replied', 'unchanged']) {
    assert.equal(after[name].tasks, 1, `${name} missing nudge task`);
    assert.equal(after[name].open_tasks, 1, `${name} nudge task was not open`);
    assert.equal(after[name].pings, 1, `${name} missing nudge ping`);
  }
  for (const name of ['pre-resolved', 'pre-snoozed', 'pre-replied']) {
    assert.equal(after[name].tasks + after[name].pings, 0, `${name} got nudge despite changed pre-SELECT`);
  }
  assert.deepEqual([after.resolved.status, after.snoozed.status, after.replied.status], ['done', 'snoozed', 'waiting']);
  const foreignThread = await app.inject({ method: 'GET', url: `/api/inbox/threads/${threadIds.resolved}`, headers: foreign.headers });
  assert.equal(foreignThread.statusCode, 404);
  const foreignEffects = idsOf(rival);
  assert.deepEqual(foreignEffects, { tasks: 0, pings: 0 });
  const ownerPings = await app.inject({ method: 'GET', url: '/api/tasks/pings?limit=50', headers: owner.headers });
  assert.equal(ownerPings.statusCode, 200, ownerPings.body);
  const pings = ownerPings.json().items.filter((item) => item.body.includes('J14 boundary'));
  assert.ok(pings.length >= 4, 'owner-visible pings omitted a selected J14 marker');
  record.race = { happensBefore: ['J14 SELECT captured eligible thread rows', 'owner HTTP done/snooze/reply committed', 'J14 resumed createAutomationTask/postPing/nudged_for'],
```

## Remediation

Check the thread status and latest message while creating the task and alert, so a changed thread cannot receive a stale reminder.
