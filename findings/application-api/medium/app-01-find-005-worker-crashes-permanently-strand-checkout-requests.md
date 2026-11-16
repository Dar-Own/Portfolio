# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Medium
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and a partial or modeled proof of the original behavior; the complete application or provider flow is not established by these excerpts.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Worker crashes permanently strand checkout requests

## Description

Five checkout rows stranded in working state block later checkout requests for the same pay link until manual database repair.

Prerequisite and affected workflow: The worker stops after claiming five requests and before their fulfillment commits.

Established observation: After five checkout requests were claimed and the worker stopped before committing fulfillment, all five remained in the working state and further requests using that link were refused. The recovery path did not reclaim those rows.

Limits: The local proof simulated the stop rather than crashing a deployed worker. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

The worker stops after claiming five requests and before their fulfillment commits.

Source: `apps/worker/src/jobs/checkout-requests.ts:142-162` (1d5d98db9acf).

```typescript
/** Takes every request waiting, oldest first, and answers each. */
export async function fulfilCheckoutRequests(options: CheckoutJobOptions): Promise<JobResult> {
  const companies = new Set<string>();
  let handled = 0;
  let errors = 0;
  for (;;) {
    // Claimed before the company's transaction opens, so two runs never take the same request.
    const { rows } = await options.pool.query<RequestRow>(
      `update app.checkout_requests r set status = 'working'
        where r.id in (select id from app.checkout_requests where status = 'pending' order by created_at limit 20 for update skip locked)
        returning r.id, r.tenant_id, r.token_hash, r.kind, r.return_url, r.provider_session_id`,
    );
    if (rows.length === 0) break;
    for (const row of rows) {
      companies.add(row.tenant_id);
      if ((await fulfil(options, row)) === 'done') handled += 1;
      else errors += 1;
    }
  }
  await options.pool.query(`delete from app.checkout_requests where status in ('done', 'failed') and answered_at < now() - interval '1 hour'`);
  return { companies: companies.size, processed: handled + errors, errors, summary: `${handled} answered, ${errors} failed` };
```

#### Step 2 — Cross the missing state or authority check

The worker autocommits the transition from `pending` to `working`, then fulfills rows separately. Later scans select only `pending`, and cleanup removes only terminal rows: `apps/worker/src/jobs/checkout-requests.ts:142-162`. The schema has no lease or claim timestamp, while the request door counts both `pending` and `working`: `packages/db/migrations/0106_automation_leaves_the_api.sql:455-507`.

Source: `packages/db/migrations/0106_automation_leaves_the_api.sql:455-507` (1d5d98db9acf).

```sql
create table app.checkout_requests (
  id                  uuid primary key default gen_random_uuid(),
  tenant_id           uuid not null references app.tenants(id) on delete cascade,
  token_hash          bytea not null constraint checkout_requests_token_hash_is_sha256 check (length(token_hash) = 32),
  -- 'checkout': open the provider's hosted page for what is due; 'confirm': the customer is back with the
  -- checkout's id, ask the provider how it ended and record it.
  kind                text not null constraint checkout_requests_kind_check check (kind in ('checkout', 'confirm')),
  return_url          text constraint checkout_requests_return_url_check check (length(return_url) <= 2000),
  provider_session_id text constraint checkout_requests_session_check check (provider_session_id ~ '^[A-Za-z0-9_-]{8,200}$'),
  status              text not null default 'pending'
                        constraint checkout_requests_status_check check (status in ('pending', 'working', 'done', 'failed')),
  -- The worker's answer: { checkoutUrl, isTest } for a checkout; { status, outcome, returnUrl, message? } for a confirm.
  answer              jsonb,
  error_code          text constraint checkout_requests_error_code_check check (error_code in ('invalid', 'not_found', 'unavailable')),
  error               text constraint checkout_requests_error_check check (length(error) <= 500),
  created_at          timestamptz not null default now(),
  answered_at         timestamptz,
  constraint checkout_requests_kind_fields check (case kind when 'checkout' then return_url is not null else provider_session_id is not null end)
);
create index checkout_requests_pending on app.checkout_requests (created_at) where status = 'pending';
comment on table app.checkout_requests is
  'What the customer''s pay page asked of the payment provider (migration 0106): written by the API on the pay link, done and answered by the worker as the company''s Automation member with the company''s own key. The API reads the answer back through app.checkout_request_answer only.';
revoke all on table app.checkout_requests from public;
grant select, delete, update (status, answer, error_code, error, answered_at) on table app.checkout_requests to app_worker;
-- Every app table has row-level security, enabled and forced; the worker claims a request before it opens
-- the company's transaction, so its one policy is unconditional, like the webhook inbox's.
alter table app.checkout_requests enable row level security;
alter table app.checkout_requests force row level security;
create policy worker_only on app.checkout_requests to app_worker using (true);

-- A request on a live pay link (0023's pay_token_tenant says whose). At most five unanswered ones per
-- link: a page pressing the button over and over is told to wait.
create function app.request_checkout(p_token text, p_kind text, p_return_url text, p_provider_session_id text) returns uuid
language plpgsql security definer set search_path = app, pg_temp as $$
declare
  h bytea := sha256(convert_to(coalesce(p_token, ''), 'UTF8'));
  company uuid := app.pay_token_tenant(sha256(convert_to(coalesce(p_token, ''), 'UTF8')));
  request_id uuid;
begin
  if company is null then
    raise exception 'This payment link isn''t valid any more. Ask us for a new one and we''ll send it over.' using errcode = '22023';
  end if;
  if p_kind not in ('checkout', 'confirm') then
    raise exception 'A pay page asks for a checkout or a confirmation' using errcode = '22023';
  end if;
  if (select count(*) from app.checkout_requests r where r.token_hash = h and r.status in ('pending', 'working')) >= 5 then
    raise exception 'Please wait a moment and try again.' using errcode = '54000';
  end if;
  insert into app.checkout_requests (tenant_id, token_hash, kind, return_url, provider_session_id)
  values (company, h, p_kind, p_return_url, p_provider_session_id)
  returning id into request_id;
  perform pg_notify('aos_checkout_requests', '');
  return request_id;
```


#### Step 3 — Observe the durable effect

After five checkout requests were claimed and the worker stopped before committing fulfillment, all five remained in the working state and further requests using that link were refused. The recovery path did not reclaim those rows.

The local proof simulated the stop rather than crashing a deployed worker.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/pocs/run-leads-003-010.mjs:443-466` (test source).

```javascript
function poc007WorkingRowsStranded(pg) {
  pg.psql('truncate app.checkout_requests;');
  for (let index = 0; index < 5; index++) {
    pg.psql(`select app.request_checkout('POC-PAY-TOKEN', 'checkout', '[redacted URL]', null);`);
  }
  const claimSql = `
    update app.checkout_requests r set status = 'working'
      where r.id in (select id from app.checkout_requests where status = 'pending' order by created_at limit 20 for update skip locked)
      returning r.id;
  `;
  const firstClaim = pg.psql(claimSql).stdout.split(/\r?\n/).filter(Boolean);
  assert.equal(firstClaim.length, 5);
  const afterRestart = pg.psql(claimSql).stdout;
  assert.equal(afterRestart, '');
  const sixth = pg.psql(`select app.request_checkout('POC-PAY-TOKEN', 'checkout', '[redacted URL]', null);`, { allowFailure: true });
  assert.notEqual(sixth.status, 0);
  assert.match(sixth.stderr, /Please wait a moment/);
  const statuses = pg.psql(`select status || ':' || count(*) from app.checkout_requests group by status order by status;`).stdout;
  assert.equal(statuses, 'working:5');

  const worker = readFileSync(path.join(repo, 'apps/worker/src/jobs/checkout-requests.ts'), 'utf8');
  assert.doesNotMatch(worker, /set status = 'pending'/);
  assert.match(worker, /delete from app\.checkout_requests where status in \('done', 'failed'\)/);
  return {
```

## Proof of concept

The original proof isolates a narrower component or modeled transition. It does not demonstrate every step of the complete external workflow.

Setup: The worker stops after claiming five requests and before their fulfillment commits.

Recorded result: After five checkout requests were claimed and the worker stopped before committing fulfillment, all five remained in the working state and further requests using that link were refused. The recovery path did not reclaim those rows.

Scope: The local proof simulated the stop rather than crashing a deployed worker. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/pocs/run-leads-003-010.mjs:443-466` (test source).

```javascript
function poc007WorkingRowsStranded(pg) {
  pg.psql('truncate app.checkout_requests;');
  for (let index = 0; index < 5; index++) {
    pg.psql(`select app.request_checkout('POC-PAY-TOKEN', 'checkout', '[redacted URL]', null);`);
  }
  const claimSql = `
    update app.checkout_requests r set status = 'working'
      where r.id in (select id from app.checkout_requests where status = 'pending' order by created_at limit 20 for update skip locked)
      returning r.id;
  `;
  const firstClaim = pg.psql(claimSql).stdout.split(/\r?\n/).filter(Boolean);
  assert.equal(firstClaim.length, 5);
  const afterRestart = pg.psql(claimSql).stdout;
  assert.equal(afterRestart, '');
  const sixth = pg.psql(`select app.request_checkout('POC-PAY-TOKEN', 'checkout', '[redacted URL]', null);`, { allowFailure: true });
  assert.notEqual(sixth.status, 0);
  assert.match(sixth.stderr, /Please wait a moment/);
  const statuses = pg.psql(`select status || ':' || count(*) from app.checkout_requests group by status order by status;`).stdout;
  assert.equal(statuses, 'working:5');

  const worker = readFileSync(path.join(repo, 'apps/worker/src/jobs/checkout-requests.ts'), 'utf8');
  assert.doesNotMatch(worker, /set status = 'pending'/);
  assert.match(worker, /delete from app\.checkout_requests where status in \('done', 'failed'\)/);
  return {
```

Source: `security-audit/pocs/run-leads-003-010.mjs:455-469` (test source).

```javascript
  const afterRestart = pg.psql(claimSql).stdout;
  assert.equal(afterRestart, '');
  const sixth = pg.psql(`select app.request_checkout('POC-PAY-TOKEN', 'checkout', '[redacted URL]', null);`, { allowFailure: true });
  assert.notEqual(sixth.status, 0);
  assert.match(sixth.stderr, /Please wait a moment/);
  const statuses = pg.psql(`select status || ':' || count(*) from app.checkout_requests group by status order by status;`).stdout;
  assert.equal(statuses, 'working:5');

  const worker = readFileSync(path.join(repo, 'apps/worker/src/jobs/checkout-requests.ts'), 'utf8');
  assert.doesNotMatch(worker, /set status = 'pending'/);
  assert.match(worker, /delete from app\.checkout_requests where status in \('done', 'failed'\)/);
  return {
    verdict: 'stranding-proved',
    claimedBeforeSimulatedCrash: firstClaim.length,
    reclaimedAfterRestart: 0,
```

## Remediation

Lease claimed rows, requeue expired claims, and make fulfillment safe to retry.
