# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Restricted expense or support-ticket details appear in company activity to members denied the source object

## Description

Members denied an expense or support ticket can still read its description, customer, or subject through company activity feeds.

Prerequisite and affected workflow: A same-company member has dashboard or audit-feed access while lacking permission to open the source expense or ticket.

Established observation: A member denied direct access to a synthetic expense or support ticket still received its protected description, customer, or subject through the company activity feed. The direct object denial and feed disclosure were exercised through HTTP.

Limits: The local proofs used synthetic descriptions and customer names. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A same-company member has dashboard or audit-feed access while lacking permission to open the source expense or ticket.

Source: `apps/api/src/features/expenses/store.ts:252-256` (1d5d98db9acf).

```typescript
/** The Dashboard activity line for a new expense, written as the submitter (no amount: the feed is company-wide). */
export async function noteSubmitted(ctx: DbContext, expense: ExpenseRecord): Promise<void> {
  const who = expense.submittedBy?.fullName ?? 'Someone';
  const where = expense.job ? ` for job ${expense.job.number}` : '';
  await ctx.client.query('select app.add_activity_note($1, null)', [`${who} submitted expense ${expense.number}${where}: ${expense.what}`]);
```

#### Step 2 — Cross the missing state or authority check

Expense submission copies the submitter, expense number, optional job number and `expense.what` into a null-entity activity note (`apps/api/src/features/expenses/store.ts:252-256`). The expense SELECT policy permits approvers, bill managers or the original submitter, not every member (`packages/db/migrations/0053_expenses.sql:365-385`). The separate audit/activity policy admits the null-entity note to broader role permissions (`packages/db/migrations/0087_audit_log_denies_by_default.sql:147-163`), and dashboard and team audit display it (`apps/api/src/features/dashboard/dashboard.ts:278-295`; `apps/api/src/features/team/audit.ts:468-489`). The source comment calling the feed “company-wide” was the strongest alternative explanation; [redacted name] has now rejected that sharing rule for these details.

Source: `packages/db/migrations/0053_expenses.sql:365-385` (1d5d98db9acf).

```sql
alter table app.expenses enable row level security;
alter table app.expenses force row level security;
create policy tenant_isolation on app.expenses
  using (tenant_id = app.current_tenant()) with check (tenant_id = app.current_tenant());
create policy member_only on app.expenses as restrictive to app_user
  using ((select app.is_member())) with check ((select app.is_member()));
-- expenses.approve (and bills.manage, for the pay-back) sees every expense; expenses.submit sees your own.
create policy perm_select on app.expenses as restrictive for select to app_user
  using ((select app.can('expenses.approve') or app.can('bills.manage'))
         or ((select app.can('expenses.submit')) and submitted_by = app.current_user_id()));
create policy perm_insert on app.expenses as restrictive for insert to app_user
  with check ((select app.can('expenses.submit') or app.can('expenses.approve')));
-- Approvers and bills.manage holders may update any expense (the guard says what); a submitter only their own.
create policy perm_update on app.expenses as restrictive for update to app_user
  using ((select app.can('expenses.approve') or app.can('bills.manage')) or submitted_by = app.current_user_id())
  with check ((select app.can('expenses.approve') or app.can('bills.manage')) or submitted_by = app.current_user_id());

-- Explicit grants. No DELETE: a declined expense stays on record. The number, the submitter, the
-- decision, the pay-back and the cost line are written by the database only.
grant select on app.expenses to app_user;
grant insert (id, tenant_id, job_id, amount_cents, what, spent_on, receipt_file_id) on app.expenses to app_user;
```

Source: `packages/db/migrations/0087_audit_log_denies_by_default.sql:147-163` (1d5d98db9acf).

```sql
           -- A note against a lead is that lead's. A note with no lead is the company's activity feed,
           -- so this arm names the permissions that actually write one: the expense form and its
           -- approval (apps/api/src/features/expenses/store.ts), the incident form
           -- (features/incidents/store.ts), the importer (features/import/routes.ts), a support event
           -- (features/support/events.ts and features/intake/support.ts) and the deletion of a call
           -- (app.log_call_deletion, 0079). Behind "see call recordings" alone, which is where this arm
           -- started, "Andre reported INC-5" was gated on a permission about recordings and the
           -- Dashboard's feed emptied for a role holding only audit.view.
           -- The full-scope leads.view clause keeps a note readable after its lead is deleted; that
           -- reader already sees every lead, so it opens nothing about a lead that still exists.
           when entity in ('activity') then
                exists (select 1 from app.leads l where l.tenant_id = audit_log.tenant_id and l.id = nullif(entity_id, '')::uuid)
                or (nullif(entity_id, '') is not null and (select app.has_full_scope('leads.view')))
                or (nullif(entity_id, '') is null
                    and (select app.can('calls.view') or app.can('calls.manage') or app.can('expenses.approve')
                                or app.can('bills.manage') or app.can('incidents.manage') or app.can('incidents.report')
                                or app.can('support.view') or app.can('support.manage') or app.can('import.run')))
```


#### Step 3 — Observe the durable effect

A member denied direct access to a synthetic expense or support ticket still received its protected description, customer, or subject through the company activity feed. The direct object denial and feed disclosure were exercised through HTTP.

The local proofs used synthetic descriptions and customer names.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/attack-led/workers-bulk/run-support-event-reader.mjs:151-174` (test source).

```javascript
  assert.equal(miguel.session.permissions['dash.view']?.allowed, true);
  assert.equal(miguel.session.permissions['support.view']?.scope, 'crew');
  await send('foreman-still-hidden-after-grants', 'GET', detailPath, miguel.headers, undefined, 404);
  const pings = await send('foreman-alert-read', 'GET', '/api/tasks/pings?limit=50', miguel.headers, undefined, 200);
  const matchingPings = pings.json().items.filter((entry) => entry.body.includes(marker));
  const dashboard = await send('foreman-dashboard-read', 'GET', '/api/dashboard', miguel.headers, undefined, 200);
  const matchingActivity = (dashboard.json().activity ?? []).filter((entry) => entry.text.includes(marker));
  const foreignPings = await send('foreign-alert-control', 'GET', '/api/tasks/pings?limit=50', bob.headers, undefined, 200);
  const foreignDashboard = await send('foreign-dashboard-control', 'GET', '/api/dashboard', bob.headers, undefined, 200);
  assert.ok(!foreignPings.json().items.some((entry) => entry.body.includes(marker)), 'foreign alert leak');
  assert.ok(!(foreignDashboard.json().activity ?? []).some((entry) => entry.text.includes(marker)), 'foreign activity leak');
  raw.readerEffects = { matchingPings, matchingActivity, foreignPingMatches: 0, foreignActivityMatches: 0 };
  assert.equal(raw.providerAttempts, 0, 'unexpected external provider request');
  raw.verdict = matchingPings.length || matchingActivity.length ? 'PROVED-TECHNICAL-BEHAVIOR' : 'FALSIFIED-IN-TESTED-SCOPE';
  raw.conclusion = 'The foreman retained only crew-scoped support.view and received 404 for the ticket. The actual alert and activity reader results above determine whether broader task/audit permissions exposed customer or subject metadata.';
} catch (error) {
  raw.error = { name: error?.name ?? 'Error', message: String(error?.message ?? error), stack: String(error?.stack ?? '') };
  process.exitCode = 1;
} finally {
  try { await app?.close(); } catch (error) { raw.cleanupError = String(error); process.exitCode = 1; }
  if (started) {
    try { command(path.join(pgBin, 'pg_ctl'), ['-D', data, '-m', 'immediate', '-w', 'stop']); }
    catch (error) { raw.cleanupError = String(error); process.exitCode = 1; }
  }
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A same-company member has dashboard or audit-feed access while lacking permission to open the source expense or ticket.

Recorded result: A member denied direct access to a synthetic expense or support ticket still received its protected description, customer, or subject through the company activity feed. The direct object denial and feed disclosure were exercised through HTTP.

Scope: The local proofs used synthetic descriptions and customer names. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/attack-led/workers-bulk/run-support-event-reader.mjs:151-174` (test source).

```javascript
  assert.equal(miguel.session.permissions['dash.view']?.allowed, true);
  assert.equal(miguel.session.permissions['support.view']?.scope, 'crew');
  await send('foreman-still-hidden-after-grants', 'GET', detailPath, miguel.headers, undefined, 404);
  const pings = await send('foreman-alert-read', 'GET', '/api/tasks/pings?limit=50', miguel.headers, undefined, 200);
  const matchingPings = pings.json().items.filter((entry) => entry.body.includes(marker));
  const dashboard = await send('foreman-dashboard-read', 'GET', '/api/dashboard', miguel.headers, undefined, 200);
  const matchingActivity = (dashboard.json().activity ?? []).filter((entry) => entry.text.includes(marker));
  const foreignPings = await send('foreign-alert-control', 'GET', '/api/tasks/pings?limit=50', bob.headers, undefined, 200);
  const foreignDashboard = await send('foreign-dashboard-control', 'GET', '/api/dashboard', bob.headers, undefined, 200);
  assert.ok(!foreignPings.json().items.some((entry) => entry.body.includes(marker)), 'foreign alert leak');
  assert.ok(!(foreignDashboard.json().activity ?? []).some((entry) => entry.text.includes(marker)), 'foreign activity leak');
  raw.readerEffects = { matchingPings, matchingActivity, foreignPingMatches: 0, foreignActivityMatches: 0 };
  assert.equal(raw.providerAttempts, 0, 'unexpected external provider request');
  raw.verdict = matchingPings.length || matchingActivity.length ? 'PROVED-TECHNICAL-BEHAVIOR' : 'FALSIFIED-IN-TESTED-SCOPE';
  raw.conclusion = 'The foreman retained only crew-scoped support.view and received 404 for the ticket. The actual alert and activity reader results above determine whether broader task/audit permissions exposed customer or subject metadata.';
} catch (error) {
  raw.error = { name: error?.name ?? 'Error', message: String(error?.message ?? error), stack: String(error?.stack ?? '') };
  process.exitCode = 1;
} finally {
  try { await app?.close(); } catch (error) { raw.cleanupError = String(error); process.exitCode = 1; }
  if (started) {
    try { command(path.join(pgBin, 'pg_ctl'), ['-D', data, '-m', 'immediate', '-w', 'stop']); }
    catch (error) { raw.cleanupError = String(error); process.exitCode = 1; }
  }
```

## Remediation

Redact protected activity fields for denied readers or enforce source-object read authority on each derived activity row.
