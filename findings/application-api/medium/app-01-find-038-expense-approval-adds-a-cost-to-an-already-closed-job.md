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

# Expense approval adds a cost to an already closed job

## Description

Expense approval can add a cost to a closed job, changing its supposedly final ledger and margin basis.

Prerequisite and affected workflow: An approval passes its open-job check, then authorized finance staff close the job before the approval inserts its cost.

Established observation: A job closed with three cost rows while an earlier expense approval waited. The approval then succeeded, added a fourth cost of 4,250 cents, and left the job closed, demonstrating that the final cost write can pass after closure.

Limits: The race's natural frequency and deployed financial effect remain unmeasured. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

An approval passes its open-job check, then authorized finance staff close the job before the approval inserts its cost.

Source: `packages/db/migrations/0031_money_transactions.sql:11-19` (1d5d98db9acf).

```sql
--  2. app.transactions: the income and cost lines of a job. Entered by hand, imported from a
--     QuickBooks Online CSV export, or synced from QuickBooks once the Intuit app exists. Imported
--     lines carry external_id, unique per company and source, so an import never doubles a line.
--     Once a job is closed its transactions are frozen: no inserts, updates or deletes (guard trigger).
--  3. The close rule, in app.guard_job_status (0020, 0027, every rule kept): "closed" is a final
--     step after "completed" only, taken by someone holding money.manage, and refused unless the job
--     has at least one income and one cost transaction:
--       Can't close this job: add its income and cost transactions first
--     A closed job's status never changes again through the app (admin sessions can still fix data).
```

#### Step 2 — Cross the missing state or authority check

The expense approval's open-job check and cost insert are not serialized with job closure.

Source: `apps/api/src/features/expenses/routes.ts:198-206` (1d5d98db9acf).

```typescript
    app.post('/:id/approve', { schema: { params: idParams, body: approveBody } }, async (request) => {
      requirePermission(request.session, 'expenses.approve');
      const { id } = request.params;
      const seenUpdatedAt = request.body?.seenUpdatedAt;
      return app.db.withUser(request.session, async (ctx) => {
        await approveExpense(ctx, id, seenUpdatedAt);
        const expense = await requireExpense(ctx, id);
        await noteDecision(ctx, expense, request.session.user.fullName);
        return { expense };
```

Source: `apps/api/src/features/money/routes.ts:189-193` (1d5d98db9acf).

```typescript
    app.post('/jobs/:jobId/close', { schema: { params: jobParams } }, async (request) => {
      requirePermission(request.session, 'money.manage');
      const { jobId } = request.params;
      return app.db.withUser(request.session, async (ctx) => ({ job: await closeJob(ctx, jobId) }));
    });
```


#### Step 3 — Observe the durable effect

A job closed with three cost rows while an earlier expense approval waited. The approval then succeeded, added a fourth cost of 4,250 cents, and left the job closed, demonstrating that the final cost write can pass after closure.

The race's natural frequency and deployed financial effect remain unmeasured.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/expense-boundary/run-expense-close-race.mjs:188-210` (test source).

```javascript
} catch(error) {
  out.error=String(error?.stack??error);
  process.stderr.write(`${out.error}\n`);
  process.exitCode=1;
} finally {
  try {if(lockHeld && blocker) await blocker.query('ROLLBACK');} catch(error) {out.cleanupError=`blocker rollback: ${String(error)}`;}
  try {if(blocker) await blocker.end();} catch(error) {out.cleanupError=`blocker close: ${String(error)}`;}
  try {if(app) await app.close();} catch(error) {out.cleanupError=`app: ${String(error)}`;}
  let stopped=!started;
  try {
    if(started) {
      const pidNow=Number(readFileSync(path.join(data,'postmaster.pid'),'utf8').split('\n')[0]);
      assert.equal(pidNow,postmasterPid,'private postmaster PID changed; refuse ambiguous cleanup');
      command(path.join(pgBin,'pg_ctl'),['-D',data,'-m','immediate','-w','stop']); stopped=true;
    }
  } catch(error) {out.cleanupError=`PostgreSQL: ${String(error)}`;}
  out.cleanup={privateServerStopped:stopped,scratchRemoved:false};
  writeFileSync(rawPath,`${JSON.stringify(out,null,2)}\n`,{mode:0o600});
  if(stopped && root.startsWith('/private/tmp/aos-expense-close-')) {
    rmSync(root,{recursive:true,force:true}); out.cleanup.scratchRemoved=true;
    writeFileSync(rawPath,`${JSON.stringify(out,null,2)}\n`,{mode:0o600});
  }
}
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: An approval passes its open-job check, then authorized finance staff close the job before the approval inserts its cost.

Recorded result: A job closed with three cost rows while an earlier expense approval waited. The approval then succeeded, added a fourth cost of 4,250 cents, and left the job closed, demonstrating that the final cost write can pass after closure.

Scope: The race's natural frequency and deployed financial effect remain unmeasured. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/expense-boundary/run-expense-close-race.mjs:188-210` (test source).

```javascript
} catch(error) {
  out.error=String(error?.stack??error);
  process.stderr.write(`${out.error}\n`);
  process.exitCode=1;
} finally {
  try {if(lockHeld && blocker) await blocker.query('ROLLBACK');} catch(error) {out.cleanupError=`blocker rollback: ${String(error)}`;}
  try {if(blocker) await blocker.end();} catch(error) {out.cleanupError=`blocker close: ${String(error)}`;}
  try {if(app) await app.close();} catch(error) {out.cleanupError=`app: ${String(error)}`;}
  let stopped=!started;
  try {
    if(started) {
      const pidNow=Number(readFileSync(path.join(data,'postmaster.pid'),'utf8').split('\n')[0]);
      assert.equal(pidNow,postmasterPid,'private postmaster PID changed; refuse ambiguous cleanup');
      command(path.join(pgBin,'pg_ctl'),['-D',data,'-m','immediate','-w','stop']); stopped=true;
    }
  } catch(error) {out.cleanupError=`PostgreSQL: ${String(error)}`;}
  out.cleanup={privateServerStopped:stopped,scratchRemoved:false};
  writeFileSync(rawPath,`${JSON.stringify(out,null,2)}\n`,{mode:0o600});
  if(stopped && root.startsWith('/private/tmp/aos-expense-close-')) {
    rmSync(root,{recursive:true,force:true}); out.cleanup.scratchRemoved=true;
    writeFileSync(rawPath,`${JSON.stringify(out,null,2)}\n`,{mode:0o600});
  }
}
```

## Remediation

Serialize the approval cost write with job closure and enforce a final closed-job guard on the inserted cost.
