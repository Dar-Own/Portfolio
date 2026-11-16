# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low (planning integrity; downstream harm unproved)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Out-of-service unit remains on a future equipment deployment

## Description

An out-of-service unit can remain listed on a future deployment without an assignment-level unusable flag.

Prerequisite and affected workflow: An authorized equipment manager marks a unit out of service after it was attached to a future assignment.

Established observation: A unit already linked to a future job remained on that deployment after an authorized manager marked it out of service. The assignment view showed the unit without an assignment-level unusable flag, while a new assignment of an already out-of-service unit was rejected.

Limits: Actual field dispatch, safety or service consequences, and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

An authorized equipment manager marks a unit out of service after it was attached to a future assignment.

Source: `apps/api/src/features/equipment/routes.ts:236-247` (1d5d98db9acf).

```typescript
    app.post('/:id/out-of-service', { schema: { params: idParams, body: outOfServiceBody } }, async (request) => {
      requireManage(request.session);
      return app.db.withUser(request.session, async (ctx) => {
        const view = await viewFor(ctx, request.session);
        const unit = await requireUnit(ctx, request.params.id, view);
        if (unit.retiredAt) throw invalid(`${unit.name} is retired.`);
        await setOutOfService(ctx, unit.id, request.body.reason, await appNow(ctx));
        return unitPage(ctx, view, unit.id);
      });
    });

    app.post('/:id/back-in-service', { schema: { params: idParams } }, async (request) => {
```

#### Step 2 — Cross the missing state or authority check

The out-of-service state change leaves existing assignment links intact and the deployment projection does not flag them as unusable.

Source: `packages/db/migrations/0039_equipment.sql:318-342` (1d5d98db9acf).

```sql
create or replace function app.guard_assignment_asset() returns trigger
language plpgsql as $$
declare
  unit record;
begin
  select a.name, a.out_of_service, a.out_of_service_reason, a.retired_at into unit
    from app.assets a where a.tenant_id = new.tenant_id and a.id = new.asset_id;
  if not found then
    raise exception 'We couldn''t find that unit';
  end if;
  if app.is_admin_session() then
    return new;
  end if;
  new.added_by := app.current_user_id();
  if unit.retired_at is not null then
    raise exception '% is retired', unit.name;
  end if;
  if unit.out_of_service then
    raise exception '% is out of service: %', unit.name, unit.out_of_service_reason;
  end if;
  return new;
end $$;

create trigger assignment_assets_guard before insert on app.assignment_assets
for each row execute function app.guard_assignment_asset();
```


#### Step 3 — Observe the durable effect

A unit already linked to a future job remained on that deployment after an authorized manager marked it out of service. The assignment view showed the unit without an assignment-level unusable flag, while a new assignment of an already out-of-service unit was rejected.

Actual field dispatch, safety or service consequences, and deployed effect remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/complementary/cases/equipment-deploy-unsafe.mjs:175-205` (test source).

```javascript
      payload: { assignmentId } });
    const afterLesser = effects();
    ctx.record({ label: 'lesser-role-control', route: 'R121', status: lesser.statusCode,
      before: beforeLesser, after: afterLesser });

    const readNoEffect = JSON.stringify(afterSecondOut) === JSON.stringify(afterReads);
    const foreignNoEffect = rivalCreate.statusCode === 201 && validId(foreignUnitId)
      && [403, 404].includes(foreignAttempt?.statusCode)
      && JSON.stringify(beforeForeign) === JSON.stringify(afterForeign);
    const lesserNoEffect = lesser.statusCode === 403
      && JSON.stringify(beforeLesser) === JSON.stringify(afterLesser);
    if (!readNoEffect || !foreignNoEffect || !lesserNoEffect) return { verdict: 'BLOCKED',
      summary: 'A read, foreign-unit, or lesser-role control failed; inspect the exact effects before interpreting unsafe-state persistence.',
      evidence: { readNoEffect, foreignNoEffect, lesserNoEffect, foreignStatus: foreignAttempt?.statusCode,
        lesserStatus: lesser.statusCode } };
    const persists = afterSecondOut.targetLinks === 1 && afterReads.targetLinks === 1
      && Boolean(linkedDetail) && Boolean(linkedDay?.units?.includes(current.unit.name));
    const assignmentFlaggedUnusable = Boolean(linkedDay?.unusable === true || linkedDay?.outOfService === true
      || linkedDetail?.unusable === true || linkedDetail?.outOfService === true
      || linkedDay?.units?.some((unit) => typeof unit === 'object'
        && unit?.name === current.unit.name && (unit.unusable === true || unit.outOfService === true)));
    return { verdict: persists && !assignmentFlaggedUnusable ? 'TECHNICAL-BEHAVIOR-PROVED' : 'FALSIFIED-IN-SCOPE',
      summary: persists && !assignmentFlaggedUnusable
        ? 'The API refused a new assignment while the unit was out of service, but marking a validly deployed unit out of service left its future assignment linked and listed by unit name, without an assignment-level unusable flag. This contradicts the client-confirmed immediate removal-or-unusable-flag rule.'
        : 'Marking the assigned unit out of service removed, hid, or explicitly marked its existing deployment unusable in the tested scope.',
      limitations: ['The checked-in suite already covers the ordinary attach/out-of-service/retired rule; this case targets persistence after the state transition.',
        'Client rule confirmed on 2026-10-16: marking a unit out of service must remove or flag its existing future assignment unusable immediately. The unit detail may show an out-of-service flag, but that is not an assignment-level unusable signal in the schedule output.',
        'The seed has no Rival schedule assignment, so this tests a foreign unit ID, not a genuinely foreign assignment ID. A same-tenant assignment hidden by schedule scope also remains untested.',
        'No real-world dispatch, injury, provider, or deployed edge is claimed.'],
      evidence: { unitId, assignmentId, workDate: assignment.workDate, persists,
        assignmentFlaggedUnusable,
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: An authorized equipment manager marks a unit out of service after it was attached to a future assignment.

Recorded result: A unit already linked to a future job remained on that deployment after an authorized manager marked it out of service. The assignment view showed the unit without an assignment-level unusable flag, while a new assignment of an already out-of-service unit was rejected.

Scope: Actual field dispatch, safety or service consequences, and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/complementary/cases/equipment-deploy-unsafe.mjs:175-205` (test source).

```javascript
      payload: { assignmentId } });
    const afterLesser = effects();
    ctx.record({ label: 'lesser-role-control', route: 'R121', status: lesser.statusCode,
      before: beforeLesser, after: afterLesser });

    const readNoEffect = JSON.stringify(afterSecondOut) === JSON.stringify(afterReads);
    const foreignNoEffect = rivalCreate.statusCode === 201 && validId(foreignUnitId)
      && [403, 404].includes(foreignAttempt?.statusCode)
      && JSON.stringify(beforeForeign) === JSON.stringify(afterForeign);
    const lesserNoEffect = lesser.statusCode === 403
      && JSON.stringify(beforeLesser) === JSON.stringify(afterLesser);
    if (!readNoEffect || !foreignNoEffect || !lesserNoEffect) return { verdict: 'BLOCKED',
      summary: 'A read, foreign-unit, or lesser-role control failed; inspect the exact effects before interpreting unsafe-state persistence.',
      evidence: { readNoEffect, foreignNoEffect, lesserNoEffect, foreignStatus: foreignAttempt?.statusCode,
        lesserStatus: lesser.statusCode } };
    const persists = afterSecondOut.targetLinks === 1 && afterReads.targetLinks === 1
      && Boolean(linkedDetail) && Boolean(linkedDay?.units?.includes(current.unit.name));
    const assignmentFlaggedUnusable = Boolean(linkedDay?.unusable === true || linkedDay?.outOfService === true
      || linkedDetail?.unusable === true || linkedDetail?.outOfService === true
      || linkedDay?.units?.some((unit) => typeof unit === 'object'
        && unit?.name === current.unit.name && (unit.unusable === true || unit.outOfService === true)));
    return { verdict: persists && !assignmentFlaggedUnusable ? 'TECHNICAL-BEHAVIOR-PROVED' : 'FALSIFIED-IN-SCOPE',
      summary: persists && !assignmentFlaggedUnusable
        ? 'The API refused a new assignment while the unit was out of service, but marking a validly deployed unit out of service left its future assignment linked and listed by unit name, without an assignment-level unusable flag. This contradicts the client-confirmed immediate removal-or-unusable-flag rule.'
        : 'Marking the assigned unit out of service removed, hid, or explicitly marked its existing deployment unusable in the tested scope.',
      limitations: ['The checked-in suite already covers the ordinary attach/out-of-service/retired rule; this case targets persistence after the state transition.',
        'Client rule confirmed on 2026-10-16: marking a unit out of service must remove or flag its existing future assignment unusable immediately. The unit detail may show an out-of-service flag, but that is not an assignment-level unusable signal in the schedule output.',
        'The seed has no Rival schedule assignment, so this tests a foreign unit ID, not a genuinely foreign assignment ID. A same-tenant assignment hidden by schedule scope also remains untested.',
        'No real-world dispatch, injury, provider, or deployed edge is claimed.'],
      evidence: { unitId, assignmentId, workDate: assignment.workDate, persists,
        assignmentFlaggedUnusable,
```

## Remediation

Remove future assignment links atomically on the status change or show an explicit unusable state in every assignment consumer.
