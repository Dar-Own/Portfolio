# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Medium (same-tenant financial-record integrity)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Nonfinancial equipment manager or assigned incident solver can enter maintenance cost

## Description

A member without financial write permission can store an owner-visible maintenance or incident repair cost.

Prerequisite and affected workflow: A Dispatcher changes an equipment repair cost, or an assigned Crew member records an incident repair cost, despite lacking financial permission.

Established observation: A Dispatcher without financial permission stored a 6,543-cent maintenance cost, and an assigned incident solver without that permission stored a 62,400-cent repair cost in both incident and maintenance records. The Dispatcher could not read the cost through the masked response, but the write persisted.

Limits: Downstream accounting reliance and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A Dispatcher changes an equipment repair cost, or an assigned Crew member records an incident repair cost, despite lacking financial permission.

Source: `apps/api/src/features/equipment/routes.ts:216-230` (1d5d98db9acf).

```typescript
    app.post('/:id/maintenance', { schema: { params: idParams, body: maintenanceBody } }, async (request, reply) => {
      requireManage(request.session);
      const body = request.body;
      const page = await app.db.withUser(request.session, async (ctx) => {
        const view = await viewFor(ctx, request.session);
        const unit = await requireUnit(ctx, request.params.id, view);
        if (unit.retiredAt) throw invalid(`${unit.name} is retired.`);
        await addMaintenance(ctx, unit, {
          kind: body.kind,
          performedOn: body.performedOn,
          meterReading: body.meterReading,
          description: body.description,
          vendor: body.vendor ?? null,
          costCents: body.costCents ?? null,
        });
```

#### Step 2 — Cross the missing state or authority check

Equipment and incident repair writes accept a cost amount without checking financial write permission in the API or database.

Source: `apps/api/src/features/equipment/store.ts:481-499` (1d5d98db9acf).

```typescript
export async function addMaintenance(ctx: DbContext, unit: UnitRecord, input: MaintenanceInput): Promise<string> {
  const kind = input.kind ?? 'service';
  if (!(MAINTENANCE_KINDS as readonly string[]).includes(kind)) throw invalid('Pick whether this was a service, a repair or an inspection.');
  if (!DAY_ONLY.test(input.performedOn)) throw invalid('Enter the date the work was done, like 2026-09-18.');
  const meterReading = checkReading(input.meterReading, 'The meter reading');
  if (meterReading === null) throw invalid('Enter the meter reading at the time of the work.');
  const description = cleanNotes(input.description);
  if (!description) throw invalid('Describe the service or repair.');
  if (description.length > MAX_MAINTENANCE_DESCRIPTION) throw invalid(`The description must be ${MAX_MAINTENANCE_DESCRIPTION} characters or fewer.`);
  const costCents = input.costCents ?? null;
  if (costCents !== null && (!Number.isInteger(costCents) || costCents < 0 || costCents > 99_999_999_999)) {
    throw invalid('Enter the cost as a whole number of cents, zero or more.');
  }
  const id = randomUUID();
  await ctx.client.query(
    `insert into app.maintenance_records (id, tenant_id, asset_id, kind, performed_at, meter_reading, description, vendor, cost_cents)
     values ($1, $2, $3, $4, $5::date, $6, $7, $8, $9)`,
    [id, ctx.tenantId, unit.id, kind, input.performedOn, meterReading, description, line(input.vendor, 200, 'The shop or vendor'), costCents],
  );
```

Source: `apps/api/src/features/incidents/routes.ts:226-242` (1d5d98db9acf).

```typescript
    // "Record the repair": the fix and its cost on the incident, and a maintenance record in the
    // unit's equipment history when the incident names a unit.
    app.post('/:id/repair', { schema: { params: idParams, body: repairBody } }, async (request) => {
      requireAccess(request.session);
      const viewer = viewerOf(request.session);
      const { id } = request.params;
      const body = request.body;
      return app.db.withUser(request.session, async (ctx) => {
        const recordId = await recordRepair(ctx, viewer, id, {
          description: body.description,
          performedOn: body.performedOn ?? null,
          costCents: body.costCents ?? null,
          shop: body.shop ?? null,
        });
        const incident = await requireIncidentDetail(ctx, viewer, id);
        if (incident.status === 'fixed') ctx.emit('incident.resolved', incident.id);
        return { incident, maintenanceRecordId: recordId };
```

Source: `packages/db/migrations/0058_security_review_batch_4.sql:165-208` (1d5d98db9acf).

```sql
  if not app.is_member() then
    raise exception 'You are not a member of this company';
  end if;
  select * into inc from app.incidents i where i.tenant_id = app.current_tenant() and i.id = p_incident;
  if not found then
    raise exception 'We couldn''t find that incident';
  end if;
  if not (app.can('incidents.manage') or (inc.solver_id is not null and inc.solver_id = me)) then
    raise exception 'Only people who manage incidents, or the assigned solver, can record the repair';
  end if;
  if inc.status = 'closed' then
    raise exception 'This incident is closed, so it can''t be changed';
  end if;
  if inc.asset_id is null then
    raise exception 'This incident has no unit, so there is no equipment history to write the repair into';
  end if;
  -- 0056: the file must be one of this company's incident uploads, not any file it happens to own.
  if not app.incident_file_ok(inc.tenant_id, p_receipt_file) then
    raise exception 'That file is not an incident receipt of this company';
  end if;
  if p_cost_cents is not null and p_cost_cents < 0 then
    raise exception 'The repair cost can''t be negative';
  end if;

  if inc.maintenance_record_id is not null then
    update app.maintenance_records r
       set performed_at = coalesce(p_performed_on, r.performed_at),
           description = coalesce(nullif(btrim(p_description), ''), r.description),
           vendor = coalesce(nullif(btrim(p_shop), ''), r.vendor),
           cost_cents = coalesce(p_cost_cents, r.cost_cents),
           invoice_file = coalesce(p_receipt_file, r.invoice_file)
     where r.tenant_id = inc.tenant_id and r.id = inc.maintenance_record_id;
    return inc.maintenance_record_id;
  end if;

  if coalesce(btrim(p_description), '') = '' then
    raise exception 'Describe the repair';
  end if;
  select a.meter_reading into meter from app.assets a where a.tenant_id = inc.tenant_id and a.id = inc.asset_id;
  rid := gen_random_uuid();
  -- 0058: kind 'repair', so the unit's next service point stays where the last real service put it.
  insert into app.maintenance_records (id, tenant_id, asset_id, kind, performed_at, meter_reading, description, vendor, cost_cents, invoice_file, recorded_by)
  values (rid, inc.tenant_id, inc.asset_id, 'repair', coalesce(p_performed_on, current_date), coalesce(meter, 0), btrim(p_description),
          nullif(btrim(p_shop), ''), p_cost_cents, coalesce(p_receipt_file, inc.receipt_file_id), me);
```


#### Step 3 — Observe the durable effect

A Dispatcher without financial permission stored a 6,543-cent maintenance cost, and an assigned incident solver without that permission stored a 62,400-cent repair cost in both incident and maintenance records. The Dispatcher could not read the cost through the masked response, but the write persisted.

Downstream accounting reliance and deployed effect remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/complementary/cases/equipment-maintenance-cost.mjs:69-92` (test source).

```javascript
    assert.equal(lesserDenied.response.statusCode, 403, lesserDenied.response.body);
    assert.deepEqual(lesserDenied.after, lesserDenied.before, 'lesser-role denial changed tracked effects');
    const foreignDenied = await call('foreign-unit-denied', dispatcher, foreignUnit.id, body(`${marker}-FOREIGN`, 6543));
    assert.equal(foreignDenied.response.statusCode, 404, foreignDenied.response.body);
    assert.deepEqual(foreignDenied.after, foreignDenied.before, 'foreign-unit denial changed tracked effects');
    const extraDenied = await call('overpost-actor-and-tenant-denied', dispatcher, ownUnit.id,
      { ...body(`${marker}-OVERPOST`, 6543), recordedBy: owner.session.user.id, tenantId: foreign.session.company.id });
    assert.equal(extraDenied.response.statusCode, 422, extraDenied.response.body);
    assert.deepEqual(extraDenied.after, extraDenied.before, 'overpost denial changed tracked effects');
    const ownerRead = await app.inject({ method: 'GET', url: `/api/equipment/${ownUnit.id}`, headers: owner.headers });
    const dispatcherRead = await app.inject({ method: 'GET', url: `/api/equipment/${ownUnit.id}`, headers: dispatcher.headers });
    assert.equal(ownerRead.statusCode, 200, ownerRead.body);
    assert.equal(dispatcherRead.statusCode, 200, dispatcherRead.body);
    const ownerCost = ownerRead.json().maintenance.find((r) => r.description === `${marker}-DISPATCH`)?.costCents;
    const dispatcherCost = dispatcherRead.json().maintenance.find((r) => r.description === `${marker}-DISPATCH`)?.costCents;
    ctx.record({ label: 'post-write-reader-comparison', method: 'GET', url: `/api/equipment/${ownUnit.id}`,
      ownerStatus: ownerRead.statusCode, dispatcherStatus: dispatcherRead.statusCode,
      ownerCost, dispatcherCost, sqlCost: written?.cost_cents ?? null,
      actorPermissions: { dispatcherEquipmentManage: true, dispatcherCostingView: false,
        dispatcherFleetManage: false, dispatcherMoneyView: false } });
    const costAccepted = costWrite.response.statusCode === 201 && written?.cost_cents === 6543;
    return { verdict: costAccepted ? 'TECHNICAL-BEHAVIOR-PROVED-POLICY-CONDITIONAL' : 'FALSIFIED-IN-TESTED-SCOPE',
      summary: costAccepted
        ? 'R118 accepted a 6,543-cent maintenance cost from a Dispatcher without cost-view permissions. The same response masked it, while a cost-visible owner read it. Financial write-policy intent is unresolved.'
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A Dispatcher changes an equipment repair cost, or an assigned Crew member records an incident repair cost, despite lacking financial permission.

Recorded result: A Dispatcher without financial permission stored a 6,543-cent maintenance cost, and an assigned incident solver without that permission stored a 62,400-cent repair cost in both incident and maintenance records. The Dispatcher could not read the cost through the masked response, but the write persisted.

Scope: Downstream accounting reliance and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/complementary/cases/equipment-maintenance-cost.mjs:69-92` (test source).

```javascript
    assert.equal(lesserDenied.response.statusCode, 403, lesserDenied.response.body);
    assert.deepEqual(lesserDenied.after, lesserDenied.before, 'lesser-role denial changed tracked effects');
    const foreignDenied = await call('foreign-unit-denied', dispatcher, foreignUnit.id, body(`${marker}-FOREIGN`, 6543));
    assert.equal(foreignDenied.response.statusCode, 404, foreignDenied.response.body);
    assert.deepEqual(foreignDenied.after, foreignDenied.before, 'foreign-unit denial changed tracked effects');
    const extraDenied = await call('overpost-actor-and-tenant-denied', dispatcher, ownUnit.id,
      { ...body(`${marker}-OVERPOST`, 6543), recordedBy: owner.session.user.id, tenantId: foreign.session.company.id });
    assert.equal(extraDenied.response.statusCode, 422, extraDenied.response.body);
    assert.deepEqual(extraDenied.after, extraDenied.before, 'overpost denial changed tracked effects');
    const ownerRead = await app.inject({ method: 'GET', url: `/api/equipment/${ownUnit.id}`, headers: owner.headers });
    const dispatcherRead = await app.inject({ method: 'GET', url: `/api/equipment/${ownUnit.id}`, headers: dispatcher.headers });
    assert.equal(ownerRead.statusCode, 200, ownerRead.body);
    assert.equal(dispatcherRead.statusCode, 200, dispatcherRead.body);
    const ownerCost = ownerRead.json().maintenance.find((r) => r.description === `${marker}-DISPATCH`)?.costCents;
    const dispatcherCost = dispatcherRead.json().maintenance.find((r) => r.description === `${marker}-DISPATCH`)?.costCents;
    ctx.record({ label: 'post-write-reader-comparison', method: 'GET', url: `/api/equipment/${ownUnit.id}`,
      ownerStatus: ownerRead.statusCode, dispatcherStatus: dispatcherRead.statusCode,
      ownerCost, dispatcherCost, sqlCost: written?.cost_cents ?? null,
      actorPermissions: { dispatcherEquipmentManage: true, dispatcherCostingView: false,
        dispatcherFleetManage: false, dispatcherMoneyView: false } });
    const costAccepted = costWrite.response.statusCode === 201 && written?.cost_cents === 6543;
    return { verdict: costAccepted ? 'TECHNICAL-BEHAVIOR-PROVED-POLICY-CONDITIONAL' : 'FALSIFIED-IN-TESTED-SCOPE',
      summary: costAccepted
        ? 'R118 accepted a 6,543-cent maintenance cost from a Dispatcher without cost-view permissions. The same response masked it, while a cost-visible owner read it. Financial write-policy intent is unresolved.'
```

## Remediation

Require financial write permission whenever a repair cost is supplied, in both the API and database authorization checks.
