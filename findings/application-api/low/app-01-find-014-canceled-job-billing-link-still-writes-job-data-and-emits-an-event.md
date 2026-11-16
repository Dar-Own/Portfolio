# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and a partial or modeled proof of the original behavior; the complete application or provider flow is not established by these excerpts.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Canceled-job billing link still writes job data and emits an event

## Description

A canceled job still accepts its earlier billing link, changes billing state, closes a chase task, and emits a submission event.

Prerequisite and affected workflow: A bearer submits a valid unused billing-form token issued before an authorized job cancellation.

Established observation: After a job was canceled, its previously issued billing token still passed the final database lookup and submission doors. The submission changed billing state, closed a chase task, and emitted an event.

Limits: The public HTTP handlers were source-traced; this PoC exercised their final SQL doors. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A bearer submits a valid unused billing-form token issued before an authorized job cancellation.

Source: `packages/db/migrations/0015_handoff_billing_form.sql:129-146` (1d5d98db9acf).

```sql
    from app.billing_form_tokens t
    join app.tenants tn on tn.id = t.tenant_id and tn.suspended_at is null
    join app.jobs j on j.tenant_id = t.tenant_id and j.id = t.job_id
    join app.leads l on l.tenant_id = j.tenant_id and l.id = j.lead_id
    join app.customers c on c.tenant_id = l.tenant_id and c.id = l.customer_id
    join app.business_units b on b.tenant_id = j.tenant_id and b.id = j.business_unit_id
    left join app.properties p on p.tenant_id = l.tenant_id and p.id = l.property_id
    left join lateral (
      select x.full_name from app.contacts x
       where x.tenant_id = l.tenant_id and x.customer_id = l.customer_id
       order by x.is_primary desc, x.created_at, x.id
       limit 1
    ) ct on true
   where p_token_hash is not null
     and length(p_token_hash) = 32
     and t.token_hash = p_token_hash
     and t.revoked_at is null
     and t.expires_at > now()
```

#### Step 2 — Cross the missing state or authority check

The signed-in status route can move an open job to `cancelled` (`apps/api/src/features/leads/routes.ts:127-132`; `apps/api/src/features/leads/status.ts:172-190,238-243`; `packages/db/migrations/0020_unified_job_statuses.sql:212-215`). The public billing lookup checks token hash, expiry, revocation, and tenant suspension, but omits job status (`packages/db/migrations/0015_handoff_billing_form.sql:118-147`). The final submission function locks and uses the token, updates the job and tasks, and emits a system event without checking `jobs.status` (`packages/db/migrations/0106_automation_leaves_the_api.sql:386-411`). No cancellation path revokes those links.

Source: `packages/db/migrations/0106_automation_leaves_the_api.sql:386-410` (1d5d98db9acf).

```sql
  select k.id, k.tenant_id, k.job_id, k.used_at, k.revoked_at, k.expires_at into link
    from app.billing_form_tokens k
    join app.tenants t on t.id = k.tenant_id and t.suspended_at is null
   where k.token_hash = sha256(convert_to(coalesce(p_token, ''), 'UTF8'))
   for update of k;
  if not found or link.revoked_at is not null or link.expires_at <= now() then
    return 'invalid_link';
  end if;
  if link.used_at is not null then
    return 'already_received';
  end if;
  update app.billing_form_tokens k set used_at = now() where k.id = link.id;
  update app.jobs j
     set ap_contact_name = v_name, ap_email = v_email, po_number = v_po, billing_address = v_address,
         tax_exempt = coalesce(p_tax_exempt, false), billing_returned_at = now()
   where j.tenant_id = link.tenant_id and j.id = link.job_id;
  if not found then
    return 'invalid_link';
  end if;
  update app.job_documents d set status = 'received'
   where d.tenant_id = link.tenant_id and d.job_id = link.job_id and d.doc_type = 'po_billing_form' and d.status <> 'received';
  -- The chase task (the worker's billing-form-chase job) is done: the customer answered.
  update app.tasks t set status = 'done', completed_at = now(), completed_by = app.automation_user_id(link.tenant_id)
   where t.tenant_id = link.tenant_id and t.job_id = link.job_id and t.status = 'open' and t.title ilike '%chase%';
  perform app._emit_system_event(link.tenant_id, 'billing_form.submitted', link.job_id);
```


#### Step 3 — Observe the durable effect

After a job was canceled, its previously issued billing token still passed the final database lookup and submission doors. The submission changed billing state, closed a chase task, and emitted an event.

The public HTTP handlers were source-traced; this PoC exercised their final SQL doors.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/pocs/run-lead-016.mjs:132-155` (test source).

```javascript
    select count(*) from app.billing_form_by_token(decode('${cancelledHash}', 'hex'));
    reset role;`);
  assert.equal(cancelledView, '1');
  const cancelledAccepted = sql(`set role poc_billing_auth;
    select app.submit_billing_form('${cancelledToken}', 'Accounts Payable', '[redacted email]', null, '100 Test Rd', false);
    reset role;`);
  assert.equal(cancelledAccepted, 'saved');
  const cancelledState = sql(`select status::text || '|' || (billing_returned_at is not null)::text from app.jobs where id = '${otherJob}';`);
  assert.equal(cancelledState, 'cancelled|true');
  const cancelledTask = sql(`select status::text from app.tasks where id = '${otherJobChase}';`);
  assert.equal(cancelledTask, 'done');
  const cancelledEventCount = sql(`select count(*) from app.domain_events where tenant_id = '${tenant}' and name = 'billing_form.submitted' and subject = '${otherJob}';`);
  assert.equal(cancelledEventCount, '1');
  const result = { snapshot: expected, environment: 'PostgreSQL 16; Unix socket only; synthetic tenant; PostGIS spatial columns stubbed', migrationCount: migrations.length,
    publicRoute: '/api/public/billing/:token', invalid, accepted, replay,
    tasks: { billingChase: byId[billingTask], unrelatedSafetyChase: byId[unrelatedChase], sameJobNonChase: byId[nonChase], otherJobChaseBeforeCancelledSubmit: byId[otherJobChase] },
    cancelledJob: { linkViewRows: Number(cancelledView), submission: cancelledAccepted, finalJobState: cancelledState,
      chaseTaskStatus: cancelledTask, submittedEventCount: Number(cancelledEventCount), rejectedControls: rejectedCancelledTokens } };
  const output = path.join(repo, 'security-audit/pocs/results/2026-10-06-lead-016.json');
  writeFileSync(output, JSON.stringify(result, null, 2) + '\n');
  process.stdout.write(JSON.stringify(result, null, 2) + '\n');
} finally {
  if (started) spawnSync(path.join(pgBin, 'pg_ctl'), ['-D', data, '-m', 'immediate', 'stop'], { cwd: repo, stdio: 'ignore' });
  rmSync(root, { recursive: true, force: true });
```

## Proof of concept

The original proof isolates a narrower component or modeled transition. It does not demonstrate every step of the complete external workflow.

Setup: A bearer submits a valid unused billing-form token issued before an authorized job cancellation.

Recorded result: After a job was canceled, its previously issued billing token still passed the final database lookup and submission doors. The submission changed billing state, closed a chase task, and emitted an event.

Scope: The public HTTP handlers were source-traced; this PoC exercised their final SQL doors. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/pocs/run-lead-016.mjs:132-155` (test source).

```javascript
    select count(*) from app.billing_form_by_token(decode('${cancelledHash}', 'hex'));
    reset role;`);
  assert.equal(cancelledView, '1');
  const cancelledAccepted = sql(`set role poc_billing_auth;
    select app.submit_billing_form('${cancelledToken}', 'Accounts Payable', '[redacted email]', null, '100 Test Rd', false);
    reset role;`);
  assert.equal(cancelledAccepted, 'saved');
  const cancelledState = sql(`select status::text || '|' || (billing_returned_at is not null)::text from app.jobs where id = '${otherJob}';`);
  assert.equal(cancelledState, 'cancelled|true');
  const cancelledTask = sql(`select status::text from app.tasks where id = '${otherJobChase}';`);
  assert.equal(cancelledTask, 'done');
  const cancelledEventCount = sql(`select count(*) from app.domain_events where tenant_id = '${tenant}' and name = 'billing_form.submitted' and subject = '${otherJob}';`);
  assert.equal(cancelledEventCount, '1');
  const result = { snapshot: expected, environment: 'PostgreSQL 16; Unix socket only; synthetic tenant; PostGIS spatial columns stubbed', migrationCount: migrations.length,
    publicRoute: '/api/public/billing/:token', invalid, accepted, replay,
    tasks: { billingChase: byId[billingTask], unrelatedSafetyChase: byId[unrelatedChase], sameJobNonChase: byId[nonChase], otherJobChaseBeforeCancelledSubmit: byId[otherJobChase] },
    cancelledJob: { linkViewRows: Number(cancelledView), submission: cancelledAccepted, finalJobState: cancelledState,
      chaseTaskStatus: cancelledTask, submittedEventCount: Number(cancelledEventCount), rejectedControls: rejectedCancelledTokens } };
  const output = path.join(repo, 'security-audit/pocs/results/2026-10-06-lead-016.json');
  writeFileSync(output, JSON.stringify(result, null, 2) + '\n');
  process.stdout.write(JSON.stringify(result, null, 2) + '\n');
} finally {
  if (started) spawnSync(path.join(pgBin, 'pg_ctl'), ['-D', data, '-m', 'immediate', 'stop'], { cwd: repo, stdio: 'ignore' });
  rmSync(root, { recursive: true, force: true });
```

## Remediation

Revoke outstanding billing tokens in the cancellation transaction and reject canceled jobs at both public lookup and submission.
