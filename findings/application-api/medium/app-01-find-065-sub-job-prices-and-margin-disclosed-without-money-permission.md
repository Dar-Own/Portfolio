# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Medium (same-tenant financial confidentiality)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Sub-job prices and margin disclosed without money permission

## Description

A sub-job reader explicitly denied `money.costs` can receive subcontract price, customer price, and margin for a same-company job.

Prerequisite and affected workflow: A member allowed to view sub-jobs but explicitly denied cost access opens a known sub-job detail.

Established observation: A member with sub-job view permission and an explicit financial-cost deny received subcontract price, customer price, and margin in the sub-job response. A member without sub-job access and a foreign-company member were denied.

Limits: Foreign-tenant disclosure and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A member allowed to view sub-jobs but explicitly denied cost access opens a known sub-job detail.

Source: `apps/api/src/features/subjobs/routes.ts:116-121` (1d5d98db9acf).

```typescript
    // The job's active sub job, for the job drawer's "Subcontract this job" link.
    app.get('/by-job/:jobId', { schema: { params: z.object({ jobId: z.uuid() }) } }, async (request) => {
      requirePermission(request.session, 'subjobs.view');
      const { jobId } = request.params;
      return app.db.withUser(request.session, async (ctx) => ({ subJob: await getSubJobForJob(ctx, jobId, await viewFor(ctx)) }));
    });
```

#### Step 2 — Cross the missing state or authority check

The sub-job detail endpoint checks `subjobs.view` but returns financial fields without checking `money.costs`.

Source: `apps/api/src/features/subjobs/store.ts:123-136` (1d5d98db9acf).

```typescript
const SELECT_SUB_JOB = `
  select s.id, s.stage, s.stage_entered_at, s.offered_at, s.accepted_at, s.started_at, s.completed_at, s.paid_at,
         s.sub_price_cents, s.customer_price_cents, s.notes,
         j.id as job_id, j.number as job_number, j.status::text as job_status, j.lead_id,
         c.display_name as customer_name, p.address_line, p.city,
         s.vendor_id, v.company_name as vendor_name, v.contact_name as vendor_contact, v.phone_e164 as vendor_phone,
         v.email::text as vendor_email, v.is_active as vendor_active, v.coi_expires_on,
         s.bill_id, b.amount_cents as bill_amount_cents, b.due_on as bill_due_on, b.status as bill_status, b.reference as bill_reference,
         s.created_by, cu.full_name as created_by_name, s.created_at, s.updated_at
    from app.sub_jobs s
    join app.jobs j on j.tenant_id = s.tenant_id and j.id = s.job_id
    left join app.leads l on l.tenant_id = j.tenant_id and l.id = j.lead_id
    left join app.customers c on c.tenant_id = l.tenant_id and c.id = l.customer_id
    left join app.properties p on p.tenant_id = l.tenant_id and p.id = l.property_id
```


#### Step 3 — Observe the durable effect

A member with sub-job view permission and an explicit financial-cost deny received subcontract price, customer price, and margin in the sub-job response. A member without sub-job access and a foreign-company member were denied.

Foreign-tenant disclosure and deployed effect remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/complementary/cases/subjob-money-denied.mjs:49-72` (test source).

```javascript
    const blindOwn = await read('money-denied-own-job', blind, own);
    const lead = await app.inject({ method: 'GET', url: `/api/leads/${hidden.leadId}`, headers: blind.headers });
    const blindHidden = await read('money-denied-hidden-lead-job', blind, hidden);
    const foreignRead = await read('foreign-tenant-job', foreign, own);
    const lesserRead = await read('no-subjobs-view', lesser, own);
    const after = ctx.sql(`select json_build_object('subjobs',(select count(*) from app.sub_jobs),'events',(select count(*) from app.domain_events),'messages',(select count(*) from app.messages),'audit',(select count(*) from app.audit_log))::text`);
    assert.equal(before, after, 'GET controls mutated tracked rows');
    assert.equal(positive.status, 200); assert.equal(positive.fields?.id, own.subJobId);
    assert.equal(blindOwn.status, 200); assert.equal(blindOwn.fields?.id, own.subJobId);
    assert.ok([403, 404].includes(lead.statusCode), 'hidden lead control did not deny');
    assert.equal(foreignRead.fields, null, 'foreign object returned');
    assert.ok([401, 403].includes(lesserRead.status), 'no-subjobs-view actor reached object');
    assert.equal(ctx.providerAttempts.length, 0);
    const keys = ['subPriceCents', 'customerPriceCents', 'marginCents', 'billAmountCents'];
    const exposed = keys.filter((key) => typeof blindOwn.fields?.[key] === 'number');
    return { verdict: exposed.length ? 'TEST-CONFIRMED' : 'FALSIFIED-IN-TESTED-SCOPE',
      summary: exposed.length ? `A member explicitly denied money.costs received ${exposed.join(', ')} from R367 despite the client rule.` : 'R367 suppressed money fields for the explicitly cost-denied member.',
      evidence: { costAllowed: costAllowed.session.permissions['money.costs'], costDenied: blind.session.permissions['money.costs'],
        subjobsView: blind.session.permissions['subjobs.view'], positive, blindOwn, blindHidden,
        hiddenLeadStatus: lead.statusCode, foreignRead, lesserRead, before, after,
        source: ['apps/api/src/features/subjobs/routes.ts:116-121', 'apps/api/src/features/subjobs/store.ts:123-136', 'apps/api/src/features/subjobs/store.ts:225-232'] },
      limitations: ['Synthetic local app.inject with final migrations; no Caddy/deployed proof.', 'This tests R367 by-job; other subjob projections require separate reconciliation.'] };
  } finally { await app.close(); }
}
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A member allowed to view sub-jobs but explicitly denied cost access opens a known sub-job detail.

Recorded result: A member with sub-job view permission and an explicit financial-cost deny received subcontract price, customer price, and margin in the sub-job response. A member without sub-job access and a foreign-company member were denied.

Scope: Foreign-tenant disclosure and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/complementary/cases/subjob-money-denied.mjs:49-72` (test source).

```javascript
    const blindOwn = await read('money-denied-own-job', blind, own);
    const lead = await app.inject({ method: 'GET', url: `/api/leads/${hidden.leadId}`, headers: blind.headers });
    const blindHidden = await read('money-denied-hidden-lead-job', blind, hidden);
    const foreignRead = await read('foreign-tenant-job', foreign, own);
    const lesserRead = await read('no-subjobs-view', lesser, own);
    const after = ctx.sql(`select json_build_object('subjobs',(select count(*) from app.sub_jobs),'events',(select count(*) from app.domain_events),'messages',(select count(*) from app.messages),'audit',(select count(*) from app.audit_log))::text`);
    assert.equal(before, after, 'GET controls mutated tracked rows');
    assert.equal(positive.status, 200); assert.equal(positive.fields?.id, own.subJobId);
    assert.equal(blindOwn.status, 200); assert.equal(blindOwn.fields?.id, own.subJobId);
    assert.ok([403, 404].includes(lead.statusCode), 'hidden lead control did not deny');
    assert.equal(foreignRead.fields, null, 'foreign object returned');
    assert.ok([401, 403].includes(lesserRead.status), 'no-subjobs-view actor reached object');
    assert.equal(ctx.providerAttempts.length, 0);
    const keys = ['subPriceCents', 'customerPriceCents', 'marginCents', 'billAmountCents'];
    const exposed = keys.filter((key) => typeof blindOwn.fields?.[key] === 'number');
    return { verdict: exposed.length ? 'TEST-CONFIRMED' : 'FALSIFIED-IN-TESTED-SCOPE',
      summary: exposed.length ? `A member explicitly denied money.costs received ${exposed.join(', ')} from R367 despite the client rule.` : 'R367 suppressed money fields for the explicitly cost-denied member.',
      evidence: { costAllowed: costAllowed.session.permissions['money.costs'], costDenied: blind.session.permissions['money.costs'],
        subjobsView: blind.session.permissions['subjobs.view'], positive, blindOwn, blindHidden,
        hiddenLeadStatus: lead.statusCode, foreignRead, lesserRead, before, after,
        source: ['apps/api/src/features/subjobs/routes.ts:116-121', 'apps/api/src/features/subjobs/store.ts:123-136', 'apps/api/src/features/subjobs/store.ts:225-232'] },
      limitations: ['Synthetic local app.inject with final migrations; no Caddy/deployed proof.', 'This tests R367 by-job; other subjob projections require separate reconciliation.'] };
  } finally { await app.close(); }
}
```

## Remediation

Mask financial fields at every sub-job response projection unless the reader has effective money-field authority.
