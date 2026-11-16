# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low (conditional integrity)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Expense decision can finalize a revised amount/description without current-version review

## Description

An expense decision can finalize a revised amount or description that the approver did not review. A versionless approval can add the revised amount to job cost.

Prerequisite and affected workflow: The submitter edits an expense after the approver views it, then the approver declines from the stale view or approves through the API without a version.

Established observation: A stale Decline finalized an expense after its submitter changed the amount or description. Separately, an approval request with no review version finalized a raised amount and added 9,800 cents to job cost; a supplied stale version was rejected.

Limits: Use of versionless third-party clients, cash payout, and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

The submitter edits an expense after the approver views it, then the approver declines from the stale view or approves through the API without a version.

Source: `apps/api/src/features/expenses/routes.ts:56-57` (1d5d98db9acf).

```typescript
// The updatedAt the approver was shown: the approval lands only on the expense as they saw it.
const approveBody = z.strictObject({ seenUpdatedAt: z.iso.datetime('Open the expense again, then approve it.').optional() }).optional();
```

#### Step 2 — Cross the missing state or authority check

The approval version is optional and Decline supplies no version, so the final decision can lack a current-review comparison.

Source: `apps/api/src/features/expenses/store.ts:325-356` (1d5d98db9acf).

```typescript
/**
 * Moves an expense. `seenUpdatedAt` (the updatedAt the person was shown) makes the move land only on the
 * expense as they saw it: a waiting expense can still be edited, and an approval must not pay out an
 * amount the approver never saw.
 */
async function move(ctx: DbContext, expenseId: string, status: ExpenseStatus, declineReason: string | null, seenUpdatedAt?: string): Promise<void> {
  if (!(EXPENSE_STATUSES as readonly string[]).includes(status)) throw invalid('Pick a status.');
  const expense = await requireExpense(ctx, expenseId);
  // Only a waiting expense changes under someone's eyes; for any other the guard's own words say why the move is refused.
  const seen = seenUpdatedAt !== undefined && expense.status === 'submitted' ? seenUpdatedAt : null;
  if (seen !== null && seen !== expense.updatedAt) throw conflict(CHANGED_SINCE_OPENED);
  const result = await ctx.client.query(
    `update app.expenses set status = $3, decline_reason = coalesce($4, decline_reason)
      where tenant_id = $1 and id = $2 and ($5::timestamptz is null or date_trunc('milliseconds', updated_at) = $5::timestamptz)`,
    [ctx.tenantId, expenseId, status, declineReason, seen],
  );
  if (result.rowCount === 0) {
    if (seen !== null) throw conflict(CHANGED_SINCE_OPENED);
    // The row is visible (requireExpense found it), so a miss means the update policy kept this person out.
    throw forbidden(NOT_YOURS_TO_CHANGE);
  }
}

export async function approveExpense(ctx: DbContext, expenseId: string, seenUpdatedAt?: string): Promise<void> {
  await move(ctx, expenseId, 'approved', null, seenUpdatedAt);
}

export async function declineExpense(ctx: DbContext, expenseId: string, reason: string): Promise<void> {
  const text = cleanLine(reason);
  if (!text) throw invalid(DECLINE_NEEDS_REASON);
  if (text.length > MAX_DECLINE_REASON_LENGTH) throw invalid(`Keep the reason under ${MAX_DECLINE_REASON_LENGTH} characters.`);
  await move(ctx, expenseId, 'declined', text);
```

Source: `packages/db/migrations/0053_expenses.sql:249-280` (1d5d98db9acf).

```sql
      elsif new.status in ('approved', 'declined') then
        if old.status <> 'submitted' then
          raise exception '%', already;
        end if;
        if not app.can('expenses.approve') then
          raise exception 'Only people who approve expenses can approve or decline one';
        end if;
        if old.submitted_by = me then
          raise exception 'You can''t approve or decline your own expense';
        end if;
        if new.status = 'declined' and new.decline_reason is null then
          raise exception 'Give a reason when declining an expense';
        end if;
        if new.status = 'approved' then
          new.decline_reason := null;
        end if;
        new.decided_by := me;
        new.decided_at := now();
        new.paid_back_by := null;
        new.paid_back_at := null;
      elsif new.status = 'paid_back' then
        if old.status = 'submitted' then
          raise exception 'An expense must be approved before it is marked paid back';
        end if;
        if old.status <> 'approved' then
          raise exception '%', already;
        end if;
        if not (app.can('expenses.approve') or app.can('bills.manage')) then
          raise exception 'Only people who approve expenses or manage bills can mark an expense paid back';
        end if;
        -- The person who gets the money never records that the company handed it over.
        if old.submitted_by = me then
```


#### Step 3 — Observe the durable effect

A stale Decline finalized an expense after its submitter changed the amount or description. Separately, an approval request with no review version finalized a raised amount and added 9,800 cents to job cost; a supplied stale version was rejected.

Use of versionless third-party clients, cash payout, and deployed effect remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/attack-led/authority-data/run-expense-stale-decline.mjs:130-153` (test source).

```javascript
  const decline = await send('stale-decline', 'POST', `/api/expenses/${active.id}/decline`, approver.headers,
    { reason: 'Old detail appears wrong' }, 200, [active.id]);
  const declined = decline.response.json().expense;
  assert.equal(declined.status, 'declined');
  assert.equal(declined.amountCents, 9800);
  assert.equal(declined.what, 'Synthetic revised medical equipment');
  assert.equal(declined.declineReason, 'Old detail appears wrong');
  assert.ok(afterEdit.expenses[0].updated_at !== staleSeen.updatedAt);
  assert.equal(sql(`select count(*) from app.audit_log where action='note' and entity='activity' and after->>'summary' like '%declined expense%' and after->>'summary' like '%Synthetic revised medical equipment%'`), '1');
  const normal = await send('fresh-decline-positive', 'POST', `/api/expenses/${ordinary.id}/decline`, approver.headers,
    { reason: 'Fresh current review' }, 200, [ordinary.id]);
  assert.equal(normal.response.json().expense.status, 'declined');
  assert.equal(raw.providerAttempts, 0);
  raw.verdict = 'PROVED-TECHNICAL-BEHAVIOR';
  raw.conclusion = 'A previously seen detail remained actionable after the submitter revised its amount and description; R149 rejected neither the stale decision nor its reason. R148 with seenUpdatedAt rejected the same stale version. Whether decline requires current-state review is an unconfirmed policy rule, so business-impact classification is conditional.';
  raw.businessImpact = 'CONDITIONAL: erroneous/unauthorized rejection of a revised reimbursement, and activity/audit carrying a decision reason based on an obsolete version. No provider or cash effect.';
  raw.clientDecision = 'Must an approver review the latest expense amount and description before declining it, as approval already requires?';
} catch (error) {
  raw.error = { name: error?.name ?? 'Error', message: String(error?.message ?? error), stack: String(error?.stack ?? '') };
  process.exitCode = 1;
} finally {
  try { await app?.close(); } catch (error) { raw.cleanupError = String(error); process.exitCode = 1; }
  if (started) {
    try { command(path.join(pgBin, 'pg_ctl'), ['-D', data, '-m', 'immediate', '-w', 'stop']); }
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: The submitter edits an expense after the approver views it, then the approver declines from the stale view or approves through the API without a version.

Recorded result: A stale Decline finalized an expense after its submitter changed the amount or description. Separately, an approval request with no review version finalized a raised amount and added 9,800 cents to job cost; a supplied stale version was rejected.

Scope: Use of versionless third-party clients, cash payout, and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/attack-led/authority-data/run-expense-stale-decline.mjs:130-153` (test source).

```javascript
  const decline = await send('stale-decline', 'POST', `/api/expenses/${active.id}/decline`, approver.headers,
    { reason: 'Old detail appears wrong' }, 200, [active.id]);
  const declined = decline.response.json().expense;
  assert.equal(declined.status, 'declined');
  assert.equal(declined.amountCents, 9800);
  assert.equal(declined.what, 'Synthetic revised medical equipment');
  assert.equal(declined.declineReason, 'Old detail appears wrong');
  assert.ok(afterEdit.expenses[0].updated_at !== staleSeen.updatedAt);
  assert.equal(sql(`select count(*) from app.audit_log where action='note' and entity='activity' and after->>'summary' like '%declined expense%' and after->>'summary' like '%Synthetic revised medical equipment%'`), '1');
  const normal = await send('fresh-decline-positive', 'POST', `/api/expenses/${ordinary.id}/decline`, approver.headers,
    { reason: 'Fresh current review' }, 200, [ordinary.id]);
  assert.equal(normal.response.json().expense.status, 'declined');
  assert.equal(raw.providerAttempts, 0);
  raw.verdict = 'PROVED-TECHNICAL-BEHAVIOR';
  raw.conclusion = 'A previously seen detail remained actionable after the submitter revised its amount and description; R149 rejected neither the stale decision nor its reason. R148 with seenUpdatedAt rejected the same stale version. Whether decline requires current-state review is an unconfirmed policy rule, so business-impact classification is conditional.';
  raw.businessImpact = 'CONDITIONAL: erroneous/unauthorized rejection of a revised reimbursement, and activity/audit carrying a decision reason based on an obsolete version. No provider or cash effect.';
  raw.clientDecision = 'Must an approver review the latest expense amount and description before declining it, as approval already requires?';
} catch (error) {
  raw.error = { name: error?.name ?? 'Error', message: String(error?.message ?? error), stack: String(error?.stack ?? '') };
  process.exitCode = 1;
} finally {
  try { await app?.close(); } catch (error) { raw.cleanupError = String(error); process.exitCode = 1; }
  if (started) {
    try { command(path.join(pgBin, 'pg_ctl'), ['-D', data, '-m', 'immediate', '-w', 'stop']); }
```

## Remediation

Require the current reviewed revision for both approval and decline, enforcing it atomically with status and cost writes.
