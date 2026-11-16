# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low (same-tenant confidentiality)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Foreman reads another crew's customer and site through Plants and Loads

## Description

A crew-scoped Foreman can read another crew's customer and full site address through Plants and Loads.

Prerequisite and affected workflow: A Crew A Foreman calls the Plants or Loads reads for a same-company Crew B job hidden by Schedule.

Established observation: Schedule hid Crew B's job from a Crew A Foreman, but Plants and Loads responses disclosed Crew B's customer and full site address. Lesser-role and foreign-company controls did not expose the same data.

Limits: Real customer disclosure and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A Crew A Foreman calls the Plants or Loads reads for a same-company Crew B job hidden by Schedule.

Source: `apps/api/src/features/plants/routes.ts:73-75` (1d5d98db9acf).

```typescript
/** plants.view or plants.manage: either lets a person look. */
function requireView(session: Session): void {
  if (!can(session, 'plants.view') && !can(session, 'plants.manage')) throw forbidden(permissionDeniedMessage('plants.view'));
```

#### Step 2 — Cross the missing state or authority check

Plants and Loads reads omit the crew filter that hides the same job in Schedule.

Source: `apps/api/src/features/plants/store.ts:463-477` (1d5d98db9acf).

```typescript
/** Jobs a plant can be picked for: accepted, scheduled, in progress or on hold; the ones with a date first. */
export async function listPickableJobs(ctx: DbContext): Promise<JobSite[]> {
  const { rows } = await ctx.client.query<{ id: string }>(
    `select j.id from app.jobs j
      where j.tenant_id = $1 and j.status::text in ('unscheduled', 'scheduled', 'in_progress', 'on_hold')
      order by j.created_at desc, j.id
      limit 200`,
    [ctx.tenantId],
  );
  const sites = await jobSites(ctx, rows.map((row) => row.id));
  return sites.sort((a, b) => {
    if (a.firstWorkDay && b.firstWorkDay && a.firstWorkDay !== b.firstWorkDay) return a.firstWorkDay < b.firstWorkDay ? -1 : 1;
    if (Boolean(a.firstWorkDay) !== Boolean(b.firstWorkDay)) return a.firstWorkDay ? -1 : 1;
    return a.number.localeCompare(b.number);
  });
```

Source: `packages/db/migrations/0034_plants_and_equipment_needs.sql:310-348` (1d5d98db9acf).

```sql
create or replace function app.plant_job_sites(p_job_ids uuid[])
returns table (
  job_id uuid,
  number text,
  status text,
  customer_name text,
  property_id uuid,
  address_line text,
  city text,
  state text,
  postal_code text,
  located boolean,
  geocode_source text,
  first_work_day date
)
language sql stable security definer set search_path = app, pg_temp as $$
  select j.id,
         j.number,
         j.status::text,
         c.display_name,
         p.id,
         p.address_line,
         p.city,
         p.state::text,
         p.postal_code,
         p.location is not null,
         p.geocode_source,
         (select min(sa.work_date) from app.schedule_assignments sa where sa.tenant_id = j.tenant_id and sa.job_id = j.id)
    from app.jobs j
    join app.leads l on l.tenant_id = j.tenant_id and l.id = j.lead_id
    join app.customers c on c.tenant_id = l.tenant_id and c.id = l.customer_id
    left join app.properties p on p.tenant_id = l.tenant_id and p.id = l.property_id
   where j.tenant_id = app.current_tenant()
     and j.id = any(p_job_ids)
     and app.is_member()
     and app.plants_reader()
$$;
revoke all on function app.plant_job_sites(uuid[]) from public;
grant execute on function app.plant_job_sites(uuid[]) to app_user;
```


#### Step 3 — Observe the durable effect

Schedule hid Crew B's job from a Crew A Foreman, but Plants and Loads responses disclosed Crew B's customer and full site address. Lesser-role and foreign-company controls did not expose the same data.

Real customer disclosure and deployed effect remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/attack-led/authority-data/run-plants-crew-site-boundary.mjs:174-197` (test source).

```javascript
  assert.ok(hasSite(sitesB.items, b.id) && hasSite(sitesB.items, a.id));

  const lesserPanel = await request('lesser-no-plants-panel', lesser.headers, `/api/plants/jobs/${b.id}`, 403);
  const lesserList = await request('lesser-no-plants-list', lesser.headers, '/api/plants/jobs', 403);
  const foreignPanel = await request('foreign-panel-denied', foreign.headers, `/api/plants/jobs/${b.id}`, 404);
  const foreignNearest = await request('foreign-nearest-denied', foreign.headers, `/api/plants/nearest?jobId=${b.id}`, 404);
  const foreignSites = await request('foreign-sites-omitted', foreign.headers, `/api/plants/sites?jobIds=${b.id}`, 200);
  assert.deepEqual(foreignSites.items, []);
  assert.equal(lesserPanel, null);
  assert.equal(lesserList, null);
  assert.equal(foreignPanel, null);
  assert.equal(foreignNearest, null);
  assert.equal(raw.providerAttempts, 0);
  raw.verdict = 'PROVED-TECHNICAL-BEHAVIOR';
  raw.conclusion = 'Two synthetic crew-scoped Foremen could not see each other\'s job in Schedule, but Plants list/direct/nearest/sites disclosed the other crew\'s customer and address. Lesser role and foreign tenant stayed denied. Business-security significance awaits the client\'s companywide Plants visibility rule.';
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

Setup: A Crew A Foreman calls the Plants or Loads reads for a same-company Crew B job hidden by Schedule.

Recorded result: Schedule hid Crew B's job from a Crew A Foreman, but Plants and Loads responses disclosed Crew B's customer and full site address. Lesser-role and foreign-company controls did not expose the same data.

Scope: Real customer disclosure and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/attack-led/authority-data/run-plants-crew-site-boundary.mjs:174-197` (test source).

```javascript
  assert.ok(hasSite(sitesB.items, b.id) && hasSite(sitesB.items, a.id));

  const lesserPanel = await request('lesser-no-plants-panel', lesser.headers, `/api/plants/jobs/${b.id}`, 403);
  const lesserList = await request('lesser-no-plants-list', lesser.headers, '/api/plants/jobs', 403);
  const foreignPanel = await request('foreign-panel-denied', foreign.headers, `/api/plants/jobs/${b.id}`, 404);
  const foreignNearest = await request('foreign-nearest-denied', foreign.headers, `/api/plants/nearest?jobId=${b.id}`, 404);
  const foreignSites = await request('foreign-sites-omitted', foreign.headers, `/api/plants/sites?jobIds=${b.id}`, 200);
  assert.deepEqual(foreignSites.items, []);
  assert.equal(lesserPanel, null);
  assert.equal(lesserList, null);
  assert.equal(foreignPanel, null);
  assert.equal(foreignNearest, null);
  assert.equal(raw.providerAttempts, 0);
  raw.verdict = 'PROVED-TECHNICAL-BEHAVIOR';
  raw.conclusion = 'Two synthetic crew-scoped Foremen could not see each other\'s job in Schedule, but Plants list/direct/nearest/sites disclosed the other crew\'s customer and address. Lesser role and foreign tenant stayed denied. Business-security significance awaits the client\'s companywide Plants visibility rule.';
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

Apply the approved crew or whole-schedule visibility rule consistently across Plants and Loads read projections.
