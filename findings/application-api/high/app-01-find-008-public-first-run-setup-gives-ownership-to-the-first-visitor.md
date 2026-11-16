# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: High
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below. A narrow private-lab observation is described below; it does not establish public deployment exposure.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Public first-run setup gives ownership to the first visitor

## Description

The first untrusted visitor to an empty installation can create the company, complete MFA, and obtain an owner session.

Prerequisite and affected workflow: An empty installation exposes the public welcome setup route before the intended operator completes bootstrap.

Established observation: The first visitor to a fresh installation completed public setup, enrolled MFA, and obtained an authenticated owner session. The private audit deployment reproduced that takeover path with development sign-in disabled.

Limits: The future public AWS setup window was not observed. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

An empty installation exposes the public welcome setup route before the intended operator completes bootstrap.

Source: `apps/api/src/features/welcome/routes.ts:66-92` (1d5d98db9acf).

```typescript
  publicRoutes: {
    prefix: '/api/public/welcome',
    register(app) {
      const { db, config } = app;
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
        // The owner's Session, or { secondStep: 'enroll' }: a new company requires two-factor sign-in.
        return reply.status(201).send(answerSignIn(reply, request, signIn));
      });
```

#### Step 2 — Cross the missing state or authority check

The welcome status and setup routes require no session or bootstrap credential: `apps/api/src/features/welcome/routes.ts:66-92`; `apps/api/src/features/welcome/setup.ts:75-130`. The database serializes callers but intentionally makes the first one authoritative: `packages/db/migrations/0100_verified_identity.sql:757-761`. The installer warns that the first visitor becomes owner in public mode: `deploy/install.sh:132-155,275-300`.

Source: `apps/api/src/features/welcome/setup.ts:75-130` (1d5d98db9acf).

```typescript
/**
 * Creates the first company and signs its owner in, all in one transaction. A new company requires
 * two-factor sign-in (require_mfa's default), so the owner is usually led straight into setting it up
 * (migration 0071) before anything else opens.
 */
export async function setUpFirstCompany(
  db: Db,
  devTools: boolean,
  input: SetupInput,
  /** Runs last, in the setup's own transaction, as the new owner (the staff invite). */
  asTheOwner?: (owner: Session, ctx: DbContext) => Promise<unknown>,
): Promise<SetupResult> {
  const brands = brandRows(input.brands);
  // A quick check before the slow password hash (the function checks again, under a lock).
  if (!(await needsSetup(db))) throw setupDone();
  // A weak password needs the tick; once the company exists, its history log says so (never the password).
  const weak = checkPasswordStrength(input.owner.password, { fullName: input.owner.fullName, email: input.owner.email, companyName: input.company.name }, input.owner.acceptWeakPassword);
  const passwordHash = await hashPassword(input.owner.password);

  return db
    .withAuth(async ({ client }): Promise<SetupResult> => {
      // A session cookie the browser already had (from a wiped install, say) is ended.
      if (isWellFormedToken(input.request.oldSessionToken)) {
        await client.query('select auth.end_session($1)', [hashSessionToken(input.request.oldSessionToken)]);
      }
      const { token, ticket } = signInTokens();
      const row = await createCompany(client, input, brands, { passwordHash, weak, token, ticket });
      const tenantId = row.tenant_id;
      const signIn = await afterSignIn(client, {
        row,
        token,
        ticket,
        devTools,
        userAgent: input.request.userAgent,
        ip: input.request.ip,
        deviceToken: input.request.deviceToken,
      });
      // app.setup_first_company made the new owner the acting person for the rest of this transaction
      // (migration 0100): there is no session yet when the company requires two-factor, only this.
      const owner = 'session' in signIn ? signIn.session : await actingSession(client, devTools);
      if (!owner || owner.company.id !== tenantId) throw new Error('The owner is not a member of the new company.');
      const ctx: DbContext = {
        client,
        tenantId,
        userId: owner.user.id,
        actor: 'user',
        devTools,
        emit(name) {
          throw new Error(`emit("${name}") during the company setup: nothing listens for it there.`);
        },
      };
      await asTheOwner?.(owner, ctx);
      return { signIn };
    })
    .catch((error: unknown) => {
      throw setupError(error);
```

Source: `packages/db/migrations/0100_verified_identity.sql:757-761` (1d5d98db9acf).

```sql
  -- One at a time: two first visits at the same moment must not both create a company.
  perform pg_advisory_xact_lock(hashtext('app.setup_first_company'));
  if exists (select 1 from app.tenants t where t.suspended_at is null) then
    raise exception 'This [redacted name] already has a company set up.';
  end if;
```


#### Step 3 — Observe the durable effect

The first visitor to a fresh installation completed public setup, enrolled MFA, and obtained an authenticated owner session. The private audit deployment reproduced that takeover path with development sign-in disabled.

The future public AWS setup window was not observed.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/pocs/run-leads-003-010.mjs:315-338` (test source).

```javascript
function poc010FirstRunOwner(pg, setup) {
  const route = readFileSync(path.join(repo, 'apps/api/src/features/welcome/routes.ts'), 'utf8');
  assert.match(route, /publicRoutes:\s*\{/);
  assert.match(route, /prefix: '\/api\/public\/welcome'/);
  assert.match(route, /app\.post\('\/'/);
  const owner = pg.psql(`
    select u.email || '|' || r.name || '|' || r.is_owner || '|' || (u.disabled_at is null)
      from app.users u
      join app.memberships m on m.user_id = u.id and m.tenant_id = ${sqlLiteral(setup.tenantId)}::uuid
      join app.roles r on r.tenant_id = m.tenant_id and r.id = m.role_id
     where u.id = ${sqlLiteral(setup.userId)}::uuid;
  `).stdout;
  assert.equal(owner, '[redacted email]|CEO|true|true');
  assert.equal(pg.psql(`select needs_setup from app.setup_status();`).stdout, 'f');

  const second = pg.psql(`
    select * from auth.setup_first_company_and_sign_in(
      '{"name":"Second Paving","timezone":"America/New_York","homeState":"GA","serviceCounties":[]}'::jsonb,
      '{"fullName":"Second Owner","email":"[redacted email]","passwordHash":"$argon2id$v=19$m=19456,t=2,p=1$c2VlZHNhbHQ$c2VlZGhhc2g"}'::jsonb,
      '[{"name":"Second Paving","sendingEmail":"[redacted email]"}]'::jsonb,
      false, '${'C'.repeat(43)}', '${'D'.repeat(43)}', null, null, null, '127.0.0.2'::inet
    );
  `, { allowFailure: true });
  assert.notEqual(second.status, 0);
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: An empty installation exposes the public welcome setup route before the intended operator completes bootstrap.

Recorded result: The first visitor to a fresh installation completed public setup, enrolled MFA, and obtained an authenticated owner session. The private audit deployment reproduced that takeover path with development sign-in disabled.

Scope: The future public AWS setup window was not observed. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/pocs/run-leads-003-010.mjs:315-338` (test source).

```javascript
function poc010FirstRunOwner(pg, setup) {
  const route = readFileSync(path.join(repo, 'apps/api/src/features/welcome/routes.ts'), 'utf8');
  assert.match(route, /publicRoutes:\s*\{/);
  assert.match(route, /prefix: '\/api\/public\/welcome'/);
  assert.match(route, /app\.post\('\/'/);
  const owner = pg.psql(`
    select u.email || '|' || r.name || '|' || r.is_owner || '|' || (u.disabled_at is null)
      from app.users u
      join app.memberships m on m.user_id = u.id and m.tenant_id = ${sqlLiteral(setup.tenantId)}::uuid
      join app.roles r on r.tenant_id = m.tenant_id and r.id = m.role_id
     where u.id = ${sqlLiteral(setup.userId)}::uuid;
  `).stdout;
  assert.equal(owner, '[redacted email]|CEO|true|true');
  assert.equal(pg.psql(`select needs_setup from app.setup_status();`).stdout, 'f');

  const second = pg.psql(`
    select * from auth.setup_first_company_and_sign_in(
      '{"name":"Second Paving","timezone":"America/New_York","homeState":"GA","serviceCounties":[]}'::jsonb,
      '{"fullName":"Second Owner","email":"[redacted email]","passwordHash":"$argon2id$v=19$m=19456,t=2,p=1$c2VlZHNhbHQ$c2VlZGhhc2g"}'::jsonb,
      '[{"name":"Second Paving","sendingEmail":"[redacted email]"}]'::jsonb,
      false, '${'C'.repeat(43)}', '${'D'.repeat(43)}', null, null, null, '127.0.0.2'::inet
    );
  `, { allowFailure: true });
  assert.notEqual(second.status, 0);
```

Source: `security-audit/pocs/run-leads-003-010.mjs:329-343` (test source).

```javascript

  const second = pg.psql(`
    select * from auth.setup_first_company_and_sign_in(
      '{"name":"Second Paving","timezone":"America/New_York","homeState":"GA","serviceCounties":[]}'::jsonb,
      '{"fullName":"Second Owner","email":"[redacted email]","passwordHash":"$argon2id$v=19$m=19456,t=2,p=1$c2VlZHNhbHQ$c2VlZGhhc2g"}'::jsonb,
      '[{"name":"Second Paving","sendingEmail":"[redacted email]"}]'::jsonb,
      false, '${'C'.repeat(43)}', '${'D'.repeat(43)}', null, null, null, '127.0.0.2'::inet
    );
  `, { allowFailure: true });
  assert.notEqual(second.status, 0);
  assert.match(second.stderr, /already has a company set up/i);
  return {
    verdict: 'bootstrap-semantics-proved',
    publicRouteHasNoSessionGuard: true,
    firstCallerResult: { outcome: setup.outcome, step: setup.step, owner },
```

## Remediation

Require a single-use bootstrap secret or authenticated administrator identity and keep public ingress closed until setup is complete.
