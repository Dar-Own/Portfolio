# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Medium (conditional)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Acknowledged, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Revoked staff session completes a pending platform read or write

## Description

A logged-out MFA staff session can complete an already-started cross-company platform read or billing change.

Prerequisite and affected workflow: The staff member logs out after the database accepts the session for a request but before the protected platform operation executes.

Established observation: A staff member logged out the exact MFA-authenticated session before pending protected SQL ran, yet the in-flight request completed platform reads and billing changes. New requests using the logged-out session were rejected.

Limits: Whether logout must cancel an already-started staff request remains a policy-dependent boundary. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

The staff member logs out after the database accepts the session for a request but before the protected platform operation executes.

Source: `apps/api/src/features/staff/routes.ts:85-103` (1d5d98db9acf).

```typescript
    // A payment the company made to [redacted name].
    app.post('/companies/:id/payments', { schema: { params: idParams, body: paymentBody }, config: STAFF_WRITE }, async (request, reply) => {
      const { id } = request.params;
      const body = request.body;
      const paymentId = await app.db.withUser(request.session, async (ctx) => {
        await requireCompany(ctx, id);
        return recordPayment(ctx, id, body);
      });
      return reply.status(201).send({ id: paymentId });
    });

    app.delete('/companies/:id/payments/:paymentId', { schema: { params: paymentParams }, config: STAFF_WRITE }, async (request, reply) => {
      const { id, paymentId } = request.params;
      await app.db.withUser(request.session, async (ctx) => {
        await requireCompany(ctx, id);
        await deletePayment(ctx, id, paymentId);
      });
      return reply.status(204).send();
    });
```

#### Step 2 — Cross the missing state or authority check

The platform door checks staff membership and MFA after entry without checking the session's revoked state.

Source: `packages/db/migrations/0113_staff_panel.sql:319-338` (1d5d98db9acf).

```sql
revoke all on function platform.plain_text(text, integer, text) from public;

-- Every door's first line. Not a door itself: EXECUTE revoked, it runs only inside the owner's own code.
create function platform.require_staff() returns uuid
language plpgsql stable set search_path = pg_catalog, pg_temp as $$
begin
  if not platform.is_staff() then
    raise exception 'That''s for [redacted name] staff' using errcode = '42501';
  end if;
  return app.current_user_id();
end $$;
revoke all on function platform.require_staff() from public;

create function platform.log_event(p_staff uuid, p_tenant uuid, p_action text, p_details jsonb) returns void
language sql set search_path = pg_catalog, pg_temp as $$
  insert into platform.events (staff_user_id, tenant_id, action, details) values (p_staff, p_tenant, p_action, coalesce(p_details, '{}'::jsonb))
$$;
revoke all on function platform.log_event(uuid, uuid, text, jsonb) from public;

-- The company's billing record, whole. The table's checks say what is allowed (1 to 28, a price of 0 or
```


#### Step 3 — Observe the durable effect

A staff member logged out the exact MFA-authenticated session before pending protected SQL ran, yet the in-flight request completed platform reads and billing changes. New requests using the logged-out session were rejected.

Whether logout must cancel an already-started staff request remains a policy-dependent boundary.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/attack-led/authority-data/run-r362-reactivation-final-authority-2026-10-16.mjs:220-244` (synthetic test).

```javascript
    logoutAt = Date.now();
    afterLogout = effects();
    sessionAfterLogout = sessionRow(staff.token);
    logoutReleased = heldLogout.release();
    resumedResponse = await pending;
  } finally {
    heldLogout.release();
    app.db.withUser = original;
  }
  const afterResumed = effects();
  raw.cases.push({ label: 'logout-after-door-before-final-R362-query', timing: { entered: logoutEntered, logoutAt, released: logoutReleased },
    queryHits: heldLogout.hits(), logoutStatus: logoutResponse.statusCode, status: resumedResponse.statusCode,
    result: body(resumedResponse), before: beforeLogout, afterLogout, after: afterResumed,
    sessionBeforeLogout, sessionAfterLogout, providerAttempts: h.providerAttempts() });
  assert.equal(logoutResponse.statusCode, 204, logoutResponse.body);
  assert.equal(sessionBeforeLogout.revoked_at, null);
  assert.ok(sessionAfterLogout.revoked_at);
  assert.ok(logoutEntered <= logoutAt && logoutAt <= logoutReleased);
  assert.equal(resumedResponse.statusCode, 204, resumedResponse.body);
  assertSuspended(afterLogout, rival, true);
  assertSuspended(afterResumed, rival, false);
  exactEventDelta(afterLogout, afterResumed, 'company_reactivated', rival, alexId);
  await call('fresh-logged-out-cookie-reactivation-denied', 'POST', reactivate(rival), staff, undefined, 401, true);
  assert.equal(h.providerAttempts(), 0);
  raw.verdict = 'CURRENT-STAFF-REVOKE-BLOCKS-FINAL-R362; POST-DOOR-LOGOUT-REACTIVATES-FIND-033-VARIANT';
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: The staff member logs out after the database accepts the session for a request but before the protected platform operation executes.

Recorded result: A staff member logged out the exact MFA-authenticated session before pending protected SQL ran, yet the in-flight request completed platform reads and billing changes. New requests using the logged-out session were rejected.

Scope: Whether logout must cancel an already-started staff request remains a policy-dependent boundary. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/attack-led/authority-data/run-r362-reactivation-final-authority-2026-10-16.mjs:220-244` (synthetic test).

```javascript
    logoutAt = Date.now();
    afterLogout = effects();
    sessionAfterLogout = sessionRow(staff.token);
    logoutReleased = heldLogout.release();
    resumedResponse = await pending;
  } finally {
    heldLogout.release();
    app.db.withUser = original;
  }
  const afterResumed = effects();
  raw.cases.push({ label: 'logout-after-door-before-final-R362-query', timing: { entered: logoutEntered, logoutAt, released: logoutReleased },
    queryHits: heldLogout.hits(), logoutStatus: logoutResponse.statusCode, status: resumedResponse.statusCode,
    result: body(resumedResponse), before: beforeLogout, afterLogout, after: afterResumed,
    sessionBeforeLogout, sessionAfterLogout, providerAttempts: h.providerAttempts() });
  assert.equal(logoutResponse.statusCode, 204, logoutResponse.body);
  assert.equal(sessionBeforeLogout.revoked_at, null);
  assert.ok(sessionAfterLogout.revoked_at);
  assert.ok(logoutEntered <= logoutAt && logoutAt <= logoutReleased);
  assert.equal(resumedResponse.statusCode, 204, resumedResponse.body);
  assertSuspended(afterLogout, rival, true);
  assertSuspended(afterResumed, rival, false);
  exactEventDelta(afterLogout, afterResumed, 'company_reactivated', rival, alexId);
  await call('fresh-logged-out-cookie-reactivation-denied', 'POST', reactivate(rival), staff, undefined, 401, true);
  assert.equal(h.providerAttempts(), 0);
  raw.verdict = 'CURRENT-STAFF-REVOKE-BLOCKS-FINAL-R362; POST-DOOR-LOGOUT-REACTIVATES-FIND-033-VARIANT';
```

## Remediation

Recheck session validity at the sensitive SQL effect or serialize logout with in-flight platform operations.
