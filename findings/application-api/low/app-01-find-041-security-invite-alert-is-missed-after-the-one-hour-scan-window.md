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

# Security invite alert is missed after the one-hour scan window

## Description

A valid security invite can miss its owner task and queued security notice when the first alert scan runs late.

Prerequisite and affected workflow: The next security-alert scan runs more than one hour after the invitation is recorded.

Established observation: A manager created a valid invite and its audit event. When the security-alert worker first ran 61 minutes later, it created no owner task or queued notice, although the invite remained visible in audit history.

Limits: An actual worker outage, deployed scheduler delay, and owner notice delivery remain untested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

The next security-alert scan runs more than one hour after the invitation is recorded.

Source: `apps/api/src/features/team/routes.ts:247-261` (1d5d98db9acf).

```typescript
  app.post('/invites', { schema: { body: inviteBody }, config: { permission: 'users.manage', dangerous: true } }, async (request, reply) => {
    const body = request.body;
    // The letter is the database's own and the worker sends it (migration 0106 stage 2d); a letter that
    // could not leave is refused here first, in plain words.
    await requireInviteMail(app, request.session);
    const invite = await app.db.withUser(request.session, (ctx) =>
      createInvite(ctx, request.session, {
        email: body.email,
        fullName: body.fullName,
        roleId: body.roleId,
        title: body.title ?? null,
        crewId: body.crewId ?? null,
      }),
    );
    return reply.status(201).send({ invite });
```

#### Step 2 — Cross the missing state or authority check

The security-alert worker searches only the previous hour and has no recovery pass for older unalerted events.

Source: `apps/api/src/features/team/invites.ts:105-135` (1d5d98db9acf).

```typescript
/**
 * Creates the invite; the database queues its letter in the outbox (migration 0106 stage 2d), and the
 * worker mints the one-time link as it sends. The door is handed a hash no token matches, so until the
 * letter leaves nothing opens the invite. Runs inside the caller's withUser transaction.
 */
export async function createInvite(ctx: DbContext, session: Session, input: CreateInviteInput): Promise<TeamInvite> {
  const problem = inviteProblem(input);
  if (problem) throw invalid(problem);
  const email = input.email.trim().toLowerCase();
  const role = (
    await ctx.client.query<{ name: string; rank: number }>('select name, rank from app.roles where tenant_id = $1 and id = $2 and not is_system', [ctx.tenantId, input.roleId])
  ).rows[0];
  if (!role) throw invalid("That role doesn't exist.");
  if (!outranks(session, role.rank)) throw ruleViolation(TEAM_TEXTS.inviteRankRule);

  let id: string;
  try {
    const door = input.vendorStaff === true ? 'auth.invite_vendor_staff' : 'auth.create_invite';
    const { rows } = await ctx.client.query<{ id: string }>(`select ${door}($1, $2, $3, $4, $5, $6) as id`, [
      email, input.fullName.trim(), input.roleId, input.title, input.crewId, pendingTokenHash(),
    ]);
    id = rows[0]!.id;
  } catch (error) {
    throw inviteRefusal(error);
  }

  const invites = await listInvites(ctx, session);
  const created = invites.find((invite) => invite.id === id);
  if (!created) throw notFound("The invite was created but couldn't be read back.");
  return created;
}
```

Source: `apps/worker/src/jobs/security-alerts.ts:109-135` (1d5d98db9acf).

```typescript
async function company(client: pg.PoolClient, tenantId: string): Promise<Company> {
  const { rows } = await client.query<{ name: string; business_tz: string | null; business_start: string | null; business_end: string | null; business_days: number[] | null; now: Date }>(
    `select t.name, s.business_tz, s.business_start::text as business_start, s.business_end::text as business_end, s.business_days, now() as now
       from app.tenants t left join app.tenant_settings s on s.tenant_id = t.id where t.id = $1`,
    [tenantId],
  );
  const row = rows[0];
  if (!row) throw new Error(`Company ${tenantId} is not visible to its Automation member.`);
  const owner = await client.query<{ id: string; full_name: string; email: string }>(
    `select u.id, u.full_name, u.email::text as email
       from app.memberships m
       join app.roles r on r.tenant_id = m.tenant_id and r.id = m.role_id and r.is_owner
       join app.users u on u.id = m.user_id and not u.is_system and u.disabled_at is null
      where m.tenant_id = $1 order by u.created_at, u.id limit 1`,
    [tenantId],
  );
  const person = owner.rows[0];
  if (!person) throw new Error('This company has no owner to alert.');
  // The company's first brand (the same choice the invite email and the reminders make).
  const brand = await client.query<{ sending_email: string | null }>('select sending_email::text as sending_email from app.business_units where tenant_id = $1 order by created_at, name, id limit 1', [tenantId]);
  return {
    name: row.name,
    hours: businessHoursFromSettings(row),
    owner: { id: person.id, firstName: person.full_name.split(/\s+/)[0] ?? person.full_name, email: person.email },
    sender: brand.rows[0]?.sending_email ?? '',
    now: row.now,
  };
```


#### Step 3 — Observe the durable effect

A manager created a valid invite and its audit event. When the security-alert worker first ran 61 minutes later, it created no owner task or queued notice, although the invite remained visible in audit history.

An actual worker outage, deployed scheduler delay, and owner notice delivery remain untested.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/security-alert-window/run-security-alert-window.mjs:180-203` (test source).

```javascript
  const history = await app.inject({ method: 'GET', url: '/api/team/security-events?limit=100', headers: owner.headers });
  assert.equal(history.statusCode, 200, history.body);
  assert.ok(history.body.includes('Synthetic Late Security Invite'), 'alternate owner Security history retains event');
  const foreignHistory = await app.inject({ method: 'GET', url: '/api/team/security-events?limit=100', headers: foreign.headers });
  assert.equal(foreignHistory.statusCode, 200, foreignHistory.body);
  assert.ok(!foreignHistory.body.includes('Synthetic Late Security Invite'));
  raw.cases.push({ name: 'first-run-61m-late', status: late.statusCode, inviteId: lateId, auditId: lateAudit.id,
    afterProducer: afterLateProducer, simulatedWorkerNow: simulatedWorkerNow.toISOString(), clockOverrides, workerVisible,
    job: lateRun, afterJob: afterLateJob, repeat, afterRepeat, ownerSecurityHistory: { status: history.statusCode, marker: true },
    foreignSecurityHistory: { status: foreignHistory.statusCode, marker: false } });
  assert.equal(afterRepeat.rivalSecurityTasks, 0);
  assert.equal(afterRepeat.rivalSecurityNotices, 0);
  assert.equal(networkAttempts, 0);
  raw.verdict = 'J22-TIMELY-ALERT-WORKS; HTTP-INVITE-EVENT-61M-LATE-SILENTLY-MISSED';
  raw.limits.push('The 61-minute delay is emulated by overriding only J22 company() PostgreSQL now() result on its test-only pool; the genuine HTTP-created audit event and its timestamp remain unchanged. No wall-clock wait or deployed outage occurred.');
  raw.limits.push('The owner can still find the event by opening Security history, but J22 created no owner task or queued security notice. No SMTP delivery or production scheduling was tested.');
} catch (error) {
  raw.error = error instanceof Error ? { message: error.message, stack: error.stack } : String(error);
} finally {
  if (app) await app.close();
  if (worker) await worker.close();
  if (started) {
    const pidFile = path.join(data, 'postmaster.pid');
    if (existsSync(pidFile) && Number(readFileSync(pidFile, 'utf8').split('\n')[0]) === postmasterPid) {
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: The next security-alert scan runs more than one hour after the invitation is recorded.

Recorded result: A manager created a valid invite and its audit event. When the security-alert worker first ran 61 minutes later, it created no owner task or queued notice, although the invite remained visible in audit history.

Scope: An actual worker outage, deployed scheduler delay, and owner notice delivery remain untested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/security-alert-window/run-security-alert-window.mjs:180-203` (test source).

```javascript
  const history = await app.inject({ method: 'GET', url: '/api/team/security-events?limit=100', headers: owner.headers });
  assert.equal(history.statusCode, 200, history.body);
  assert.ok(history.body.includes('Synthetic Late Security Invite'), 'alternate owner Security history retains event');
  const foreignHistory = await app.inject({ method: 'GET', url: '/api/team/security-events?limit=100', headers: foreign.headers });
  assert.equal(foreignHistory.statusCode, 200, foreignHistory.body);
  assert.ok(!foreignHistory.body.includes('Synthetic Late Security Invite'));
  raw.cases.push({ name: 'first-run-61m-late', status: late.statusCode, inviteId: lateId, auditId: lateAudit.id,
    afterProducer: afterLateProducer, simulatedWorkerNow: simulatedWorkerNow.toISOString(), clockOverrides, workerVisible,
    job: lateRun, afterJob: afterLateJob, repeat, afterRepeat, ownerSecurityHistory: { status: history.statusCode, marker: true },
    foreignSecurityHistory: { status: foreignHistory.statusCode, marker: false } });
  assert.equal(afterRepeat.rivalSecurityTasks, 0);
  assert.equal(afterRepeat.rivalSecurityNotices, 0);
  assert.equal(networkAttempts, 0);
  raw.verdict = 'J22-TIMELY-ALERT-WORKS; HTTP-INVITE-EVENT-61M-LATE-SILENTLY-MISSED';
  raw.limits.push('The 61-minute delay is emulated by overriding only J22 company() PostgreSQL now() result on its test-only pool; the genuine HTTP-created audit event and its timestamp remain unchanged. No wall-clock wait or deployed outage occurred.');
  raw.limits.push('The owner can still find the event by opening Security history, but J22 created no owner task or queued security notice. No SMTP delivery or production scheduling was tested.');
} catch (error) {
  raw.error = error instanceof Error ? { message: error.message, stack: error.stack } : String(error);
} finally {
  if (app) await app.close();
  if (worker) await worker.close();
  if (started) {
    const pidFile = path.join(data, 'postmaster.pid');
    if (existsSync(pidFile) && Number(readFileSync(pidFile, 'utf8').split('\n')[0]) === postmasterPid) {
```

## Remediation

Use a durable event cursor or catch-up watermark with event-ID deduplication so delayed scans process unalerted events.
