# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Medium
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy
Evidence: Source inspection and recorded regression tests for corrected behavior; these excerpts are not a reproduction of the original failure.
Remediation: Resolved in reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Public setup validation amplifies memory use

## Description

An unauthenticated setup request can consume substantial API memory and CPU, reducing capacity available to business users.

Prerequisite: A malformed brands array fits under the 1 MiB body cap and reaches per-element validation, including after the company is configured.

Recorded result: A 1,020,290-byte local request returned 422 after 1,424 ms while RSS rose from 217.5 MB to 681.9 MB. The patched control returned 422 in 36 ms with RSS rising from 212.0 MB to 250.7 MB; health remained 200 and the fourth request hit the existing rate limit.

Evidence limits: The assessment rated this issue Medium. The recorded result is controlled local resource amplification; no crash, production outage or capacity SLA is established. Existing request body and rate limits constrain the trigger.

### Mechanism step by step

#### Step 1 — An unauthenticated caller supplies a large malformed collection

A malformed brands array fits under the 1 MiB body cap and reaches per-element validation, including after the company is configured.

Source: `apps/api/src/features/welcome/routes.ts:50-58` (1d5d98db9acf).

```typescript
const setupBody = z.strictObject({
  company: companyBody,
  brands: z
    .array(brandBody, { error: E.brands })
    .min(1, E.brands)
    .max(10, 'Set up at most 10 brands here; more can be added in Settings later.')
    .refine((brands) => new Set(brands.map((brand) => brand.name.trim())).size === brands.length, { error: E.brandNames }),
  owner: ownerBody,
});
```

#### Step 2 — Element validation allocates work before the collection guard stops setup

The collection-length schema does not prevent expensive element validation and error allocation before setup eligibility is checked.

Source: `apps/api/src/features/welcome/routes.ts:70-89` (1d5d98db9acf).

```typescript
      // Every /login visit asks; the end-to-end run raises the public limit for its many sign-ins.
      const statusRateLimit = { max: Math.max(60, config.rateLimits.publicPerMinute), timeWindow: 60_000 };
      const setupRateLimit = { max: 3, timeWindow: 60_000 };

      app.get('/status', { config: { rateLimit: statusRateLimit } }, async () => ({ needsSetup: await needsSetup(db) }));

      app.post('/', { schema: { body: setupBody }, config: { rateLimit: setupRateLimit } }, async (request, reply) => {
        const { company, brands, owner } = request.body;
        // The install's [redacted name] support person is on the team from the start (AOS_STAFF_EMAIL), until the owner takes them off.
        const { signIn } = await setUpFirstCompany(db, config.devTools, {
          company: { ...company, homeState: company.homeState || undefined },
          brands,
          owner,
          request: {
            userAgent: request.headers['user-agent'],
            ip: request.ip,
            oldSessionToken: request.cookies[SESSION_COOKIE],
            deviceToken: request.cookies[DEVICE_COOKIE],
          },
        }, (ownerSession, ctx) => invite[redacted name]Staff(app, ownerSession, ctx));
```

#### Step 3 — Observe the effect and its correction

A 1,020,290-byte local request returned 422 after 1,424 ms while RSS rose from 217.5 MB to 681.9 MB. The patched control returned 422 in 36 ms with RSS rising from 212.0 MB to 250.7 MB; health remained 200 and the fourth request hit the existing rate limit.

Reject oversized brands and county collections before element validation and pin the patched Zod 4.6.6 dependency. Preserve existing field validation, body limits and rate limits.

Regression test for the corrected behavior:

Source: `apps/api/src/features/welcome/collection-bounds.test.ts:6-20` (corrected implementation).

```typescript
for (const [field, limit] of [['brands', 10], ['serviceCounties', 200]] as const) {
  test(`setup rejects oversized ${field} without visiting its elements`, () => {
    const values = new Array(340_000);
    Object.defineProperty(values, 0, { get() { throw new Error('element validation must not run'); } });
    const payload = field === 'brands' ? { brands: values } : { company: { serviceCounties: values } };
    assert.throws(() => boundSetupCollections(payload), (e: unknown) => e instanceof HttpError && e.statusCode === 422 && e.code === 'invalid');
    values.length = limit;
    assert.doesNotThrow(() => boundSetupCollections(payload), 'in-bound elements remain for ordinary schema validation');
  });
}
test('setup collection guard leaves missing, malformed and scalar values for the schema', () => {
  for (const body of [null, 1, 'text', [], {}, { brands: null }, { brands: 'invalid' }, { company: null }, { company: { serviceCounties: {} } }]) {
    assert.doesNotThrow(() => boundSetupCollections(body));
  }
});
```

## Proof of concept

The excerpts below test the corrected behavior. They support the remediation within the tested conditions, rather than independently demonstrating the original vulnerability.

Setup: A malformed brands array fits under the 1 MiB body cap and reaches per-element validation, including after the company is configured.

Recorded result: A 1,020,290-byte local request returned 422 after 1,424 ms while RSS rose from 217.5 MB to 681.9 MB. The patched control returned 422 in 36 ms with RSS rising from 212.0 MB to 250.7 MB; health remained 200 and the fourth request hit the existing rate limit.

Scope: The assessment rated this issue Medium. The recorded result is controlled local resource amplification; no crash, production outage or capacity SLA is established. Existing request body and rate limits constrain the trigger.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Regression test: `apps/api/src/features/welcome/collection-bounds.test.ts`.

Source: `apps/api/src/features/welcome/collection-bounds.test.ts:6-20` (corrected implementation).

```typescript
for (const [field, limit] of [['brands', 10], ['serviceCounties', 200]] as const) {
  test(`setup rejects oversized ${field} without visiting its elements`, () => {
    const values = new Array(340_000);
    Object.defineProperty(values, 0, { get() { throw new Error('element validation must not run'); } });
    const payload = field === 'brands' ? { brands: values } : { company: { serviceCounties: values } };
    assert.throws(() => boundSetupCollections(payload), (e: unknown) => e instanceof HttpError && e.statusCode === 422 && e.code === 'invalid');
    values.length = limit;
    assert.doesNotThrow(() => boundSetupCollections(payload), 'in-bound elements remain for ordinary schema validation');
  });
}
test('setup collection guard leaves missing, malformed and scalar values for the schema', () => {
  for (const body of [null, 1, 'text', [], {}, { brands: null }, { brands: 'invalid' }, { company: null }, { company: { serviceCounties: {} } }]) {
    assert.doesNotThrow(() => boundSetupCollections(body));
  }
});
```
