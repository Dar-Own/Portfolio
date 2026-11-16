# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Medium (operational integrity)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Default Foreman can add load tickets to another crew's job

## Description

A default Foreman can add load tickets to another crew's job and change its delivered-ton total.

Prerequisite and affected workflow: A Foreman assigned to Crew A records a load against a Crew B job in the same company.

Established observation: A Foreman assigned to Crew A submitted a load ticket for Crew B's same-company job. The application accepted the ticket and changed that job's delivered-ton total; a foreign-company job remained denied.

Limits: Real delivery, billing, and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A Foreman assigned to Crew A records a load against a Crew B job in the same company.

Source: `apps/api/src/features/loads/routes.ts:63-68` (1d5d98db9acf).

```typescript
    app.post('/', { schema: { body: recordBody } }, async (request, reply) => {
      requireLoadsRecord(request.session);
      const ticket = await app.db.withUser(request.session, (ctx) => recordLoad(ctx, request.body));
      return reply.status(201).send(ticket);
    });

```

#### Step 2 — Cross the missing state or authority check

The database policy treats the default `tickets.add` permission as an unrestricted alternative to crew-scoped `loads.record`.

Source: `packages/db/migrations/0035_asphalt_per_ton_and_loads.sql:223-239` (1d5d98db9acf).

```sql
-- Writing: tickets.add as before, or a driver on their own ticket for a job scheduled to their crew.
drop policy perm_insert on app.load_tickets;
drop policy perm_update on app.load_tickets;
create policy perm_insert on app.load_tickets as restrictive for insert to app_user
  with check ((select app.can('tickets.add'))
              or ((select app.can('loads.record'))
                  and recorded_by = (select app.current_user_id())
                  and app.job_scheduled_to_my_crew(tenant_id, job_id)));
create policy perm_update on app.load_tickets as restrictive for update to app_user
  using ((select app.can('tickets.add'))
         or ((select app.can('loads.record'))
             and recorded_by = (select app.current_user_id())
             and app.job_scheduled_to_my_crew(tenant_id, job_id)))
  with check ((select app.can('tickets.add'))
              or ((select app.can('loads.record'))
                  and recorded_by = (select app.current_user_id())
                  and app.job_scheduled_to_my_crew(tenant_id, job_id)));
```


#### Step 3 — Observe the durable effect

A Foreman assigned to Crew A submitted a load ticket for Crew B's same-company job. The application accepted the ticket and changed that job's delivered-ton total; a foreign-company job remained denied.

Real delivery, billing, and deployed effect remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/attack-led/authority-data/run-loads-cross-crew-authority.mjs:183-210` (synthetic test).

```javascript
  const ownWrite = await request('foreman-own-crew-write-positive', 'POST', '/api/loads', foreman.headers,
    { jobId: a.id, plantId, haulerName: 'Synthetic outside hauler', ticketNo: ownTicket, tons: 10 }, a.id, b.id);
  assert.equal(ownWrite.response.statusCode, 201, ownWrite.response.body);
  assert.equal(ticket(ownWrite.after, ownTicket)?.job_id, a.id);
  const ownAfter = await request('foreman-own-crew-total-after-write', 'GET', `/api/loads/job/${a.id}`, foreman.headers, undefined, a.id, b.id);
  assert.equal(ownAfter.response.statusCode, 200, ownAfter.response.body);
  assert.equal(Math.round((ownAfter.result.deliveredTons - ownRead.result.deliveredTons) * 1000) / 1000, 10);

  const crossTicket = 'AOS-FOREMAN-CROSS-' + randomBytes(4).toString('hex');
  const crossWrite = await request('foreman-hidden-other-crew-write', 'POST', '/api/loads', foreman.headers,
    { jobId: b.id, plantId, haulerName: 'Synthetic outside hauler', ticketNo: crossTicket, tons: 7.5 }, a.id, b.id);
  assert.equal(crossWrite.response.statusCode, 201, crossWrite.response.body);
  assert.equal(crossWrite.result.jobId, b.id);
  assert.equal(crossWrite.result.recordedBy?.fullName, 'Tyler Nash');
  assert.equal(ticket(crossWrite.after, crossTicket)?.job_id, b.id);
  const bAfter = await request('foreman-other-crew-total-after-write', 'GET', `/api/loads/job/${b.id}`, foreman.headers, undefined, a.id, b.id);
  assert.equal(bAfter.response.statusCode, 200, bAfter.response.body);
  assert.equal(Math.round((bAfter.result.deliveredTons - otherRead.result.deliveredTons) * 1000) / 1000, 7.5);
  assert.ok(bAfter.result.tickets.some((item) => item.ticketNo === crossTicket));

  const duplicate = await request('same-plant-duplicate-ticket-denied', 'POST', '/api/loads', foreman.headers,
    { jobId: b.id, plantId, haulerName: 'Synthetic outside hauler', ticketNo: crossTicket, tons: 7.5 }, a.id, b.id);
  assert.equal(duplicate.response.statusCode, 422, duplicate.response.body);
  await expectNoBusinessChange(duplicate, 'duplicate');
  const foreignRead = await request('foreign-company-summary-denied', 'GET', `/api/loads/job/${b.id}`, foreign.headers, undefined, a.id, b.id);
  assert.equal(foreignRead.response.statusCode, 404, foreignRead.response.body);
  await expectNoBusinessChange(foreignRead, 'foreign read');
  const foreignWrite = await request('foreign-company-load-denied', 'POST', '/api/loads', foreign.headers,
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A Foreman assigned to Crew A records a load against a Crew B job in the same company.

Recorded result: A Foreman assigned to Crew A submitted a load ticket for Crew B's same-company job. The application accepted the ticket and changed that job's delivered-ton total; a foreign-company job remained denied.

Scope: Real delivery, billing, and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/attack-led/authority-data/run-loads-cross-crew-authority.mjs:183-210` (synthetic test).

```javascript
  const ownWrite = await request('foreman-own-crew-write-positive', 'POST', '/api/loads', foreman.headers,
    { jobId: a.id, plantId, haulerName: 'Synthetic outside hauler', ticketNo: ownTicket, tons: 10 }, a.id, b.id);
  assert.equal(ownWrite.response.statusCode, 201, ownWrite.response.body);
  assert.equal(ticket(ownWrite.after, ownTicket)?.job_id, a.id);
  const ownAfter = await request('foreman-own-crew-total-after-write', 'GET', `/api/loads/job/${a.id}`, foreman.headers, undefined, a.id, b.id);
  assert.equal(ownAfter.response.statusCode, 200, ownAfter.response.body);
  assert.equal(Math.round((ownAfter.result.deliveredTons - ownRead.result.deliveredTons) * 1000) / 1000, 10);

  const crossTicket = 'AOS-FOREMAN-CROSS-' + randomBytes(4).toString('hex');
  const crossWrite = await request('foreman-hidden-other-crew-write', 'POST', '/api/loads', foreman.headers,
    { jobId: b.id, plantId, haulerName: 'Synthetic outside hauler', ticketNo: crossTicket, tons: 7.5 }, a.id, b.id);
  assert.equal(crossWrite.response.statusCode, 201, crossWrite.response.body);
  assert.equal(crossWrite.result.jobId, b.id);
  assert.equal(crossWrite.result.recordedBy?.fullName, 'Tyler Nash');
  assert.equal(ticket(crossWrite.after, crossTicket)?.job_id, b.id);
  const bAfter = await request('foreman-other-crew-total-after-write', 'GET', `/api/loads/job/${b.id}`, foreman.headers, undefined, a.id, b.id);
  assert.equal(bAfter.response.statusCode, 200, bAfter.response.body);
  assert.equal(Math.round((bAfter.result.deliveredTons - otherRead.result.deliveredTons) * 1000) / 1000, 7.5);
  assert.ok(bAfter.result.tickets.some((item) => item.ticketNo === crossTicket));

  const duplicate = await request('same-plant-duplicate-ticket-denied', 'POST', '/api/loads', foreman.headers,
    { jobId: b.id, plantId, haulerName: 'Synthetic outside hauler', ticketNo: crossTicket, tons: 7.5 }, a.id, b.id);
  assert.equal(duplicate.response.statusCode, 422, duplicate.response.body);
  await expectNoBusinessChange(duplicate, 'duplicate');
  const foreignRead = await request('foreign-company-summary-denied', 'GET', `/api/loads/job/${b.id}`, foreign.headers, undefined, a.id, b.id);
  assert.equal(foreignRead.response.statusCode, 404, foreignRead.response.body);
  await expectNoBusinessChange(foreignRead, 'foreign read');
  const foreignWrite = await request('foreign-company-load-denied', 'POST', '/api/loads', foreign.headers,
```

## Remediation

Require the Foreman to be assigned to the job's crew in the database policy and the API permission check.
