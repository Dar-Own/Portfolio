# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low (conditional; provisional)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Untouched-lead worker creates a false task/alert after staff already touched lead

## Description

A lead can receive a false untouched task, alert, and activity line after staff have already recorded contact.

Prerequisite and affected workflow: The worker selects a lead, then staff record contact before the reminder is saved.

Established observation: Staff recorded a lead contact note after the worker selected the lead but before it saved its work. The worker still created an untouched-lead task, alert, and activity entry; contact recorded before selection suppressed them.

Limits: The client cutoff rule, natural overlap, and deployed effect remain unresolved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

The worker selects a lead, then staff record contact before the reminder is saved.

Source: `apps/worker/src/jobs/untouched-watchdog.ts:92-185` (1d5d98db9acf).

```typescript
  return untouchedSettingsFromRow(rows[0] ?? null);
}

async function watchTenant(ctx: JobContext, client: Client, tenantId: string, now: Date, tally: UntouchedTally, maxPerRun: number): Promise<void> {
  const untouched = await loadUntouchedSettings(client, tenantId);
  if (!untouched.enabled) return;
  const settings = await loadAutomationSettings(client, tenantId);

  // Only leads whose reminder is due now come back, so the page is never taken up by leads that are
  // not due yet (a lead in a stage with a long limit, touched a while ago): those used to sit at the
  // front of the page run after run, and a company with more than a page of them was never reminded
  // about the rest. The wall-clock test goes first because it is cheap and business minutes never
  // exceed wall-clock minutes; app.business_minutes then applies the stage's own limit (the shared
  // business-time maths agrees with it). Leads never reminded go first, then the ones reminded
  // longest ago, then the longest idle, so past the cap everyone is reached in turn.
  const { rows } = await client.query<IdleLead>(
    `select x.lead_id, x.stage, x.stage_entered_at, x.last_touch_at, l.assigned_to, c.display_name as lead_name
       from app.lead_last_touch($1) x
       cross join lateral (select greatest(x.stage_entered_at, coalesce(x.last_touch_at, x.stage_entered_at)) as idle_since,
                                  ($3::jsonb ->> x.stage)::int as hours) h
       join app.leads l on l.tenant_id = $1 and l.id = x.lead_id
       join app.customers c on c.tenant_id = l.tenant_id and c.id = l.customer_id
      where c.deleted_at is null
        and h.idle_since <= $2::timestamptz - make_interval(hours => h.hours)
        and app.business_minutes($1, h.idle_since, $2::timestamptz) >= h.hours * 60
        -- Already reminded for this idle period (a touch-base task made since the last touch).
        and not exists (select 1 from app.tasks t
                         where t.tenant_id = $1
                           and t.dedupe_key = 'auto:untouched:' || x.lead_id || ':'
                             || to_char(x.stage_entered_at at time zone 'UTC', 'YYYY-MM-DD"T"HH24:MI:SS.MS"Z"') || ':'
                             || coalesce(to_char(x.last_touch_at at time zone 'UTC', 'YYYY-MM-DD"T"HH24:MI:SS.MS"Z"'), 'none'))
      order by (select max(t.created_at) from app.tasks t
                 where t.tenant_id = $1 and t.lead_id = x.lead_id and t.dedupe_key like 'auto:untouched:%') nulls first,
               h.idle_since, x.lead_id
      limit $4`,
    [tenantId, now, JSON.stringify(untouched.hoursByStage), maxPerRun + 1],
  );
  if (rows.length > maxPerRun) {
    ctx.logger.info(
      `untouched-watchdog: company ${tenantId} has more than ${maxPerRun} ${maxPerRun === 1 ? 'lead' : 'leads'} due a reminder; the rest go first next run.`,
    );
    rows.length = maxPerRun;
  }

  for (const lead of rows) {
    const hours = untouched.hoursByStage[lead.stage];
    if (hours === undefined) continue; // not a chased stage (the database only returns open ones)
    const elapsed = businessMinutesBetween(untouchedIdleSince(lead.stage_entered_at, lead.last_touch_at), now, settings.hours);
    if (!untouchedReminderDue(elapsed, hours)) continue;
    const reminded = await inSavepoint(client, () => remind(client, tenantId, lead, elapsed, hours, settings, now));
    if (reminded.ok) {
      if (reminded.value) tally.reminders++;
    } else {
      tally.errors++;
      ctx.logger.error(`untouched-watchdog: lead ${lead.lead_id}: ${errorText(reminded.error)}`);
    }
  }
}

/** The task, then the ping and the timeline line when the task was new. */
async function remind(
  client: Client,
  tenantId: string,
  lead: IdleLead,
  elapsedMinutes: number,
  hours: number,
  settings: AutomationSettings,
  now: Date,
): Promise<boolean> {
  const byStage = lead.stage === 'new' || lead.stage === 'sizing' ? settings.routing.intake : settings.routing.tier_2;
  const owner = await findTaskOwner(client, tenantId, [lead.assigned_to, byStage, settings.routing.escalation]);
  if (!owner) throw new Error('no one to send the reminder to: set the routing rules on the Automations page');

  const created = await createAutomationTask(client, {
    tenantId,
    title: untouchedTaskTitle(lead.lead_name, hours),
    kind: UNTOUCHED_TASK_KIND,
    ownerId: owner.id,
    leadId: lead.lead_id,
    dueAt: now,
    dedupeKey: untouchedDedupeKey(lead.lead_id, lead.stage_entered_at, lead.last_touch_at),
  });
  if (!created) return false;

  await postPing(
    client,
    tenantId,
    '#alerts',
    untouchedSlackText({ leadName: lead.lead_name, stage: lead.stage, elapsedMinutes, hours, ownerName: owner.fullName }),
    { forUsers: [owner.id], path: `/pipelines/${lead.lead_id}`, textSummary: `Untouched: ${lead.lead_name} needs a touch.` },
  );
  await client.query('select app.add_activity_note($1, $2)', [untouchedActivityNote({ leadName: lead.lead_name, hours, ownerName: owner.fullName }), lead.lead_id]);
  return true;
}
```

#### Step 2 — Cross the missing state or authority check

The untouched-lead worker creates reminders from an earlier selection without checking the current last-contact time or lead stage.

Source: `packages/db/migrations/0033_untouched_reminders.sql:55-100` (1d5d98db9acf).

```sql
create or replace function app.lead_last_touch(p_tenant uuid)
returns table (lead_id uuid, stage text, stage_entered_at timestamptz, last_touch_at timestamptz)
language plpgsql stable security definer set search_path = app, pg_temp as $$
begin
  -- No admin exception: app.is_admin_session() looks at current_user, which inside this SECURITY DEFINER
  -- body is the function's owner. A superuser that wants the report sets app.tenant_id first.
  if p_tenant is null or p_tenant is distinct from app.current_tenant() then
    raise exception 'The last-touch report is only for the company the session is acting in';
  end if;
  return query
    select l.id, l.stage::text, l.stage_entered_at,
           greatest(
             -- a stage change by a person
             (select max(h.changed_at) from app.stage_history h
                join app.users u on u.id = h.changed_by and not u.is_system
               where h.tenant_id = l.tenant_id and h.lead_id = l.id),
             -- a note or call note by a person
             (select max(n.created_at) from app.lead_notes n
                join app.users u on u.id = n.created_by and not u.is_system
               where n.tenant_id = l.tenant_id and n.lead_id = l.id),
             -- a message a person sent, or the customer's own message in
             (select max(m.created_at) from app.messages m
                left join app.users u on u.id = m.created_by
               where m.tenant_id = l.tenant_id and m.lead_id = l.id
                 and (m.direction = 'inbound' or (u.id is not null and not u.is_system))),
             -- a task about the lead a person created
             (select max(t.created_at) from app.tasks t
                join app.users u on u.id = t.created_by and not u.is_system
               where t.tenant_id = l.tenant_id and t.lead_id = l.id),
             -- a task about the lead a person completed
             (select max(t.completed_at) from app.tasks t
                join app.users u on u.id = t.completed_by and not u.is_system
               where t.tenant_id = l.tenant_id and t.lead_id = l.id and t.completed_at is not null),
             -- a quote appointment a person booked
             (select max(a.created_at) from app.appointments a
                join app.users u on u.id = a.created_by and not u.is_system
               where a.tenant_id = l.tenant_id and a.lead_id = l.id),
             -- a quote appointment marked done (only people do that)
             (select max(a.completed_at) from app.appointments a
               where a.tenant_id = l.tenant_id and a.lead_id = l.id and a.completed_at is not null)
           )
      from app.leads l
     where l.tenant_id = p_tenant
       and l.record_type = 'sales_job'
       and l.stage in ('new', 'sizing', 'estimating', 'proposal_sent', 'follow_up')
     order by l.stage_entered_at, l.id;
```

Source: `apps/api/src/features/leads/routes.ts:151-160` (1d5d98db9acf).

```typescript
    app.post('/:id/notes', { schema: { params: idParams, body: noteBody } }, async (request) => {
      requireWorkRight(request.session);
      const { id } = request.params;
      const body = request.body;
      return app.db.withUser(request.session, async (ctx) => {
        await requireLead(ctx, id);
        const note = await addLeadNote(ctx, id, { kind: body.kind ?? 'call', body: body.body });
        return { noteId: note.id };
      });
    });
```


#### Step 3 — Observe the durable effect

Staff recorded a lead contact note after the worker selected the lead but before it saved its work. The worker still created an untouched-lead task, alert, and activity entry; contact recorded before selection suppressed them.

The client cutoff rule, natural overlap, and deployed effect remain unresolved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/untouched-effect/run-note-after-scan.mjs:218-234` (synthetic test).

```javascript
    pingStatus: pingPage.statusCode, visiblePing: { body: visiblePing.body, channel: visiblePing.channel },
  };
  result.race = { selection, before, touchStatus: touched.statusCode, afterTouch, currentClock, job, after, stale, staffApi };
  assert.equal(job.errors, 0, JSON.stringify(job));
  assert.equal(after.church.open_tasks, before.church.open_tasks + 1, 'stale untouched task missing');
  assert.equal(after.church.untouched_alerts, before.church.untouched_alerts + 1, 'stale untouched alert missing');
  assert.equal(after.church.untouched_activity, before.church.untouched_activity + 1, 'stale activity note missing');
  assert.equal(after.gwinnett.untouched_tasks, preEffects.untouched_tasks, 'pre-selection touched control got an untouched task');
  assert.ok(after.marietta.untouched_tasks > before.marietta.untouched_tasks, 'unchanged Rival positive did not get a reminder');
  assert.equal(after.church.messages, afterTouch.messages, 'J26 unexpectedly queued a customer message');
  assert.match(stale.task_title, /^Touch base:/);
  assert.match(stale.alert_body, /Untouched:/);
  const retry = await runUntouchedWatchdog(ctx);
  const afterRetry = effects(peach, church);
  assert.equal(retry.errors, 0, JSON.stringify(retry));
  assert.deepEqual(afterRetry, after.church, 'retry unexpectedly cleaned or repeated stale reminder');
  result.race = {
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: The worker selects a lead, then staff record contact before the reminder is saved.

Recorded result: Staff recorded a lead contact note after the worker selected the lead but before it saved its work. The worker still created an untouched-lead task, alert, and activity entry; contact recorded before selection suppressed them.

Scope: The client cutoff rule, natural overlap, and deployed effect remain unresolved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/untouched-effect/run-note-after-scan.mjs:218-234` (synthetic test).

```javascript
    pingStatus: pingPage.statusCode, visiblePing: { body: visiblePing.body, channel: visiblePing.channel },
  };
  result.race = { selection, before, touchStatus: touched.statusCode, afterTouch, currentClock, job, after, stale, staffApi };
  assert.equal(job.errors, 0, JSON.stringify(job));
  assert.equal(after.church.open_tasks, before.church.open_tasks + 1, 'stale untouched task missing');
  assert.equal(after.church.untouched_alerts, before.church.untouched_alerts + 1, 'stale untouched alert missing');
  assert.equal(after.church.untouched_activity, before.church.untouched_activity + 1, 'stale activity note missing');
  assert.equal(after.gwinnett.untouched_tasks, preEffects.untouched_tasks, 'pre-selection touched control got an untouched task');
  assert.ok(after.marietta.untouched_tasks > before.marietta.untouched_tasks, 'unchanged Rival positive did not get a reminder');
  assert.equal(after.church.messages, afterTouch.messages, 'J26 unexpectedly queued a customer message');
  assert.match(stale.task_title, /^Touch base:/);
  assert.match(stale.alert_body, /Untouched:/);
  const retry = await runUntouchedWatchdog(ctx);
  const afterRetry = effects(peach, church);
  assert.equal(retry.errors, 0, JSON.stringify(retry));
  assert.deepEqual(afterRetry, after.church, 'retry unexpectedly cleaned or repeated stale reminder');
  result.race = {
```

## Remediation

Recheck the current last-touch clock and lead stage at the final task, alert, and activity write.
