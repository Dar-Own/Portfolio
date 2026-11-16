# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Alerts and digest identify hidden leads or tickets to members denied those objects

## Description

Alerts and digests identify hidden leads or tickets through names, customer details, numbers, or links shown to members denied the underlying records.

Prerequisite and affected workflow: An SLA, support, or digest worker addresses content to a member who cannot open its linked lead or ticket.

Established observation: Alerts and a daily digest disclosed identifiers or details of synthetic leads and tickets to members whose direct object requests were denied. The digest test exposed a hidden ticket number even though its detail route returned a denial.

Limits: Real alert email delivery was not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

An SLA, support, or digest worker addresses content to a member who cannot open its linked lead or ticket.

Source: `apps/worker/src/jobs/sla-watchdog.ts:100-120` (1d5d98db9acf).

```typescript
 * disabled), with their name for the ping. Null when nobody fits, so the job can log it and move on
 * instead of failing on the task's foreign key.
 */
export async function findTaskOwner(client: Client, tenantId: string, candidates: readonly (string | null | undefined)[]): Promise<TaskOwner | null> {
  for (const candidate of candidates) {
    if (!candidate) continue;
    const { rows } = await client.query<{ id: string; full_name: string }>(
      `select u.id, u.full_name
         from app.memberships m
         join app.users u on u.id = m.user_id
        where m.tenant_id = $1 and m.user_id = $2 and not u.is_system and u.disabled_at is null`,
      [tenantId, candidate],
    );
    const owner = rows[0];
    if (owner) return { id: owner.id, fullName: owner.full_name };
  }
  return null;
}

export interface AutomationTask {
  tenantId: string;
```

#### Step 2 — Cross the missing state or authority check

`findTaskOwner` validates active tenant membership but not lead read rights (`apps/worker/src/jobs/sla-watchdog.ts:100-120`). The SLA producer writes `lead_name` to the alert body and `text_summary`, and links `/pipelines/<lead id>` (`:290-326`). `app.address_alert` combines explicit recipients and channel permission holders without checking the linked object; `app.deliver_alert` queues `text_summary` and link (`packages/db/migrations/0077_alerts_profiles_and_ssn.sql:376-465`). The lead HTTP route requires `leads.view` (`apps/api/src/features/leads/routes.ts:106-115`), while `listNotifications` serves the short summary to the addressed member (`apps/api/src/core/domain/notifications.ts:86-119`). Admin routing accepts any active non-system member (`apps/api/src/features/automations/automations.ts:100-109`).

Source: `packages/db/migrations/0077_alerts_profiles_and_ssn.sql:376-465` (1d5d98db9acf).

```sql
create or replace function app.address_alert() returns trigger
language plpgsql security definer set search_path = app, pg_temp as $$
begin
  if app.is_app_login() and not app.is_automation_member() then
    new.posted_for := null;
    if not app.can('tasks.all') then
      new.text_summary := null;
      new.link := null;
      new.urgent := false;
    end if;
  end if;
  new.recipient_ids := array(
    select distinct x
      from unnest(coalesce(new.recipient_ids, '{}'::uuid[])
                  || array(select unnest(app.members_holding(new.tenant_id, app.alert_channel_permissions(new.channel)))
                            except select coalesce(new.posted_for, app.current_user_id()))) x
      join app.memberships m on m.tenant_id = new.tenant_id and m.user_id = x
      join app.users u on u.id = m.user_id and not u.is_system
     order by x);
  return new;
end $$;
revoke all on function app.address_alert() from public;
create trigger notifications_address before insert on app.notifications
for each row execute function app.address_alert();

-- After insert: one outbox row per recipient and way. The text and the email carry the short summary and
-- the link, never the body. A text is held until quiet hours end unless the alert is urgent and the person
-- asked for urgent alerts anyway. The company's first brand is the sender, as for customer messages.
-- ponytail: a 'push' choice adds one more branch here (a push outbox row), the seam for the phone apps.
create or replace function app.deliver_alert() returns trigger
language plpgsql security definer set search_path = app, pg_temp as $$
declare
  r record;
  brand record;
  tz text;
  line text;
  phone text;
  local_ts timestamp;
  local_t time;
  in_quiet boolean;
  after_ts timestamptz;
begin
  if cardinality(new.recipient_ids) = 0 then
    return null;
  end if;
  select bu.phone_e164::text as phone, bu.sending_email::text as email into brand
    from app.business_units bu where bu.tenant_id = new.tenant_id order by bu.created_at, bu.name, bu.id limit 1;
  select t.timezone into tz from app.tenants t where t.id = new.tenant_id;
  line := coalesce(new.text_summary, 'You have a new alert in [redacted name] (' || new.channel || ').') || coalesce(' ' || new.link, '');

  for r in
    select c.*, u.email::text as user_email
      from app.member_contacts c
      join app.users u on u.id = c.user_id and u.disabled_at is null and not u.is_system
     where c.tenant_id = new.tenant_id and c.user_id = any(new.recipient_ids)
     order by c.user_id
  loop
    if r.alert_channel in ('text', 'text_email') then
      phone := case r.sms_phone when 'work' then r.work_phone else r.personal_phone end;
      if phone is not null then
        local_ts := now() at time zone tz;
        local_t := local_ts::time;
        in_quiet := case when r.quiet_start < r.quiet_end then local_t >= r.quiet_start and local_t < r.quiet_end
                         when r.quiet_start > r.quiet_end then local_t >= r.quiet_start or local_t < r.quiet_end
                         else false end;
        after_ts := null;
        if in_quiet and not (new.urgent and r.urgent_anyway) then
          after_ts := (local_ts::date + r.quiet_end) at time zone tz;
          if after_ts <= now() then
            after_ts := after_ts + interval '1 day';
          end if;
        end if;
        insert into app.messages (tenant_id, channel, direction, to_address, from_address, template_key, body, status,
                                  recipient_user_id, notification_id, send_after)
        values (new.tenant_id, 'sms', 'outbound', phone, coalesce(brand.phone, ''), 'internal_alert',
                line || ' Reply STOP to opt out.', 'queued', r.user_id, new.id, after_ts);
      end if;
    end if;
    if r.alert_channel in ('email', 'text_email') then
      insert into app.messages (tenant_id, channel, direction, to_address, from_address, template_key, body, status,
                                recipient_user_id, notification_id)
      values (new.tenant_id, 'email', 'outbound', r.user_email, coalesce(brand.email, ''), 'internal_alert',
              line, 'queued', r.user_id, new.id);
    end if;
  end loop;
  return null;
end $$;
revoke all on function app.deliver_alert() from public;
create trigger notifications_deliver after insert on app.notifications
for each row execute function app.deliver_alert();
```

Source: `apps/api/src/features/leads/routes.ts:106-115` (1d5d98db9acf).

```typescript
      const session = request.session;
      const drawer = await app.db.withUser(session, (ctx) => loadLeadDrawer(ctx, request.params.id, { can: (key) => can(session, key) }));
      return { ...drawer, canEdit: canWorkLeads(session) };
    });

    app.patch('/:id', { schema: { params: idParams, body: patchBody } }, async (request) => {
      requireWorkRight(request.session);
      // The measured size: only the people who attach the screenshot (the database checks too, 0074).
      if (request.body.asphaltSqft !== undefined) requirePermission(request.session, 'takeoff.use');
      const { id } = request.params;
```


#### Step 3 — Observe the durable effect

Alerts and a daily digest disclosed identifiers or details of synthetic leads and tickets to members whose direct object requests were denied. The digest test exposed a hidden ticket number even though its detail route returned a denial.

Real alert email delivery was not tested.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/campaign-2026-10-08/database-alerts/run-dtc-002.mjs:137-157` (test source).

```javascript
  const fake=delivered.emails.map(e=>({to:addressOf(e),text:e.text,via:e.via}));
  assert.ok(fake.some(e=>e.to==='[redacted email]'&&e.text.includes(mark)));
  assert.equal(effects().foreignOutbox,0);
  // Preference control on a second stage entry: the same denied member is still addressed in-app,
  // but the database must not queue a second email or send it to the fake provider.
  sql(`update app.member_contacts set alert_channel='app' where tenant_id='${a.tenant}' and user_id='${a.denied}';
    update app.leads set stage_entered_at=now()-interval '3 days' where id='${a.lead}';`);
  const beforeDisabled=effects();
  const second=await runSlaWatchdog(ctx,{maxPerRun:10});
  const afterDisabled=effects();
  const deliveredDisabled=await deliverOutbox(app);
  assert.equal(second.processed,1);assert.equal(afterDisabled.deniedOutbox,beforeDisabled.deniedOutbox);
  assert.equal(deliveredDisabled.emails.filter(e=>addressOf(e)==='[redacted email]').length,0);
  console.log(JSON.stringify({revision:pinned,migrations:migrations.length,isolation:{database,dataDirectoryVerified:true,listenAddresses:'',socketOnly:true,spatialShim:true},
    marker:mark,routingHttp:{status:route.statusCode,selectedUser:a.denied},reads,pings,before,job,after,db,fakeProvider:fake,disabledPreference:{before:beforeDisabled,job:second,after:afterDisabled,fakeEmails:deliveredDisabled.emails.length},
    externalDelivery:false},null,2));
} finally {
  if(app) await app.close();
  if(started) spawnSync(path.join(pgBin,'pg_ctl'),['-D',data,'-m','immediate','stop'],{cwd:repo,stdio:'ignore',timeout:30_000});
  rmSync(root,{recursive:true,force:true});
}
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: An SLA, support, or digest worker addresses content to a member who cannot open its linked lead or ticket.

Recorded result: Alerts and a daily digest disclosed identifiers or details of synthetic leads and tickets to members whose direct object requests were denied. The digest test exposed a hidden ticket number even though its detail route returned a denial.

Scope: Real alert email delivery was not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/campaign-2026-10-08/database-alerts/run-dtc-002.mjs:137-157` (test source).

```javascript
  const fake=delivered.emails.map(e=>({to:addressOf(e),text:e.text,via:e.via}));
  assert.ok(fake.some(e=>e.to==='[redacted email]'&&e.text.includes(mark)));
  assert.equal(effects().foreignOutbox,0);
  // Preference control on a second stage entry: the same denied member is still addressed in-app,
  // but the database must not queue a second email or send it to the fake provider.
  sql(`update app.member_contacts set alert_channel='app' where tenant_id='${a.tenant}' and user_id='${a.denied}';
    update app.leads set stage_entered_at=now()-interval '3 days' where id='${a.lead}';`);
  const beforeDisabled=effects();
  const second=await runSlaWatchdog(ctx,{maxPerRun:10});
  const afterDisabled=effects();
  const deliveredDisabled=await deliverOutbox(app);
  assert.equal(second.processed,1);assert.equal(afterDisabled.deniedOutbox,beforeDisabled.deniedOutbox);
  assert.equal(deliveredDisabled.emails.filter(e=>addressOf(e)==='[redacted email]').length,0);
  console.log(JSON.stringify({revision:pinned,migrations:migrations.length,isolation:{database,dataDirectoryVerified:true,listenAddresses:'',socketOnly:true,spatialShim:true},
    marker:mark,routingHttp:{status:route.statusCode,selectedUser:a.denied},reads,pings,before,job,after,db,fakeProvider:fake,disabledPreference:{before:beforeDisabled,job:second,after:afterDisabled,fakeEmails:deliveredDisabled.emails.length},
    externalDelivery:false},null,2));
} finally {
  if(app) await app.close();
  if(started) spawnSync(path.join(pgBin,'pg_ctl'),['-D',data,'-m','immediate','stop'],{cwd:repo,stdio:'ignore',timeout:30_000});
  rmSync(root,{recursive:true,force:true});
}
```

## Remediation

Check each recipient's authority on the linked object before rendering in-app and outgoing alert text, using generic wording when denied.
