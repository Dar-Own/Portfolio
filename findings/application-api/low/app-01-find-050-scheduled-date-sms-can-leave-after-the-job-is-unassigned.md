# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low (conditional communication integrity)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Scheduled-date SMS can leave after the job is unassigned

## Description

A queued scheduled-date SMS can hand off an obsolete date after the job has been unassigned.

Prerequisite and affected workflow: A date message is queued, staff unschedule the job, and the queued message is processed afterward.

Established observation: A scheduled-date text was queued, then staff unassigned the job before message handoff. The application handed the obsolete date to the SMS adapter; unassignment before queueing suppressed that message.

Limits: Actual carrier receipt, deployed ordering, and the client communication rule remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A date message is queued, staff unschedule the job, and the queued message is processed afterward.

Source: `apps/api/src/features/schedule/routes.ts:144-151` (1d5d98db9acf).

```typescript
    app.post('/jobs/:jobId/assign', { schema: { params: jobParams, body: assignBody } }, async (request) => {
      requireScheduleEdit(request.session);
      return app.db.withUser(request.session, (ctx) => assignJob(ctx, request.params.jobId, request.body));
    });

    app.post('/jobs/:jobId/unassign', { schema: { params: jobParams, body: unassignBody } }, async (request) => {
      requireScheduleEdit(request.session);
      return app.db.withUser(request.session, (ctx) => unassignJob(ctx, request.params.jobId, request.body));
```

#### Step 2 — Cross the missing state or authority check

The queued text message retains the original scheduled date and reaches outbound SMS processing without checking the current assignment.

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

Source: `packages/db/migrations/0108_mail_manager.sql:1835-1895` (1d5d98db9acf).

```sql
  if (new.channel, new.direction, new.to_address, new.recipient_user_id) is distinct from (old.channel, old.direction, old.to_address, old.recipient_user_id)
     and not app.is_admin_session() then
    raise exception 'A message''s channel, direction and recipient can''t be changed';
  end if;
  -- 0108: a reply's subject and threading headers are written once, as the letter is queued.
  if (new.subject, new.in_reply_to, new.references_header) is distinct from (old.subject, old.in_reply_to, old.references_header)
     and not app.is_admin_session() then
    raise exception 'A letter''s subject and threading headers are set when it is queued, never changed after';
  end if;
  -- 0108 stage 5: so is the AI draft a letter is.
  if new.ai_draft_id is distinct from old.ai_draft_id and not app.is_admin_session() then
    raise exception 'The AI draft a letter carries is named when it is queued, never changed after';
  end if;
  if new.template_key is distinct from old.template_key
     and (coalesce(lower(new.template_key) like 'security\_%', false) or coalesce(lower(old.template_key) like 'security\_%', false))
     and not app.is_admin_session() then
    raise exception 'A security notice stays a security notice';
  end if;
  if coalesce(lower(old.template_key) like 'security\_%', false)
     and not app.is_admin_session() and not app.is_automation_member() then
    raise exception '[redacted name] writes and sends a security warning itself: nobody edits, retries or un-sends one by hand';
  end if;
  -- Leftover A: a recipient-marked row that is NOT a security notice (an ordinary internal alert) is
  -- readable only by the recipient, a users.manage holder and the Automation member (0077's
  -- internal_alert_select). Reading it is shut, but a blind `update app.messages set ...` reads nothing, so
  -- no SELECT policy stops it and 0003's update policy asks only for messages.send -- so without this a
  -- member could rewrite or un-send a colleague's alert they cannot see. The security_ rows are already
  -- handled above; this lets exactly the readers change one, so Try again on a colleague's failed alert
  -- still works for a users.manage holder.
  if old.recipient_user_id is not null and not app.is_admin_session()
     and old.recipient_user_id is distinct from app.current_user_id()
     and not app.can('users.manage') and not app.is_automation_member() then
    raise exception 'You can only change an internal alert you''re allowed to see';
  end if;
  if new.status = old.status or 'sms' not in (old.channel, new.channel) then
    return new;
  end if;
  if old.status = 'blocked' then
    raise exception 'A blocked text can''t be sent or retried';
  end if;
  if (old.status = 'received' or 'inbound' in (old.direction, new.direction))
     and new.status in ('queued', 'sent', 'delivered') then
    raise exception 'An incoming text can''t be sent';
  end if;
  if new.status = 'queued' and old.status <> 'failed' and not app.is_admin_session() then
    raise exception 'Only a failed text can be queued again';
  end if;
  if new.channel = 'sms' and new.direction = 'outbound'
     and new.status in ('queued', 'sent', 'delivered')
     and not (old.status = 'sent' and new.status = 'delivered')
     and not exists (
       select 1 from app.current_consents c
       where c.tenant_id = new.tenant_id and c.channel = 'sms'
         and c.address = new.to_address and c.granted
     )
     and not app.is_missed_call_reply(new.tenant_id, new.id, new.to_address, new.template_key, new.body)
     and not (new.recipient_user_id is not null and app.internal_text_allowed(new.tenant_id, new.to_address)) then
    new.status := 'blocked';
    new.blocked_reason := 'no_sms_consent';
    new.sent_at := old.sent_at;                   -- nothing was sent by this change
  end if;
```


#### Step 3 — Observe the durable effect

A scheduled-date text was queued, then staff unassigned the job before message handoff. The application handed the obsolete date to the SMS adapter; unassignment before queueing suppressed that message.

Actual carrier receipt, deployed ordering, and the client communication rule remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/attack-led/workers-bulk/run-schedule-stale-date.mjs:208-224` (synthetic test).

```javascript
    let afterCorrection = null;
    if (name === 'stale') {
      await waitForEvents(app);
      afterCorrection = state(clone, peach, job);
    }
    outcome.cases[name] = { database: clone, tenant: peach, rival, job, crew, day, before, assignStatus: assign.statusCode, assigned,
      queued, controls: { foreignUnassign: foreignUnassign.statusCode, lesserUnassign: lesserUnassign.statusCode, deniedBefore, foreignBefore },
      unassignStatus: unassign?.statusCode ?? null, afterUnassign, beforeSend, tally, afterSend, captures,
      targetCaptures, retryTally, afterCorrection, eventErrors: [...eventErrors(app)] };
    assert.equal(outcome.cases[name].eventErrors.length, 0, `${name}: event handler failed`);
    await app.close(); app = null;
    await ctx.close(); ctx = null;
  }
  assert.ok(outcome.cases.stale.afterUnassign.pending_event_names.includes('schedule.job_unscheduled'), 'unschedule event was not pending at fake handoff');
  assert.ok(outcome.cases.stale.afterSend.pending_event_names.includes('schedule.job_unscheduled'), 'J19 unexpectedly drained unschedule event');
  assert.equal(outcome.cases.pre_event.afterSend.texts.some(text => text.template === 'crew_scheduled'), false);
  outcome.state = 'PROVED-LOCAL-SCHEDULE-DATE-SMS-AFTER-UNASSIGN';
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A date message is queued, staff unschedule the job, and the queued message is processed afterward.

Recorded result: A scheduled-date text was queued, then staff unassigned the job before message handoff. The application handed the obsolete date to the SMS adapter; unassignment before queueing suppressed that message.

Scope: Actual carrier receipt, deployed ordering, and the client communication rule remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/attack-led/workers-bulk/run-schedule-stale-date.mjs:208-224` (synthetic test).

```javascript
    let afterCorrection = null;
    if (name === 'stale') {
      await waitForEvents(app);
      afterCorrection = state(clone, peach, job);
    }
    outcome.cases[name] = { database: clone, tenant: peach, rival, job, crew, day, before, assignStatus: assign.statusCode, assigned,
      queued, controls: { foreignUnassign: foreignUnassign.statusCode, lesserUnassign: lesserUnassign.statusCode, deniedBefore, foreignBefore },
      unassignStatus: unassign?.statusCode ?? null, afterUnassign, beforeSend, tally, afterSend, captures,
      targetCaptures, retryTally, afterCorrection, eventErrors: [...eventErrors(app)] };
    assert.equal(outcome.cases[name].eventErrors.length, 0, `${name}: event handler failed`);
    await app.close(); app = null;
    await ctx.close(); ctx = null;
  }
  assert.ok(outcome.cases.stale.afterUnassign.pending_event_names.includes('schedule.job_unscheduled'), 'unschedule event was not pending at fake handoff');
  assert.ok(outcome.cases.stale.afterSend.pending_event_names.includes('schedule.job_unscheduled'), 'J19 unexpectedly drained unschedule event');
  assert.equal(outcome.cases.pre_event.afterSend.texts.some(text => text.template === 'crew_scheduled'), false);
  outcome.state = 'PROVED-LOCAL-SCHEDULE-DATE-SMS-AFTER-UNASSIGN';
```

## Remediation

Revalidate current assignment and schedule version before the SMS handoff, suppressing superseded dates.
