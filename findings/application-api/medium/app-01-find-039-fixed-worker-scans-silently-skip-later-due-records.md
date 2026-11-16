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

# Fixed worker scans silently skip later due records

## Description

Later due driver, vendor, equipment, ticket, or lead records can receive no task or alert on repeated worker runs. The equipment due board can also omit a due unit from its summary.

Prerequisite and affected workflow: An eligible due record falls beyond a fixed 200- or 500-row scan prefix while earlier records remain selected.

Established observation: Repeated worker runs missed eligible records placed just beyond the fixed scan limits: the 201st driver and vendor records and rank-501 equipment, ticket, and lead records received no reminder. Moving one earlier record out of the prefix caused each skipped record to be processed; the equipment due board also omitted its target.

Limits: Real tenant volumes, operational reliance, and deployed scheduler behavior remain unmeasured. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

An eligible due record falls beyond a fixed 200- or 500-row scan prefix while earlier records remain selected.

Source: `apps/worker/src/jobs/compliance-reminders.ts:120-162` (1d5d98db9acf).

```typescript
      // The current document per driver and kind (the one that runs out last), for people still employed.
      const { rows } = await client.query<CurrentDocumentRow>(
        `select d.id, d.item, d.expires_on::text as expires_on, d.driver_name, d.driver_user_id
           from (
             select distinct on (f.employee_id, f.item)
                    f.id, f.item, f.expires_on,
                    trim(e.first_name || ' ' || e.last_name) as driver_name,
                    -- The membership's payroll link is out of the table grant (migration 0083);
                    -- app.employee_user_id answers for the Automation member.
                    app.employee_user_id(e.id) as driver_user_id
               from app.driver_files f
               join app.employees e on e.tenant_id = f.tenant_id and e.id = f.employee_id
              where f.tenant_id = $1 and e.termination_date is null and f.expires_on is not null
              order by f.employee_id, f.item, f.expires_on desc, f.created_at desc, f.id desc
           ) d
          where d.expires_on <= $2::date
          order by d.expires_on, d.driver_name, d.id`,
        [tenantId, addDays(company.today, WARNING_DAYS)],
      );

      // The reminders that should be open now (all of them, so none is cancelled by mistake), and the
      // open ones that are not among them are cancelled.
      // ponytail: a cancelled key stays cancelled if its exact date and step come back (the insert
      // below skips existing keys); reopen it here if the office ever toggles dates back and forth.
      const wanted: string[] = [];
      for (const document of rows) {
        const step = reminderStep(daysBetween(company.today, document.expires_on));
        if (step === null) continue;
        const key = `compliance-file:${document.id}:${document.expires_on}:${step}`;
        wanted.push(`${key}:office`);
        if (document.driver_user_id && document.driver_user_id !== company.dispatcher?.userId) wanted.push(`${key}:driver`);
      }
      await client.query(
        `update app.tasks set status = 'canceled'
          where tenant_id = $1 and status = 'open' and dedupe_key like 'compliance-file:%' and dedupe_key <> all($2::text[])`,
        [tenantId, wanted],
      );

      if (rows.length === 0) return;
      if (!company.dispatcher) {
        throw new Error('Nobody may manage driver compliance, so there is no one to remind. Give the operations person that permission.');
      }
      for (const document of rows.slice(0, MAX_PER_RUN)) {
```

#### Step 2 — Cross the missing state or authority check

Affected scans impose fixed row limits before filtering for actual due status or prior reminder state.

Source: `apps/worker/src/jobs/vendor-coi-reminders.ts:42-79` (1d5d98db9acf).

```typescript
  const scan = await forEachActiveTenant(ctx.pool, async (tenantId) => {
    const company = await withAutomation(ctx.pool, tenantId, (client) => reminderCompany(client, tenantId, now));
    await withAutomation(ctx.pool, tenantId, async (client) => {
      const { rows } = await client.query<ExpiringVendorRow>(
        // ::text because the worker's connections hand dates back as JavaScript Dates.
        `select v.id, v.company_name, v.coi_expires_on::text as coi_expires_on
           from app.vendors v
          where v.tenant_id = $1 and v.kind = 'subcontractor' and v.is_active
            and v.coi_expires_on is not null and v.coi_expires_on <= $2::date
          order by v.coi_expires_on, v.id
          limit $3`,
        [tenantId, addDays(company.today, company.coiWarningDays), MAX_PER_RUN],
      );
      if (rows.length === 0) return;
      if (!company.office) {
        throw new Error('Nobody is set up to look after paperwork, so there is no one to remind. Set the office routing rule.');
      }
      for (const vendor of rows) {
        const daysUntilExpiry = daysBetween(company.today, vendor.coi_expires_on);
        const message = {
          company: vendor.company_name,
          expiresLabel: formatDateLabel(vendor.coi_expires_on),
          daysUntilExpiry,
        };
        const created = await createReminderTask(client, {
          tenantId,
          title: coiReminderTaskTitle(message),
          ownerId: company.office.userId,
          // Get it before it runs out: due on the expiry day, or now when that day has passed.
          dueAt: daysUntilExpiry <= 0 ? now : atLocalHour(vendor.coi_expires_on, 9, company.timeZone),
          dedupeKey: `vendor-coi:${vendor.id}:${vendor.coi_expires_on}`,
        });
        if (!created) continue;
        await postNotification(client, tenantId, PRODUCTION_CHANNEL, coiReminderNotification(message), {
          forUsers: [company.office.userId],
          path: '/vendors',
          textSummary: `${message.company}'s certificate of insurance expires ${message.expiresLabel}.`,
        });
```

Source: `apps/worker/src/jobs/service-due.ts:68-113` (1d5d98db9acf).

```typescript
  const scan = await forEachActiveTenant(ctx.pool, async (tenantId) => {
    await withAutomation(ctx.pool, tenantId, async (client) => {
      const { rows } = await client.query<UnitRow>(
        `select a.id, a.name, a.meter_unit, a.meter_reading, a.next_service_at
           from app.assets a
          where a.tenant_id = $1 and a.retired_at is null and a.next_service_at is not null
          order by lower(a.name), a.id
          limit $2`,
        [tenantId, MAX_PER_RUN],
      );

      // A recorded service moved the unit's service point past the one its open task is about.
      const units = new Map(rows.map((row) => [row.id, row]));
      const open = await client.query<{ id: string; title: string; dedupe_key: string }>(
        `select t.id, t.title, t.dedupe_key from app.tasks t
          where t.tenant_id = $1 and t.status = 'open' and t.dedupe_key like 'auto:service-due:%'`,
        [tenantId],
      );
      for (const task of open.rows) {
        const [, , unitId = '', point = ''] = task.dedupe_key.split(':');
        const unit = units.get(unitId);
        if (!unit || unit.next_service_at === null || Number(unit.next_service_at) <= Number(point)) continue;
        await client.query(
          `update app.tasks set status = 'done', completed_at = now(), title = $3 where tenant_id = $1 and id = $2 and status = 'open'`,
          [tenantId, task.id, serviceDoneTaskTitle(task.title, formatMeter(unit.next_service_at, unit.meter_unit))],
        );
        closed++;
      }

      const due = rows
        .map((row) => ({ row, status: serviceStatus({ meterUnit: row.meter_unit, meterReading: row.meter_reading, nextServiceAt: row.next_service_at }) }))
        .filter((unit) => isDueForService(unit.status));
      if (due.length === 0) return;

      const owner = await serviceTaskOwner(client, tenantId);
      if (!owner) {
        throw new Error('Nobody is set up to look after equipment service (no VP Operations, no operations routing person, no owner).');
      }

      for (const { row, status } of due) {
        const nextServiceAt = status.nextServiceAt ?? 0;
        const overdue = status.state === 'overdue';
        const message = {
          unitName: row.name,
          meterLabel: formatMeter(row.meter_reading, row.meter_unit),
          dueAtLabel: formatMeter(nextServiceAt, row.meter_unit),
```


#### Step 3 — Observe the durable effect

Repeated worker runs missed eligible records placed just beyond the fixed scan limits: the 201st driver and vendor records and rank-501 equipment, ticket, and lead records received no reminder. Moving one earlier record out of the prefix caused each skipped record to be processed; the equipment due board also omitted its target.

Real tenant volumes, operational reliance, and deployed scheduler behavior remain unmeasured.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/worker-frontier/run-vendor-coi-page.mjs:177-200` (test source).

```javascript
  record.cases.push({ label: '201st-vendor-starved', targetId,
    before: beforeStarved, secondRun, afterSecond, thirdRun, afterThird });

  const freeSlot = await app.inject({ method: 'PATCH', url: `/api/vendors/${firstIds[0]}`,
    headers: owner.headers, payload: { coiExpiresOn: '2028-01-01' } });
  assert.equal(freeSlot.statusCode, 200, freeSlot.body);
  const beforeFreed = effects(peach.id, targetId);
  const freedRun = await remindAboutCertificates(ctx);
  const afterFreed = effects(peach.id, targetId);
  assert.equal(freedRun.errors, 0, JSON.stringify(freedRun));
  assert.equal(afterFreed.target_tasks, 1, 'target still missed after earlier slot freed');
  assert.equal(afterFreed.notifications, beforeFreed.notifications + 1, 'target alert not created');
  assert.deepEqual(effects(rival.id), foreignBefore, 'rival changed under A worker backlog');
  record.cases.push({ label: 'owner-edit-frees-slot', editStatus: freeSlot.statusCode,
    before: beforeFreed, job: freedRun, after: afterFreed, rivalBefore: foreignBefore, rivalAfter: effects(rival.id) });
  record.state = 'PROVED-IN-LOCAL-SYNTHETIC-SCOPE';
  record.limits = 'No real vendor documents, scheduled AWS worker, natural subcontractor volume, positive B-due fairness control or provider delivery tested. All 201 vendor rows used the actual authenticated HTTP producer.';
} catch (error) {
  record.error = String(error?.stack ?? error);
  throw error;
} finally {
  try { if (app) await app.close(); } catch (error) { record.cleanupError = `app: ${String(error)}`; }
  try { if (ctx) await ctx.close(); } catch (error) { record.cleanupError = `context: ${String(error)}`; }
  let stopped = !started;
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: An eligible due record falls beyond a fixed 200- or 500-row scan prefix while earlier records remain selected.

Recorded result: Repeated worker runs missed eligible records placed just beyond the fixed scan limits: the 201st driver and vendor records and rank-501 equipment, ticket, and lead records received no reminder. Moving one earlier record out of the prefix caused each skipped record to be processed; the equipment due board also omitted its target.

Scope: Real tenant volumes, operational reliance, and deployed scheduler behavior remain unmeasured. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/worker-frontier/run-vendor-coi-page.mjs:177-200` (test source).

```javascript
  record.cases.push({ label: '201st-vendor-starved', targetId,
    before: beforeStarved, secondRun, afterSecond, thirdRun, afterThird });

  const freeSlot = await app.inject({ method: 'PATCH', url: `/api/vendors/${firstIds[0]}`,
    headers: owner.headers, payload: { coiExpiresOn: '2028-01-01' } });
  assert.equal(freeSlot.statusCode, 200, freeSlot.body);
  const beforeFreed = effects(peach.id, targetId);
  const freedRun = await remindAboutCertificates(ctx);
  const afterFreed = effects(peach.id, targetId);
  assert.equal(freedRun.errors, 0, JSON.stringify(freedRun));
  assert.equal(afterFreed.target_tasks, 1, 'target still missed after earlier slot freed');
  assert.equal(afterFreed.notifications, beforeFreed.notifications + 1, 'target alert not created');
  assert.deepEqual(effects(rival.id), foreignBefore, 'rival changed under A worker backlog');
  record.cases.push({ label: 'owner-edit-frees-slot', editStatus: freeSlot.statusCode,
    before: beforeFreed, job: freedRun, after: afterFreed, rivalBefore: foreignBefore, rivalAfter: effects(rival.id) });
  record.state = 'PROVED-IN-LOCAL-SYNTHETIC-SCOPE';
  record.limits = 'No real vendor documents, scheduled AWS worker, natural subcontractor volume, positive B-due fairness control or provider delivery tested. All 201 vendor rows used the actual authenticated HTTP producer.';
} catch (error) {
  record.error = String(error?.stack ?? error);
  throw error;
} finally {
  try { if (app) await app.close(); } catch (error) { record.cleanupError = `app: ${String(error)}`; }
  try { if (ctx) await ctx.close(); } catch (error) { record.cleanupError = `context: ${String(error)}`; }
  let stopped = !started;
```

## Remediation

Page through eligible due records or persist a fair cursor, applying work caps after due and prior-alert filtering.
