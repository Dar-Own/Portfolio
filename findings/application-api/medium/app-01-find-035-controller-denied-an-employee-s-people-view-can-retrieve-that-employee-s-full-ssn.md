# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Medium
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Controller denied an employee's People view can retrieve that employee's full SSN

## Description

A Controller can reveal the full SSN of a same-company employee whose People record the Controller cannot open.

Prerequisite and affected workflow: A Controller with `ssn.reveal`, their own password, and a colleague ID from a visible bill requests that colleague's SSN.

Established observation: In a fresh two-company HTTP test, a Controller obtained an employee ID from a permitted bill, was denied that employee's People record, then retrieved the employee's full synthetic SSN with their own password. A lesser role and the other company were denied, and the reveal generated audit records.

Limits: Disclosure of real employee data in a deployed build remains untested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A Controller with `ssn.reveal`, their own password, and a colleague ID from a visible bill requests that colleague's SSN.

Source: `packages/db/migrations/0005_owner_permissions.sql:66-70` (1d5d98db9acf).

```sql
   {"name":"Controller","rank":70,"desc":"Invoicing, payroll and company settings","grants":{
     "dash.view":{},"leads.view":{"scope":"all"},"costing.view":{},"invoices.view":{},"invoices.create":{},
     "invoices.payment":{},"payroll.view":{"scope":"all"},"payroll.hours":{"scope":"all"},"payroll.approve":{},
     "payroll.submit":{},"people.manage":{},"ssn.reveal":{},"audit.view":{},"settings.edit":{},"reports.view":{},"import.run":{},
     "paperwork.manage":{},"vendors.view":{},"vendors.manage":{},"bills.view":{},"bills.manage":{}}},
```

#### Step 2 — Cross the missing state or authority check

The SSN endpoint and database function check SSN permission, password, and company, but do not check whether the requester may view the employee.

Source: `apps/api/src/features/profile/routes.ts:169-173` (1d5d98db9acf).

```typescript

    app.get('/people/:userId', { schema: { params: userParams } }, async (request) => {
      requirePermission(request.session, 'users.manage');
      return app.db.withUser(request.session, (ctx) => readPersonContact(ctx, request.params.userId));
    });
```

Source: `apps/api/src/features/team/routes.ts:120-132` (1d5d98db9acf).

```typescript
  app.get('/people', async (request) => {
    requirePermission(request.session, 'users.manage');
    return app.db.withUser(request.session, async (ctx) => ({ people: await listPeople(ctx, request.session) }));
  });

  app.get('/options', async (request) => {
    requirePermission(request.session, 'users.manage');
    return app.db.withUser(request.session, (ctx) => teamOptions(ctx, request.session));
  });

  app.get('/people/:userId', { schema: { params: userParams } }, async (request) => {
    requirePermission(request.session, 'users.manage');
    return app.db.withUser(request.session, (ctx) => getPerson(ctx, request.session, request.params.userId));
```


#### Step 3 — Observe the durable effect

In a fresh two-company HTTP test, a Controller obtained an employee ID from a permitted bill, was denied that employee's People record, then retrieved the employee's full synthetic SSN with their own password. A lesser role and the other company were denied, and the reveal generated audit records.

Disclosure of real employee data in a deployed build remains untested.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/http-poc/run-ssn-reveal-visibility.mjs:199-219` (test source).

```javascript
  output.result = 'PROVED-LOCAL-HTTP-SYNTHETIC';
  output.summary = 'A Controller denied the target People view received the full synthetic SSN; the authorized visible owner worked; no-permission and foreign-tenant controls denied. Each successful reveal added one audit row and one event; denied requests changed no tracked contact/audit/event/message/notification state.';
  process.stdout.write(`${JSON.stringify({ result: output.result, resultPath, cases: output.cases.map(({ name, status }) => ({ name, status })) }, null, 2)}\n`);
} catch (error) {
  output.error = String(error?.stack ?? error);
  throw error;
} finally {
  try { if (app) await app.close(); } catch (error) { output.cleanupError = `app: ${String(error)}`; }
  let stopped = !started;
  try {
    if (started) {
      const pidNow = Number(readFileSync(path.join(data, 'postmaster.pid'), 'utf8').split('\n')[0]);
      assert.equal(pidNow, postmasterPid, 'private postmaster PID changed; refuse ambiguous cleanup');
      command(path.join(pgBin, 'pg_ctl'), ['-D', data, '-m', 'immediate', '-w', 'stop']);
      stopped = true;
    }
  } catch (error) { output.cleanupError = `PostgreSQL: ${String(error)}`; }
  mkdirSync(path.dirname(resultPath), { recursive: true });
  writeFileSync(resultPath, `${JSON.stringify(output, null, 2)}\n`, { mode: 0o600 });
  if (stopped && root.startsWith('/private/tmp/aos-ssn-visibility-')) rmSync(root, { recursive: true, force: true });
}
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A Controller with `ssn.reveal`, their own password, and a colleague ID from a visible bill requests that colleague's SSN.

Recorded result: In a fresh two-company HTTP test, a Controller obtained an employee ID from a permitted bill, was denied that employee's People record, then retrieved the employee's full synthetic SSN with their own password. A lesser role and the other company were denied, and the reveal generated audit records.

Scope: Disclosure of real employee data in a deployed build remains untested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/http-poc/run-ssn-reveal-visibility.mjs:199-219` (test source).

```javascript
  output.result = 'PROVED-LOCAL-HTTP-SYNTHETIC';
  output.summary = 'A Controller denied the target People view received the full synthetic SSN; the authorized visible owner worked; no-permission and foreign-tenant controls denied. Each successful reveal added one audit row and one event; denied requests changed no tracked contact/audit/event/message/notification state.';
  process.stdout.write(`${JSON.stringify({ result: output.result, resultPath, cases: output.cases.map(({ name, status }) => ({ name, status })) }, null, 2)}\n`);
} catch (error) {
  output.error = String(error?.stack ?? error);
  throw error;
} finally {
  try { if (app) await app.close(); } catch (error) { output.cleanupError = `app: ${String(error)}`; }
  let stopped = !started;
  try {
    if (started) {
      const pidNow = Number(readFileSync(path.join(data, 'postmaster.pid'), 'utf8').split('\n')[0]);
      assert.equal(pidNow, postmasterPid, 'private postmaster PID changed; refuse ambiguous cleanup');
      command(path.join(pgBin, 'pg_ctl'), ['-D', data, '-m', 'immediate', '-w', 'stop']);
      stopped = true;
    }
  } catch (error) { output.cleanupError = `PostgreSQL: ${String(error)}`; }
  mkdirSync(path.dirname(resultPath), { recursive: true });
  writeFileSync(resultPath, `${JSON.stringify(output, null, 2)}\n`, { mode: 0o600 });
  if (stopped && root.startsWith('/private/tmp/aos-ssn-visibility-')) rmSync(root, { recursive: true, force: true });
}
```

## Remediation

Check permission to view the specific employee in the database function, while retaining the password, company, and audit checks.
