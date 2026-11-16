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

# Job-comment route bypasses linked-lead visibility for a same-tenant member

## Description

A member denied the linked lead can read or create job comments through the weaker job-visibility rule. A prior comment author can also edit or delete after losing lead access.

Prerequisite and affected workflow: A same-company member retains job visibility and comment rights while the linked lead is hidden from them.

Established observation: A member denied the linked lead and paperwork could read job comments and add a new comment through the job route. Existing comment edit and delete operations also succeeded after that member lost lead access, while the lead-comment route denied the same actor.

Limits: The tests used synthetic comment bodies. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A same-company member retains job visibility and comment rights while the linked lead is hidden from them.

Source: `apps/api/src/features/comments/comments.ts:114-129` (1d5d98db9acf).

```typescript
/**
 * The record a thread belongs to, as the acting person sees it (404 otherwise), with the web app page
 * that opens its thread. Leads and jobs come through the person's own row-level security (a job's thread
 * is on its lead's drawer); other kinds are asked of app.can_see_comment_entity, which the migration that
 * adds them fills in.
 */
export async function requireVisibleEntity(ctx: DbContext, entity: CommentEntity): Promise<{ path: string }> {
  if (entity.entityType === 'lead' || entity.entityType === 'job') {
    const { rows } = await ctx.client.query<{ lead_id: string }>(
      entity.entityType === 'lead'
        ? 'select l.id as lead_id from app.leads l where l.tenant_id = $1 and l.id = $2'
        : 'select j.lead_id from app.jobs j where j.tenant_id = $1 and j.id = $2',
      [ctx.tenantId, entity.entityId],
    );
    if (!rows[0]) throw notFound(RECORD_NOT_FOUND);
    return { path: `/pipelines/${rows[0].lead_id}` };
```

#### Step 2 — Cross the missing state or authority check

Comment routes and final RLS use job visibility without enforcing the linked lead's read boundary.

Source: `packages/db/migrations/0036_comments.sql:84-94` (1d5d98db9acf).

```sql
-- The one reading rule, for the policies below: a lead or a job is visible when the reader's own
-- view of app.leads / app.jobs has it (this function runs as the caller, so their row-level
-- security applies); anything else is asked of the hook above.
create function app.comment_record_visible(p_tenant uuid, p_type text, p_id uuid) returns boolean
language sql stable as $$
  select case p_type
    when 'lead' then exists (select 1 from app.leads l where l.tenant_id = p_tenant and l.id = p_id)
    when 'job'  then exists (select 1 from app.jobs j where j.tenant_id = p_tenant and j.id = p_id)
    else app.can_see_comment_entity(p_type, p_id)
  end
$$;
```


#### Step 3 — Observe the durable effect

A member denied the linked lead and paperwork could read job comments and add a new comment through the job route. Existing comment edit and delete operations also succeeded after that member lost lead access, while the lead-comment route denied the same actor.

The tests used synthetic comment bodies.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/campaign-2026-10-08/whiteboard-ingress/run-job-comment-boundary.mjs:121-144` (test source).

```javascript
  assert.equal(reads.owner.job, 200); assert.equal(reads.owner.jobHasMarker, true);
  assert.equal(reads.denied.lead, 404); assert.equal(reads.denied.job, 200); assert.equal(reads.denied.jobHasMarker, true);
  assert.equal(reads.foreign.job, 404); assert.equal(reads.foreign.jobHasMarker, false);
  assert.equal(anonymous.statusCode, 401);
  const deniedPostMarker = `SYNTHETIC-DENIED-LEAD-WRITE-${process.pid}`;
  const deniedPost = await app.inject({ method: 'POST', url: '/api/comments', headers: signed.denied.headers, payload: { entityType: 'job', entityId: job, body: deniedPostMarker } });
  assert.equal(deniedPost.statusCode, 201, deniedPost.body);
  const deniedLeadPost = await app.inject({ method: 'POST', url: '/api/comments', headers: signed.denied.headers, payload: { entityType: 'lead', entityId: lead, body: `SYNTHETIC-BLOCKED-${process.pid}` } });
  assert.equal(deniedLeadPost.statusCode, 404, deniedLeadPost.body);
  const after = effects(tenant, job);
  const foreignRows = Number(one(`select count(*) from app.comments where tenant_id=${q(foreignTenant)} and (body like '%${process.pid}%')`));
  assert.equal(afterOwner.jobComments, prior.jobComments + 1);
  assert.equal(after.jobComments, afterOwner.jobComments + 1); assert.equal(foreignRows, 0);
  const audit = {
    revision: pinned, test: 'job-comments-alternate-to-hidden-lead', isolation: { socketOnly: true, migrations: migrations.length, spatialShim: 'inert and unrelated' },
    actors: { owner: 'Alex, same tenant owner', denied: 'Andre, own-scope estimator', foreign: 'Bob, foreign tenant', anonymous: 'no cookie' },
    leadStatus, jobPageStatus, easyJobIdSource: { route: '/api/expenses/options', status: picker.statusCode, includesTargetJobId: true, customerName: pickerJob.customerName }, before: prior,
    ownerPost: { status: ownerPost.statusCode, bodyHasMarker: ownerPost.body.includes(marker) },
    reads, anonymous: { job: anonymous.statusCode },
    deniedPost: { job: deniedPost.statusCode, bodyHasMarker: deniedPost.body.includes(deniedPostMarker), lead: deniedLeadPost.statusCode },
    afterOwner, after: { ...after, foreignMarkerRows: foreignRows },
    disposition: 'PROVED route-level alternate read/write of job comment while lead detail/thread denied; impact conditional on job-comment policy',
  };
  console.log(JSON.stringify(audit, null, 2));
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A same-company member retains job visibility and comment rights while the linked lead is hidden from them.

Recorded result: A member denied the linked lead and paperwork could read job comments and add a new comment through the job route. Existing comment edit and delete operations also succeeded after that member lost lead access, while the lead-comment route denied the same actor.

Scope: The tests used synthetic comment bodies. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/campaign-2026-10-08/whiteboard-ingress/run-job-comment-boundary.mjs:121-144` (test source).

```javascript
  assert.equal(reads.owner.job, 200); assert.equal(reads.owner.jobHasMarker, true);
  assert.equal(reads.denied.lead, 404); assert.equal(reads.denied.job, 200); assert.equal(reads.denied.jobHasMarker, true);
  assert.equal(reads.foreign.job, 404); assert.equal(reads.foreign.jobHasMarker, false);
  assert.equal(anonymous.statusCode, 401);
  const deniedPostMarker = `SYNTHETIC-DENIED-LEAD-WRITE-${process.pid}`;
  const deniedPost = await app.inject({ method: 'POST', url: '/api/comments', headers: signed.denied.headers, payload: { entityType: 'job', entityId: job, body: deniedPostMarker } });
  assert.equal(deniedPost.statusCode, 201, deniedPost.body);
  const deniedLeadPost = await app.inject({ method: 'POST', url: '/api/comments', headers: signed.denied.headers, payload: { entityType: 'lead', entityId: lead, body: `SYNTHETIC-BLOCKED-${process.pid}` } });
  assert.equal(deniedLeadPost.statusCode, 404, deniedLeadPost.body);
  const after = effects(tenant, job);
  const foreignRows = Number(one(`select count(*) from app.comments where tenant_id=${q(foreignTenant)} and (body like '%${process.pid}%')`));
  assert.equal(afterOwner.jobComments, prior.jobComments + 1);
  assert.equal(after.jobComments, afterOwner.jobComments + 1); assert.equal(foreignRows, 0);
  const audit = {
    revision: pinned, test: 'job-comments-alternate-to-hidden-lead', isolation: { socketOnly: true, migrations: migrations.length, spatialShim: 'inert and unrelated' },
    actors: { owner: 'Alex, same tenant owner', denied: 'Andre, own-scope estimator', foreign: 'Bob, foreign tenant', anonymous: 'no cookie' },
    leadStatus, jobPageStatus, easyJobIdSource: { route: '/api/expenses/options', status: picker.statusCode, includesTargetJobId: true, customerName: pickerJob.customerName }, before: prior,
    ownerPost: { status: ownerPost.statusCode, bodyHasMarker: ownerPost.body.includes(marker) },
    reads, anonymous: { job: anonymous.statusCode },
    deniedPost: { job: deniedPost.statusCode, bodyHasMarker: deniedPost.body.includes(deniedPostMarker), lead: deniedLeadPost.statusCode },
    afterOwner, after: { ...after, foreignMarkerRows: foreignRows },
    disposition: 'PROVED route-level alternate read/write of job comment while lead detail/thread denied; impact conditional on job-comment policy',
  };
  console.log(JSON.stringify(audit, null, 2));
```

## Remediation

Enforce linked-lead visibility in both comment routes and final database read and mutation doors.
