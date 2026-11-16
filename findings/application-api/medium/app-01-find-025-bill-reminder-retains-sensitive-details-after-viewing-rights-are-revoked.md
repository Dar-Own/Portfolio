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

# Bill reminder retains sensitive details after viewing rights are revoked

## Description

A bill reminder containing the amount and period can still be addressed to a member after their bill access is revoked.

Prerequisite and affected workflow: The owner revokes bill-view rights after the reminder worker selects the recipient but before email handoff.

Established observation: The bill-reminder worker selected a recipient, then the owner revoked that member's bill-view permission before handoff. Captured mail still received a reminder containing the synthetic payroll amount and period.

Limits: No external email receipt was proved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

The owner revokes bill-view rights after the reminder worker selects the recipient but before email handoff.

Source: `apps/worker/src/jobs/bill-reminders.ts:97-120` (1d5d98db9acf).

```typescript
  // The office person first, then the owner, then everyone else by rank: the first who may see bills
  // gets the reminders (the owner always may, so the list rarely goes past the second row).
  const candidates = await client.query<{ user_id: string; full_name: string; email: string }>(
    `select u.id as user_id, u.full_name, u.email::text as email
       from app.memberships m
       join app.users u on u.id = m.user_id
       join app.roles r on r.tenant_id = m.tenant_id and r.id = m.role_id
      where m.tenant_id = $1 and not u.is_system and u.disabled_at is null
      order by (m.user_id = (select rr.user_id from app.routing_rules rr where rr.tenant_id = m.tenant_id and rr.rule_key = 'office')) desc,
               r.is_owner desc, r.rank desc, u.created_at, u.id
      limit 20`,
    [tenantId],
  );
  let person: { user_id: string; full_name: string; email: string } | undefined;
  for (const candidate of candidates.rows) {
    if (await memberMaySeeBills(client, candidate.user_id)) {
      person = candidate;
      break;
    }
  }

  const brand = await client.query<{ name: string; sending_email: string | null }>(
    `select name, sending_email::text as sending_email from app.business_units
      where tenant_id = $1 order by created_at, name limit 1`,
```

#### Step 2 — Cross the missing state or authority check

The reminder worker caches the recipient and checks bill state at final claim without rechecking the member's bill rights.

The same source excerpt above contains this boundary; the report does not supply a second distinct implementation location.

The same production boundary is reproduced here for this step:

Source: `apps/worker/src/jobs/bill-reminders.ts:97-120` (1d5d98db9acf).

```typescript
  // The office person first, then the owner, then everyone else by rank: the first who may see bills
  // gets the reminders (the owner always may, so the list rarely goes past the second row).
  const candidates = await client.query<{ user_id: string; full_name: string; email: string }>(
    `select u.id as user_id, u.full_name, u.email::text as email
       from app.memberships m
       join app.users u on u.id = m.user_id
       join app.roles r on r.tenant_id = m.tenant_id and r.id = m.role_id
      where m.tenant_id = $1 and not u.is_system and u.disabled_at is null
      order by (m.user_id = (select rr.user_id from app.routing_rules rr where rr.tenant_id = m.tenant_id and rr.rule_key = 'office')) desc,
               r.is_owner desc, r.rank desc, u.created_at, u.id
      limit 20`,
    [tenantId],
  );
  let person: { user_id: string; full_name: string; email: string } | undefined;
  for (const candidate of candidates.rows) {
    if (await memberMaySeeBills(client, candidate.user_id)) {
      person = candidate;
      break;
    }
  }

  const brand = await client.query<{ name: string; sending_email: string | null }>(
    `select name, sending_email::text as sending_email from app.business_units
      where tenant_id = $1 order by created_at, name limit 1`,
```

#### Step 3 — Observe the durable effect

The bill-reminder worker selected a recipient, then the owner revoked that member's bill-view permission before handoff. Captured mail still received a reminder containing the synthetic payroll amount and period.

No external email receipt was proved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/campaign-2026-10-08/workers/run-alert-bill-revocation.mjs:228-258` (synthetic test).

```javascript
      if (!revokedAtClaim && peachFactsRead && typeof sql === 'string' && sql.trim().toLowerCase() === 'commit') {
        const result = await queryBase(sql, args);
        revokedAtClaim = true;
        psql(`delete from app.role_permissions where tenant_id='${peach}' and role_id='${sofia.role_id}'
          and permission in ('bills.view','bills.manage');
          delete from app.member_permission_overrides where tenant_id='${peach}' and user_id='${sofia.id}'
          and permission in ('bills.view','bills.manage');`);
        return result;
      }
      return queryBase(sql, args);
    };
    return client;
  } };
  ctx.pool = pausedPool;
  const staleRun = await remindAboutBills(ctx);
  ctx.pool = pool;
  const stale = { before: staleBefore, revokedAtClaim, after: billState(billIds[1].id), tasks: billTasks(billIds[1].id),
    canViewAfter: await withAutomation(pool, peach, (client) => memberMaySeeBills(client, sofia.id)),
    attempts: fakeAttempts.slice(staleBefore.attempts), run: staleRun };
  assert.equal(revokedAtClaim, true, 'deterministic post-selection revocation fired');
  assert.equal(stale.canViewAfter, false, 'final role/override door denies recipient');
  assert.ok(stale.attempts.some((m) => m.to?.address === sofia.email), 'revoked viewer still fake emailed');
  assert.ok(stale.tasks.some((t) => t.owner_id === sofia.id), 'revoked viewer still owns new task');

  armBill(billIds[2].id);
  const freshBefore = fakeAttempts.length;
  const freshRun = await remindAboutBills(ctx);
  const fresh = { run: freshRun, bill: billState(billIds[2].id), tasks: billTasks(billIds[2].id), attempts: fakeAttempts.slice(freshBefore) };
  assert.ok(fresh.attempts.some((m) => m.to?.address === alex.email), 'fresh selection uses owner after revocation');
  assert.ok(!fresh.attempts.some((m) => m.to?.address === sofia.email), 'fresh selection excludes revoked viewer');

```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: The owner revokes bill-view rights after the reminder worker selects the recipient but before email handoff.

Recorded result: The bill-reminder worker selected a recipient, then the owner revoked that member's bill-view permission before handoff. Captured mail still received a reminder containing the synthetic payroll amount and period.

Scope: No external email receipt was proved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/campaign-2026-10-08/workers/run-alert-bill-revocation.mjs:228-258` (synthetic test).

```javascript
      if (!revokedAtClaim && peachFactsRead && typeof sql === 'string' && sql.trim().toLowerCase() === 'commit') {
        const result = await queryBase(sql, args);
        revokedAtClaim = true;
        psql(`delete from app.role_permissions where tenant_id='${peach}' and role_id='${sofia.role_id}'
          and permission in ('bills.view','bills.manage');
          delete from app.member_permission_overrides where tenant_id='${peach}' and user_id='${sofia.id}'
          and permission in ('bills.view','bills.manage');`);
        return result;
      }
      return queryBase(sql, args);
    };
    return client;
  } };
  ctx.pool = pausedPool;
  const staleRun = await remindAboutBills(ctx);
  ctx.pool = pool;
  const stale = { before: staleBefore, revokedAtClaim, after: billState(billIds[1].id), tasks: billTasks(billIds[1].id),
    canViewAfter: await withAutomation(pool, peach, (client) => memberMaySeeBills(client, sofia.id)),
    attempts: fakeAttempts.slice(staleBefore.attempts), run: staleRun };
  assert.equal(revokedAtClaim, true, 'deterministic post-selection revocation fired');
  assert.equal(stale.canViewAfter, false, 'final role/override door denies recipient');
  assert.ok(stale.attempts.some((m) => m.to?.address === sofia.email), 'revoked viewer still fake emailed');
  assert.ok(stale.tasks.some((t) => t.owner_id === sofia.id), 'revoked viewer still owns new task');

  armBill(billIds[2].id);
  const freshBefore = fakeAttempts.length;
  const freshRun = await remindAboutBills(ctx);
  const fresh = { run: freshRun, bill: billState(billIds[2].id), tasks: billTasks(billIds[2].id), attempts: fakeAttempts.slice(freshBefore) };
  assert.ok(fresh.attempts.some((m) => m.to?.address === alex.email), 'fresh selection uses owner after revocation');
  assert.ok(!fresh.attempts.some((m) => m.to?.address === sofia.email), 'fresh selection excludes revoked viewer');

```

## Remediation

Recheck bill visibility at final claim and provider handoff, then suppress or rebind stale recipients.
