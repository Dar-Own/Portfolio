# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: High (same-tenant confidentiality/authority)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Manager can delegate unheld Controller authority to reveal employee SSNs

## Description

A manager can confer unheld Controller authority on another member, who can then reveal full same-company employee SSNs with their own password.

Prerequisite and affected workflow: A stepped-up `users.manage` manager lacking `ssn.reveal` assigns Controller or removes an SSN deny from a lower-ranked member.

Established observation: A manager lacking SSN reveal authority assigned a lower-ranked Controller role. After a fresh login, that member retrieved two full synthetic employee SSNs; removing an existing Controller's SSN deny produced the same result. Pre-delegation and foreign-company requests were denied, and successful reveals created audit and event records.

Limits: Self-escalation, disclosure of real employee data, and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A stepped-up `users.manage` manager lacking `ssn.reveal` assigns Controller or removes an SSN deny from a lower-ranked member.

Source: `packages/db/migrations/0001_initial_schema.sql:304-384` (1d5d98db9acf).

```sql
  select * into me from app.my_membership();
  if me.role_id is null then raise exception 'You are not a member of this company'; end if;

  -- who is being changed?
  if tg_table_name = 'roles' then
    if not app.can('roles.manage') then raise exception 'You don''t have permission to manage roles'; end if;
  elsif tg_table_name = 'role_permissions' then
    if not app.can('roles.manage') then raise exception 'You don''t have permission to manage roles'; end if;
  else
    if not app.can('users.manage') then raise exception 'You don''t have permission to manage users'; end if;
  end if;

  if tg_op in ('UPDATE','DELETE') then
    if tg_table_name = 'roles' then
      target_rank := old.rank;
      if old.is_owner then raise exception 'The owner role can''t be changed'; end if;
    elsif tg_table_name = 'role_permissions' then
      select rank into target_rank from app.roles where tenant_id = old.tenant_id and id = old.role_id;
    else
      if old.user_id = app.current_user_id() then raise exception 'You can''t change your own access'; end if;
      select r.rank into target_rank from app.memberships m join app.roles r on r.tenant_id = m.tenant_id and r.id = m.role_id
       where m.tenant_id = old.tenant_id and m.user_id = old.user_id;
    end if;
    if not me.is_owner and target_rank >= me.rank then
      raise exception 'You can only manage roles and people ranked below you';
    end if;
  end if;

  if tg_op = 'DELETE' then return old; end if;

  if tg_table_name = 'roles' then
    if new.is_owner and not me.is_owner then raise exception 'Only owners can create owner roles'; end if;
    if not me.is_owner and new.rank >= me.rank then
      raise exception 'Role rank must be below your own (%)', me.rank;
    end if;
    return new;
  end if;

  if tg_table_name in ('role_permissions') then
    target_role := new.role_id;
  else
    if new.user_id = app.current_user_id() then raise exception 'You can''t change your own access'; end if;
    if tg_table_name = 'memberships' then
      target_role := new.role_id;
    else
      select m.role_id into target_role from app.memberships m
       where m.tenant_id = new.tenant_id and m.user_id = new.user_id;
    end if;
  end if;
  select rank into target_rank from app.roles where tenant_id = new.tenant_id and id = target_role;
  if not me.is_owner and target_rank >= me.rank then
    raise exception 'You can only assign or change roles ranked below yours';
  end if;

  if tg_table_name = 'memberships' then return new; end if;

  -- No escalation: hand out only what you hold, no wider scope, no higher limit.
  if tg_table_name = 'role_permissions' then
    is_grant := true; eff_scope := new.scope; eff_limit := new.limit_value;
  else
    is_grant := new.effect = 'allow';
    if is_grant then
      select * into role_grant from app.role_permissions x
       where x.tenant_id = new.tenant_id and x.role_id = target_role and x.permission = new.permission;
      eff_scope := coalesce(new.scope, role_grant.scope);
      eff_limit := coalesce(new.limit_value, role_grant.limit_value);
    end if;
  end if;

  if is_grant and not me.is_owner then
    select * into ep from app.effective_permission(new.permission);
    if not ep.allowed then
      raise exception 'You can''t grant % because you don''t have it', new.permission;
    end if;
    if app.scope_rank(eff_scope) > app.scope_rank(ep.scope) then
      raise exception 'You can''t grant a wider scope than your own for %', new.permission;
    end if;
    if ep.limit_value is not null and (eff_limit is null or eff_limit > ep.limit_value) then
      raise exception 'You can''t grant a limit above your own (%) for %', ep.limit_value, new.permission;
    end if;
  end if;
```

#### Step 2 — Cross the missing state or authority check

The database access-change check permits role assignment and permission-deny removal without confirming that the manager holds the authority being granted.

Source: `apps/api/src/features/team/routes.ts:135-170` (1d5d98db9acf).

```typescript
  // Someone's role, their crew (crew-scope permissions decide what they see) and the payroll record they
  // are linked to are dangerous; their job title isn't.
  app.patch('/people/:userId', { schema: { params: userParams, body: personPatch }, config: { permission: 'users.manage', dangerous: whenBodyHas('roleId', 'crewId', 'employeeId') } }, async (request) => {
    return app.db.withUser(request.session, (ctx) => updatePerson(ctx, request.session, request.params.userId, request.body));
  });

  app.delete('/people/:userId', { schema: { params: userParams }, config: { permission: 'users.manage', dangerous: true } }, async (request, reply) => {
    const pictureFile = await app.db.withUser(request.session, (ctx) => removePerson(ctx, request.session, request.params.userId));
    // After the commit: the picture their profile carried goes from the disk too.
    await app.fileStore.forget(request.session, [pictureFile]);
    return reply.status(204).send();
  });

  app.post('/people/:userId/two-factor/reset', { schema: { params: userParams }, config: { permission: 'users.manage', dangerous: true } }, async (request) => {
    return app.db.withUser(request.session, (ctx) =>
      resetTwoFactor(ctx, request.session, request.params.userId, { ip: request.ip, userAgent: request.headers['user-agent'] }),
    );
  });

  app.post('/people/:userId/exceptions', { schema: { params: userParams, body: exceptionBody }, config: { permission: 'users.manage', dangerous: true } }, async (request, reply) => {
    const body = request.body;
    const detail = await app.db.withUser(request.session, (ctx) =>
      addException(ctx, request.session, request.params.userId, {
        permission: body.permission,
        effect: body.effect,
        scope: body.scope ?? null,
        limit: body.limit ?? null,
        reason: body.reason,
      }),
    );
    return reply.status(201).send(detail);
  });

  app.delete('/people/:userId/exceptions/:permission', { schema: { params: exceptionParams }, config: { permission: 'users.manage', dangerous: true } }, async (request) => {
    return app.db.withUser(request.session, (ctx) => removeException(ctx, request.session, request.params.userId, request.params.permission));
  });
```

Source: `apps/api/src/features/team/people.ts:188-218` (1d5d98db9acf).

```typescript
export async function updatePerson(ctx: DbContext, session: Session, userId: string, patch: PersonPatch): Promise<TeamPersonDetail> {
  const row = await personRow(ctx, userId);
  if (!row) throw notFound(PERSON_NOT_FOUND);
  requireManageable(session, row);

  const sets: string[] = [];
  const values: unknown[] = [ctx.tenantId, userId];
  const set = (column: string, value: unknown): void => {
    values.push(value);
    sets.push(`${column} = $${values.length}`);
  };
  if (patch.roleId !== undefined) {
    const { rows } = await ctx.client.query<{ rank: number }>(
      'select rank from app.roles where tenant_id = $1 and id = $2 and not is_system',
      [ctx.tenantId, patch.roleId],
    );
    const role = rows[0];
    if (!role) throw invalid("That role doesn't exist.");
    if (!outranks(session, role.rank)) throw ruleViolation('You can only assign roles ranked below yours');
    set('role_id', patch.roleId);
  }
  if (patch.title !== undefined) set('title', patch.title?.trim() || null);
  if (patch.crewId !== undefined) set('crew_id', patch.crewId);
  if (patch.employeeId !== undefined) set('employee_id', patch.employeeId);
  if (sets.length > 0) {
    await withDemoMessages(session, async () => {
      const result = await ctx.client.query(`update app.memberships set ${sets.join(', ')} where tenant_id = $1 and user_id = $2`, values);
      if (result.rowCount !== 1) throw notFound(PERSON_NOT_FOUND);
    });
  }
  return getPerson(ctx, session, userId);
```

Source: `apps/api/src/features/profile/routes.ts:175-194` (1d5d98db9acf).

```typescript
    app.post('/people/:userId/ssn/reveal', { schema: { params: userParams, body: revealBody } }, async (request): Promise<SsnReveal> => {
      requirePermission(request.session, 'ssn.reveal');
      const session = request.session;
      const masterKey = app.config.secretsKey;
      if (!masterKey) throw new HttpError(409, 'no_master_key', MASTER_KEY_MISSING_MESSAGE);
      // The password again, checked like a sign-in (guesses count toward the lockout).
      const check = await confirmPassword(app.db, session, request.body.password, { ip: request.ip, userAgent: request.headers['user-agent'] });
      if (check === 'locked') throw locked();
      if (check !== 'ok') throw new HttpError(422, 'wrong_password', WRONG_PASSWORD_MESSAGE);

      const userId = request.params.userId;
      // The database writes "ssn_revealed" to the history log, records ssn.revealed (its own event, with
      // who revealed: the worker's handler in events.ts tells the company's owners) and hands out the
      // ciphertext; that commits before anything else, so an attempt that reached the number is logged
      // and told even when it can't be read.
      const { ciphertext } = await app.db.withUser(session, (ctx) => revealSsnCiphertext(ctx, userId));
      request.log.info({ of: userId }, 'A Social Security number was revealed');
      const digits = decryptSecret(masterKey, userId, 'ssn', ciphertext);
      if (!digits) throw new HttpError(409, 'unreadable', "This number was saved under another master key and can't be read. Ask the person to enter it again.");
      return { ssn: formatSsn(digits), showForSeconds: SSN_SHOW_SECONDS };
```

Source: `packages/db/migrations/0106_automation_leaves_the_api.sql:701-725` (1d5d98db9acf).

```sql
create or replace function app.reveal_ssn(p_user uuid) returns bytea
language plpgsql security definer set search_path = app, pg_temp as $$
declare
  row_contact app.member_contacts;
  person app.users;
begin
  if app.current_user_id() is null or not app.is_member() then
    raise exception 'You are not a member of this company';
  end if;
  if not app.can('ssn.reveal') then
    raise exception 'You don''t have permission to view full Social Security numbers';
  end if;
  if p_user = app.current_user_id() then
    raise exception 'You can''t reveal your own Social Security number. Enter it again if it needs a change';
  end if;
  select * into row_contact from app.member_contacts c where c.tenant_id = app.current_tenant() and c.user_id = p_user;
  if not found or row_contact.ssn_ciphertext is null then
    raise exception 'No Social Security number is on file for this person';
  end if;
  select * into person from app.users u where u.id = p_user;
  insert into app.audit_log (tenant_id, actor_id, action, entity, entity_id, after)
  values (app.current_tenant(), app.current_user_id(), 'ssn_revealed', 'member_contacts', p_user::text,
          jsonb_build_object('user_id', p_user, 'full_name', person.full_name));
  perform app._emit_system_event(app.current_tenant(), 'ssn.revealed', p_user, app.current_user_id());
  return row_contact.ssn_ciphertext;
```


#### Step 3 — Observe the durable effect

A manager lacking SSN reveal authority assigned a lower-ranked Controller role. After a fresh login, that member retrieved two full synthetic employee SSNs; removing an existing Controller's SSN deny produced the same result. Pre-delegation and foreign-company requests were denied, and successful reveals created audit and event records.

Self-escalation, disclosure of real employee data, and deployed effect remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/attack-led/authority-data/run-team-ssn-delegation-2026-10-17.mjs:190-213` (synthetic test).

```javascript
  denyNoEffects(await call('lesser-cannot-assign', 'PATCH', `/api/team/people/${target.session.user.id}`, lesser, { roleId: controllerRole.id }, people, marker), 403);
  denyNoEffects(await call('manager-cannot-assign-peer-role', 'PATCH', `/api/team/people/${target.session.user.id}`, manager, { roleId: peerRole.id }, people, marker), 422);
  denyNoEffects(await call('manager-cannot-assign-foreign-user', 'PATCH', `/api/team/people/${foreign.session.user.id}`, manager, { roleId: controllerRole.id }, people, marker), 404);
  const assigned = await call('manager-assigns-unheld-powerful-role', 'PATCH', `/api/team/people/${target.session.user.id}`, manager, { roleId: controllerRole.id }, people, marker);
  assert.equal(assigned.response.statusCode, 200, assigned.response.body);
  assert.equal(assigned.response.json().effective['bills.manage']?.allowed, true);
  const targetSession = await call('target-resumes-expanded-session', 'GET', '/api/auth/session', target, undefined, people, marker);
  assert.equal(targetSession.response.statusCode, 200, targetSession.response.body);
  assert.equal(targetSession.response.json().permissions['bills.manage']?.allowed, true);
  const elevated = await login(app, '[redacted email]', TEST_DEMO_PASSWORD);
  assert.equal(elevated.session.permissions['ssn.reveal']?.allowed, true);
  const targetSsnEmployee = await call('fresh-target-reveals-employee-ssn-after-role-assignment', 'POST', employeeReveal, elevated, { password: TEST_DEMO_PASSWORD }, people, marker);
  assert.equal(targetSsnEmployee.response.statusCode, 200, targetSsnEmployee.response.body);
  assert.equal(targetSsnEmployee.response.json().ssn, employeeSsn);
  assert.equal(Number(targetSsnEmployee.after.audit)-Number(targetSsnEmployee.before.audit), 1);
  assert.equal(Number(targetSsnEmployee.after.events)-Number(targetSsnEmployee.before.events), 1);
  const targetSsnManager = await call('fresh-target-reveals-manager-ssn-after-role-assignment', 'POST', managerReveal, elevated, { password: TEST_DEMO_PASSWORD }, people, marker);
  assert.equal(targetSsnManager.response.statusCode, 200, targetSsnManager.response.body);
  assert.equal(targetSsnManager.response.json().ssn, managerSsn);
  const targetSsnForeign = await call('fresh-target-foreign-ssn-denied', 'POST', foreignReveal, elevated, { password: TEST_DEMO_PASSWORD }, people, marker);
  assert.notEqual(targetSsnForeign.response.statusCode, 200);
  assert.deepEqual(targetSsnForeign.after, targetSsnForeign.before);
  assert.ok(!JSON.stringify(parsed(targetSsnForeign.response)).includes(foreignSsn));
  const targetWrite = await call('target-creates-bill-after-role-assignment', 'POST', '/api/bills', target, billBody(`${marker}-ROLE-EFFECT`), people, marker);
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A stepped-up `users.manage` manager lacking `ssn.reveal` assigns Controller or removes an SSN deny from a lower-ranked member.

Recorded result: A manager lacking SSN reveal authority assigned a lower-ranked Controller role. After a fresh login, that member retrieved two full synthetic employee SSNs; removing an existing Controller's SSN deny produced the same result. Pre-delegation and foreign-company requests were denied, and successful reveals created audit and event records.

Scope: Self-escalation, disclosure of real employee data, and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/attack-led/authority-data/run-team-ssn-delegation-2026-10-17.mjs:190-213` (synthetic test).

```javascript
  denyNoEffects(await call('lesser-cannot-assign', 'PATCH', `/api/team/people/${target.session.user.id}`, lesser, { roleId: controllerRole.id }, people, marker), 403);
  denyNoEffects(await call('manager-cannot-assign-peer-role', 'PATCH', `/api/team/people/${target.session.user.id}`, manager, { roleId: peerRole.id }, people, marker), 422);
  denyNoEffects(await call('manager-cannot-assign-foreign-user', 'PATCH', `/api/team/people/${foreign.session.user.id}`, manager, { roleId: controllerRole.id }, people, marker), 404);
  const assigned = await call('manager-assigns-unheld-powerful-role', 'PATCH', `/api/team/people/${target.session.user.id}`, manager, { roleId: controllerRole.id }, people, marker);
  assert.equal(assigned.response.statusCode, 200, assigned.response.body);
  assert.equal(assigned.response.json().effective['bills.manage']?.allowed, true);
  const targetSession = await call('target-resumes-expanded-session', 'GET', '/api/auth/session', target, undefined, people, marker);
  assert.equal(targetSession.response.statusCode, 200, targetSession.response.body);
  assert.equal(targetSession.response.json().permissions['bills.manage']?.allowed, true);
  const elevated = await login(app, '[redacted email]', TEST_DEMO_PASSWORD);
  assert.equal(elevated.session.permissions['ssn.reveal']?.allowed, true);
  const targetSsnEmployee = await call('fresh-target-reveals-employee-ssn-after-role-assignment', 'POST', employeeReveal, elevated, { password: TEST_DEMO_PASSWORD }, people, marker);
  assert.equal(targetSsnEmployee.response.statusCode, 200, targetSsnEmployee.response.body);
  assert.equal(targetSsnEmployee.response.json().ssn, employeeSsn);
  assert.equal(Number(targetSsnEmployee.after.audit)-Number(targetSsnEmployee.before.audit), 1);
  assert.equal(Number(targetSsnEmployee.after.events)-Number(targetSsnEmployee.before.events), 1);
  const targetSsnManager = await call('fresh-target-reveals-manager-ssn-after-role-assignment', 'POST', managerReveal, elevated, { password: TEST_DEMO_PASSWORD }, people, marker);
  assert.equal(targetSsnManager.response.statusCode, 200, targetSsnManager.response.body);
  assert.equal(targetSsnManager.response.json().ssn, managerSsn);
  const targetSsnForeign = await call('fresh-target-foreign-ssn-denied', 'POST', foreignReveal, elevated, { password: TEST_DEMO_PASSWORD }, people, marker);
  assert.notEqual(targetSsnForeign.response.statusCode, 200);
  assert.deepEqual(targetSsnForeign.after, targetSsnForeign.before);
  assert.ok(!JSON.stringify(parsed(targetSsnForeign.response)).includes(foreignSsn));
  const targetWrite = await call('target-creates-bill-after-role-assignment', 'POST', '/api/bills', target, billBody(`${marker}-ROLE-EFFECT`), people, marker);
```

## Remediation

Require the manager to hold each effective permission and scope they grant, including permissions restored by removing a deny.
