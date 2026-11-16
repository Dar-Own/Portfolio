# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low (financial-record integrity)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Cost-free repair correction leaves incident and maintenance costs inconsistent

## Description

A cost-free repair correction can leave the incident cost null while its linked maintenance row retains the old amount.

Prerequisite and affected workflow: A solver submits a valid incident repair correction with the cost field omitted after a prior cost was recorded.

Established observation: A valid repair correction omitted cost after 62,400 cents had already been recorded. The incident and API response changed cost to null, while the linked maintenance row retained 62,400 cents for the same repair.

Limits: An accounting consumer relying on the conflicting rows and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A solver submits a valid incident repair correction with the cost field omitted after a prior cost was recorded.

Source: `apps/api/src/features/incidents/store.ts:403-435` (1d5d98db9acf).

```typescript
/**
 * Records the repair: the fix notes and cost on the incident, and, when the incident names a unit,
 * a maintenance record in that unit's history (app.record_incident_repair, which also checks that the
 * caller manages incidents or is the solver). An incident still open moves to Fixed; one already
 * fixed just takes the corrections. Returns the maintenance record id, or null without a unit.
 */
export async function recordRepair(ctx: DbContext, viewer: Viewer, incidentId: string, input: RepairInput): Promise<string | null> {
  const incident = await requireIncident(ctx, viewer, incidentId);
  if (!viewer.manages && !incident.isSolver) throw forbidden(NOT_SOLVER_MESSAGE);
  if (incident.status === 'closed') throw ruleViolation("This incident is closed, so it can't be changed.");
  const description = cleanNotes(input.description);
  // Real text, as the database asks of fix notes (migration 0063): a dash or dots describe nothing.
  if (!description || !hasRealText(description)) throw invalid('Describe the repair.');
  if (description.length > 4000) throw invalid('Keep the repair description under 4,000 characters.');
  const cost = input.costCents ?? null;
  if (cost !== null && (!Number.isInteger(cost) || cost < 0 || cost >= MAX_COST_CENTS)) throw invalid('Enter the cost as an amount like 215.00.');
  const shop = cleanLine(input.shop);

  let recordId: string | null = incident.repair?.id ?? null;
  if (incident.unit) {
    const { rows } = await ctx.client.query<{ id: string }>(
      'select app.record_incident_repair($1, $2, $3::date, $4, $5, null) as id',
      [incidentId, description, input.performedOn ?? null, cost, shop],
    );
    recordId = rows[0]?.id ?? recordId;
  }
  const result = await ctx.client.query(
    `update app.incidents
        set fix_notes = $2, cost_cents = $3, maintenance_record_id = coalesce($4, maintenance_record_id),
            status = case when status in ('reported', 'assigned', 'in_repair') then 'fixed' else status end
      where id = $1`,
    [incidentId, description, cost, recordId],
  );
```

#### Step 2 — Cross the missing state or authority check

The omitted-cost correction clears incident cost while the linked maintenance cost keeps its previous value.

Source: `packages/db/migrations/0040_incidents.sql:385-393` (1d5d98db9acf).

```sql
  if inc.maintenance_record_id is not null then
    update app.maintenance_records r
       set performed_at = coalesce(p_performed_on, r.performed_at),
           description = coalesce(nullif(btrim(p_description), ''), r.description),
           vendor = coalesce(nullif(btrim(p_shop), ''), r.vendor),
           cost_cents = coalesce(p_cost_cents, r.cost_cents),
           invoice_file = coalesce(p_receipt_file, r.invoice_file)
     where r.tenant_id = inc.tenant_id and r.id = inc.maintenance_record_id;
    return inc.maintenance_record_id;
```


#### Step 3 — Observe the durable effect

A valid repair correction omitted cost after 62,400 cents had already been recorded. The incident and API response changed cost to null, while the linked maintenance row retained 62,400 cents for the same repair.

An accounting consumer relying on the conflicting rows and deployed effect remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/attack-led/public-files/run-r204-assigned-solver-maintenance-cost.mjs:237-247` (synthetic test).

```javascript
  const correction = await call('R204-Tyler-omitted-cost-correction', 'POST', `/api/incidents/${selected}/repair`, tyler,
    repair('CORRECTION-OMITTED-COST', undefined, performedOn), 200);
  assert.equal(correction.json().maintenanceRecordId, selectedRecord);
  assert.equal(state().maintenance.length, countBefore);
  const incidentAfter = incidentRow(selected), maintenanceAfter = maintenanceRow(selectedRecord);
  raw.effects.omittedCostCorrection = { before: pairBeforeCorrection, after: { incident: incidentAfter, maintenance: maintenanceAfter },
    responseCostCents: correction.json().incident.costCents };
  assert.equal(incidentAfter.cost_cents, null, 'omitted cost did not clear incident cost as source predicts');
  assert.equal(Number(maintenanceAfter.cost_cents), 62400, 'omitted cost did not preserve maintenance cost as source predicts');
  assert.equal(correction.json().incident.costCents, null);
  raw.stage = 'omitted-cost-consistency-case-complete';
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A solver submits a valid incident repair correction with the cost field omitted after a prior cost was recorded.

Recorded result: A valid repair correction omitted cost after 62,400 cents had already been recorded. The incident and API response changed cost to null, while the linked maintenance row retained 62,400 cents for the same repair.

Scope: An accounting consumer relying on the conflicting rows and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/attack-led/public-files/run-r204-assigned-solver-maintenance-cost.mjs:237-247` (synthetic test).

```javascript
  const correction = await call('R204-Tyler-omitted-cost-correction', 'POST', `/api/incidents/${selected}/repair`, tyler,
    repair('CORRECTION-OMITTED-COST', undefined, performedOn), 200);
  assert.equal(correction.json().maintenanceRecordId, selectedRecord);
  assert.equal(state().maintenance.length, countBefore);
  const incidentAfter = incidentRow(selected), maintenanceAfter = maintenanceRow(selectedRecord);
  raw.effects.omittedCostCorrection = { before: pairBeforeCorrection, after: { incident: incidentAfter, maintenance: maintenanceAfter },
    responseCostCents: correction.json().incident.costCents };
  assert.equal(incidentAfter.cost_cents, null, 'omitted cost did not clear incident cost as source predicts');
  assert.equal(Number(maintenanceAfter.cost_cents), 62400, 'omitted cost did not preserve maintenance cost as source predicts');
  assert.equal(correction.json().incident.costCents, null);
  raw.stage = 'omitted-cost-consistency-case-complete';
```

## Remediation

Preserve the prior cost in both records when correction omits cost, updating them together when cost is explicitly changed.
