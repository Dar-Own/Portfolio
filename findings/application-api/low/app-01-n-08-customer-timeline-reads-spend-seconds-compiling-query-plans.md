# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy
Evidence: Source inspection and recorded regression tests for corrected behavior; these excerpts are not a reproduction of the original failure.
Remediation: Resolved in reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Customer timeline reads spend seconds compiling query plans

## Description

Opening customer history becomes slow and occupies database capacity during concurrent office use.

Prerequisite: After statistics are collected, the strengthened visibility rules push the unchanged timeline query above PostgreSQL’s JIT threshold.

Recorded result: Controlled migration replay isolates the 0117 transition. The captured synthetic plan spent about 5.59 seconds compiling. Phone view with four concurrent peer reads took 9,628 ms with JIT versus 833 ms in the disabled-JIT control. The patched route completed in 1,577 ms with database JIT still enabled by default; the full browser suite passed.

Evidence limits: Policy-expanded planning and compilation cost were isolated with controlled statistics and concurrent synthetic reads. No lost data or authorization bypass was proved. Local timings do not explain every historic browser timeout.

### Mechanism step by step

#### Step 1 — An authorized user opens the customer timeline

After statistics are collected, the strengthened visibility rules push the unchanged timeline query above PostgreSQL’s JIT threshold.

Source: `apps/api/src/features/customers/routes.ts:253-261` (be12d54).

```typescript
    });

    app.get('/:id/timeline', { schema: { params: idParams, querystring: timelineQuery } }, async (request): Promise<CustomerTimelinePage> => {
      requireCustomerBook(request.session);
      return app.db.withUser(request.session, async (ctx) => {
        await requireCustomer(ctx, request.params.id);
        return loadTimeline(ctx, request.params.id, leadsReach(request.session) === 'own', request.query.cursor);
      });
    });
```

#### Step 2 — Policy-expanded query planning triggers costly JIT compilation

JIT compilation of a large policy-expanded plan dominates an interactive read. The visibility rules themselves remain necessary.

Source: `apps/api/src/features/customers/timeline.ts:111-133` (be12d54).

```typescript
    select 'thread-' || t.id, 'thread', t.last_message_at,
           jsonb_build_object('subject', t.subject, 'status', t.status::text, 'count', t.message_count, 'thread', t.id), null,
           case when t.lead_id in (select id from my_leads) then t.lead_id end
      from app.mail_threads t
     where t.tenant_id = $1
       and (t.lead_id in (select id from my_leads)
            or (t.lead_id is null and not $5::boolean
                and (t.customer_id = $2 or t.participants && array(select email::citext from my_emails))))
    union all
    select 'task-' || t.id, 'task', t.created_at, jsonb_build_object('title', t.title, 'status', t.status::text, 'owner', o.full_name), t.created_by, t.lead_id
      from app.tasks t left join app.users o on o.id = t.owner_id
     where t.tenant_id = $1 and (t.lead_id in (select id from my_leads)
                                 or t.job_id in (select j.id from app.jobs j where j.tenant_id = $1 and j.lead_id in (select id from my_leads)))
    union all
    select 'estimate-' || e.id, 'estimate', e.created_at, jsonb_build_object('number', e.number, 'cents', e.total_cents, 'status', e.status::text), e.created_by, e.lead_id
      from app.estimates e where e.tenant_id = $1 and e.lead_id in (select id from my_leads)
    union all
    select 'viewed-' || e.id, 'estimate_viewed', e.viewed_at, jsonb_build_object('number', e.number), null, e.lead_id
      from app.estimates e where e.tenant_id = $1 and e.lead_id in (select id from my_leads) and e.viewed_at is not null
    union all
    select 'answer-' || s.id, 'estimate_answer', s.created_at, jsonb_build_object('number', e.number, 'decision', s.decision, 'name', s.signed_name), null, s.lead_id
      from app.estimate_signatures s join app.estimates e on e.tenant_id = s.tenant_id and e.id = s.estimate_id
     where s.tenant_id = $1 and s.lead_id in (select id from my_leads)
```

#### Step 3 — Observe the effect and its correction

Controlled migration replay isolates the 0117 transition. The captured synthetic plan spent about 5.59 seconds compiling. Phone view with four concurrent peer reads took 9,628 ms with JIT versus 833 ms in the disabled-JIT control. The patched route completed in 1,577 ms with database JIT still enabled by default; the full browser suite passed.

Disable JIT only within the existing timeline transaction. Verify identical authorized results, restricted/foreign controls and restoration of the connection setting after success and rollback.

Regression test for the corrected behavior:

Source: `apps/api/src/features/customers/timeline-execution.test.ts:15-42` (corrected implementation).

```typescript
test('timeline disables compilation only during its read, restoring pool state on success and failure', async () => {
  const { headers, session } = await login(app, '[redacted email]', TEST_DEMO_PASSWORD);
  const ordinary = app.db.withUser.bind(app.db);
  const normalJit = await ordinary(session, async ctx => (await ctx.client.query('show jit')).rows[0].jit);
  let failQuery = false;
  let observed = 0;
  app.db.withUser = (actor, fn) => ordinary(actor, async ctx => {
    const originalQuery = ctx.client.query.bind(ctx.client);
    // Observe the native connection at execution, not the source text of SET LOCAL.
    ctx.client.query = (async (...args: Parameters<typeof originalQuery>) => {
      if (typeof args[0] === 'string' && args[0].includes('with my_leads')) {
        assert.equal((await originalQuery('show jit')).rows[0].jit, 'off');
        observed++;
        if (failQuery) throw new Error('controlled timeline query failure');
      }
      return originalQuery(...args);
    }) as typeof ctx.client.query;
    try { return await fn(ctx); } finally { ctx.client.query = originalQuery; }
  });
  try {
    for (const failure of [false, true]) {
      failQuery = failure;
      const response = await app.inject({ method: 'GET', url: `/api/customers/${customer}/timeline`, headers });
      assert.equal(response.statusCode, failure ? 500 : 200);
      assert.equal(await ordinary(session, async ctx => (await ctx.client.query('show jit')).rows[0].jit), normalJit);
    }
    assert.equal(observed, 2);
  } finally { app.db.withUser = ordinary; }
```

## Proof of concept

The excerpts below test the corrected behavior. They support the remediation within the tested conditions, rather than independently demonstrating the original vulnerability.

Setup: After statistics are collected, the strengthened visibility rules push the unchanged timeline query above PostgreSQL’s JIT threshold.

Recorded result: Controlled migration replay isolates the 0117 transition. The captured synthetic plan spent about 5.59 seconds compiling. Phone view with four concurrent peer reads took 9,628 ms with JIT versus 833 ms in the disabled-JIT control. The patched route completed in 1,577 ms with database JIT still enabled by default; the full browser suite passed.

Scope: Policy-expanded planning and compilation cost were isolated with controlled statistics and concurrent synthetic reads. No lost data or authorization bypass was proved. Local timings do not explain every historic browser timeout.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Regression test: `apps/api/src/features/customers/timeline-execution.test.ts`.

Source: `apps/api/src/features/customers/timeline-execution.test.ts:15-42` (corrected implementation).

```typescript
test('timeline disables compilation only during its read, restoring pool state on success and failure', async () => {
  const { headers, session } = await login(app, '[redacted email]', TEST_DEMO_PASSWORD);
  const ordinary = app.db.withUser.bind(app.db);
  const normalJit = await ordinary(session, async ctx => (await ctx.client.query('show jit')).rows[0].jit);
  let failQuery = false;
  let observed = 0;
  app.db.withUser = (actor, fn) => ordinary(actor, async ctx => {
    const originalQuery = ctx.client.query.bind(ctx.client);
    // Observe the native connection at execution, not the source text of SET LOCAL.
    ctx.client.query = (async (...args: Parameters<typeof originalQuery>) => {
      if (typeof args[0] === 'string' && args[0].includes('with my_leads')) {
        assert.equal((await originalQuery('show jit')).rows[0].jit, 'off');
        observed++;
        if (failQuery) throw new Error('controlled timeline query failure');
      }
      return originalQuery(...args);
    }) as typeof ctx.client.query;
    try { return await fn(ctx); } finally { ctx.client.query = originalQuery; }
  });
  try {
    for (const failure of [false, true]) {
      failQuery = failure;
      const response = await app.inject({ method: 'GET', url: `/api/customers/${customer}/timeline`, headers });
      assert.equal(response.statusCode, failure ? 500 : 200);
      assert.equal(await ordinary(session, async ctx => (await ctx.client.query('show jit')).rows[0].jit), normalJit);
    }
    assert.equal(observed, 2);
  } finally { app.db.withUser = ordinary; }
```
