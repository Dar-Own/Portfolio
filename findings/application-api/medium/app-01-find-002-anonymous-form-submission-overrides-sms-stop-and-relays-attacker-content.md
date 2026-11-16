# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Medium
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below. A narrow private-lab observation is described below; it does not establish public deployment exposure.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Anonymous form submission overrides SMS STOP and relays attacker content

## Description

An anonymous form submission can override a recipient's SMS STOP and queue a company-branded acknowledgment containing attacker text.

Prerequisite and affected workflow: A submitter with the public form key enters the stopped phone number and checks the consent box while acknowledgment automation is enabled.

Established observation: An anonymous form submission changed a synthetic recipient's SMS consent from STOP to granted and queued an acknowledgment containing attacker-supplied text. The private audit deployment reproduced the form-to-outbox path.

Limits: Captured SMS in the private lab did not establish carrier delivery. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A submitter with the public form key enters the stopped phone number and checks the consent box while acknowledgment automation is enabled.

Source: `apps/api/src/features/website-form/routes.ts:74-114` (1d5d98db9acf).

```typescript
  publicRoutes: {
    prefix: PUBLIC_WEBSITE_FORM_PATH,
    register(app) {
      // A browser posts a form as form-urlencoded fields; this scope alone takes that content type, as a flat field map.
      app.addContentTypeParser('application/x-www-form-urlencoded', { parseAs: 'string', bodyLimit: FORM_BODY_LIMIT }, (_request, body, done) => {
        done(null, Object.fromEntries(new URLSearchParams(String(body))));
      });
      // The company's website is another origin: every answer here, refusals included, may be read by it.
      app.addHook('onSend', async (_request, reply) => {
        void reply.header('Access-Control-Allow-Origin', '*');
      });
      const config = { csrf: false, rateLimit: { max: 10, timeWindow: 60_000 } };

      /** The preflight a website sending JSON makes first; a plain form post needs none. */
      app.options('/:key', { schema: { params: keyParams }, config }, async (_request, reply) => {
        void reply.header('Access-Control-Allow-Methods', 'POST, OPTIONS');
        void reply.header('Access-Control-Allow-Headers', 'content-type');
        void reply.header('Access-Control-Max-Age', '600');
        return reply.status(204).send();
      });

      app.post('/:key', { schema: { params: keyParams }, config, bodyLimit: FORM_BODY_LIMIT }, async (request) => {
        const { key } = request.params;
        if (!isWellFormedPublicToken(key)) throw notFound(FORM_NOT_CONNECTED);
        const result = websiteFormSubmission(request.body);
        if (result.kind === 'invalid') throw invalid(result.message);
        // A bot filled the field people never see: told what a person is told, and nothing is kept.
        if (result.kind === 'spam') return { received: true };
        const userAgent = request.headers['user-agent'];
        const stored = await app.db.withAuth(({ client }) =>
          client.query<{ stored: boolean }>('select app.receive_website_form($1, $2::jsonb, $3::inet, $4) as stored', [
            key,
            JSON.stringify(result.submission),
            request.ip,
            typeof userAgent === 'string' ? userAgent.slice(0, 500) : null,
          ]),
        );
        if (!stored.rows[0]?.stored) throw notFound(FORM_NOT_CONNECTED);
        request.log.info('A website form submission was stored for the worker');
        return { received: true };
      });
```

#### Step 2 — Cross the missing state or authority check

The form accepts unchecked contact ownership and recognizes attacker-supplied consent values: `apps/api/src/features/website-form/routes.ts:74-114`; `packages/shared/src/website-form.ts:113-160,170-205`. Automation appends a `web_form` grant, while `current_consents` selects the newest event regardless of the earlier STOP provenance: `apps/api/src/core/domain/leads.ts:329-337`; `packages/db/migrations/0001_initial_schema.sql:551-571`. Attacker-controlled name text enters the acknowledgment: `apps/api/src/features/intake/router.ts:60-89`; `packages/integrations/src/templates.ts:167-170,231-252`.

Source: `packages/shared/src/website-form.ts:113-160` (1d5d98db9acf).

```typescript
const consent = z.preprocess((value) => {
  if (value === true) return true;
  return typeof value === 'string' && CONSENT_WORDS.has(value.trim().toLowerCase());
}, z.boolean());

const digitCount = (value: string): number => value.replace(/\D/g, '').length;

// z.object drops the fields it does not name instead of refusing them, on purpose: this form is pasted into
// other people's websites, and their form tools add fields of their own (a form id, a page name, a captcha
// token). A field we never asked for is not a reason to lose a lead.
const submissionSchema = z
  .object({
    name: z.preprocess(cleanText, z.string({ error: 'Enter your name.' }).min(1, 'Enter your name.').max(WEBSITE_FORM_LIMITS.name, `Keep your name under ${WEBSITE_FORM_LIMITS.name} characters.`)),
    phone: z.preprocess(
      cleanText,
      z
        .string({ error: 'Enter a valid phone number, or leave it blank.' })
        .max(WEBSITE_FORM_LIMITS.phone, 'Enter a valid phone number, or leave it blank.')
        .refine((value) => PHONE_CHARACTERS.test(value) && digitCount(value) >= 7 && digitCount(value) <= 15, 'Enter a valid phone number, or leave it blank.')
        .optional(),
    ),
    email: z.preprocess(
      cleanText,
      z
        .string({ error: 'Enter a valid email address, like [redacted email].' })
        .max(WEBSITE_FORM_LIMITS.email, 'Enter a valid email address, like [redacted email].')
        .regex(EMAIL, 'Enter a valid email address, like [redacted email].')
        .optional(),
    ),
    address: text('The address', WEBSITE_FORM_LIMITS.address),
    city: text('The city', WEBSITE_FORM_LIMITS.city),
    state: z.preprocess(
      (value) => {
        const cleaned = cleanText(value);
        return typeof cleaned === 'string' ? cleaned.toUpperCase() : cleaned;
      },
      z.string({ error: 'Enter the two-letter state, like GA.' }).regex(/^[A-Z]{2}$/, 'Enter the two-letter state, like GA.').optional(),
    ),
    zip: text('The ZIP code', WEBSITE_FORM_LIMITS.zip),
    message: text('Your message', WEBSITE_FORM_LIMITS.message),
    service: text('The service', WEBSITE_FORM_LIMITS.service),
    sms_consent: consent,
    utm_source: tag,
    utm_medium: tag,
    utm_campaign: tag,
    page,
  })
  .refine((fields) => fields.phone !== undefined || fields.email !== undefined, {
```

Source: `apps/api/src/core/domain/leads.ts:329-337` (1d5d98db9acf).

```typescript
  if (input.smsConsent && phone) {
    await ctx.client.query(
      `insert into app.consent_events (tenant_id, channel, address, granted, method, evidence, recorded_by)
       values ($1, 'sms', $2, true, $3, $4, $5)`,
      [ctx.tenantId, phone, input.consentMethod ?? consentMethodFor(input.source), `lead ${leadId}`, ctx.userId],
    );
  }

  ctx.emit('lead.created', leadId);
```


#### Step 3 — Observe the durable effect

An anonymous form submission changed a synthetic recipient's SMS consent from STOP to granted and queued an acknowledgment containing attacker-supplied text. The private audit deployment reproduced the form-to-outbox path.

Captured SMS in the private lab did not establish carrier delivery.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/pocs/run-leads-003-010.mjs:349-372` (test source).

```javascript
async function poc004ConsentRelay(pg, setup) {
  const templatesUrl = pathToFileURL(path.join(repo, 'packages/integrations/src/templates.ts')).href;
  const { firstNameOf, leadAcknowledgmentSms } = await import(templatesUrl);
  const websiteFormSource = readFileSync(path.join(repo, 'packages/shared/src/website-form.ts'), 'utf8');
  const workerSource = readFileSync(path.join(repo, 'apps/worker/src/jobs/website-forms.ts'), 'utf8');
  assert.match(websiteFormSource, /new Set\(\['on', 'true', '1', 'yes'\]\)/);
  assert.match(workerSource, /smsConsent: submission\.smsConsent && phone !== null/);

  const tenant = sqlLiteral(setup.tenantId);
  const victim = '+14705550199';
  const company = '+14705550100';
  pg.psql(`
    insert into app.messages (tenant_id, channel, direction, to_address, from_address, body, status)
    values (${tenant}::uuid, 'sms', 'inbound', ${sqlLiteral(company)}, ${sqlLiteral(victim)}, 'STOP', 'received');
  `);
  const stopped = pg.psql(`
    select granted || '|' || method from app.current_consents
     where tenant_id = ${tenant}::uuid and channel = 'sms' and address = ${sqlLiteral(victim)};
  `).stdout;
  assert.equal(stopped, 'false|sms_keyword');

  const blocked = pg.psql(`
    insert into app.messages (tenant_id, channel, direction, to_address, from_address, body)
    values (${tenant}::uuid, 'sms', 'outbound', ${sqlLiteral(victim)}, ${sqlLiteral(company)}, 'negative control. Reply STOP to opt out.')
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A submitter with the public form key enters the stopped phone number and checks the consent box while acknowledgment automation is enabled.

Recorded result: An anonymous form submission changed a synthetic recipient's SMS consent from STOP to granted and queued an acknowledgment containing attacker-supplied text. The private audit deployment reproduced the form-to-outbox path.

Scope: Captured SMS in the private lab did not establish carrier delivery. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/pocs/run-leads-003-010.mjs:349-372` (test source).

```javascript
async function poc004ConsentRelay(pg, setup) {
  const templatesUrl = pathToFileURL(path.join(repo, 'packages/integrations/src/templates.ts')).href;
  const { firstNameOf, leadAcknowledgmentSms } = await import(templatesUrl);
  const websiteFormSource = readFileSync(path.join(repo, 'packages/shared/src/website-form.ts'), 'utf8');
  const workerSource = readFileSync(path.join(repo, 'apps/worker/src/jobs/website-forms.ts'), 'utf8');
  assert.match(websiteFormSource, /new Set\(\['on', 'true', '1', 'yes'\]\)/);
  assert.match(workerSource, /smsConsent: submission\.smsConsent && phone !== null/);

  const tenant = sqlLiteral(setup.tenantId);
  const victim = '+14705550199';
  const company = '+14705550100';
  pg.psql(`
    insert into app.messages (tenant_id, channel, direction, to_address, from_address, body, status)
    values (${tenant}::uuid, 'sms', 'inbound', ${sqlLiteral(company)}, ${sqlLiteral(victim)}, 'STOP', 'received');
  `);
  const stopped = pg.psql(`
    select granted || '|' || method from app.current_consents
     where tenant_id = ${tenant}::uuid and channel = 'sms' and address = ${sqlLiteral(victim)};
  `).stdout;
  assert.equal(stopped, 'false|sms_keyword');

  const blocked = pg.psql(`
    insert into app.messages (tenant_id, channel, direction, to_address, from_address, body)
    values (${tenant}::uuid, 'sms', 'outbound', ${sqlLiteral(victim)}, ${sqlLiteral(company)}, 'negative control. Reply STOP to opt out.')
```

Source: `security-audit/pocs/run-leads-003-010.mjs:387-401` (test source).

```javascript

  const attackerName = '[redacted URL]';
  const firstName = firstNameOf(null, attackerName);
  const message = leadAcknowledgmentSms({ firstName, brand: { name: 'PoC Paving', phone: company }, followUpMinutes: 120 }).body;
  assert.match(message, /https:\/\/attacker\.invalid/);
  const queued = pg.psql(`
    insert into app.messages (tenant_id, channel, direction, to_address, from_address, template_key, body)
    values (${tenant}::uuid, 'sms', 'outbound', ${sqlLiteral(victim)}, ${sqlLiteral(company)}, 'lead_acknowledgment', ${sqlLiteral(message)})
    returning status;
  `).stdout;
  assert.equal(queued, 'queued');
  return {
    verdict: 'mechanism-proved',
    beforeAnonymousGrant: { consent: stopped, outboundStatus: blocked },
    afterAnonymousGrant: { consent: granted, outboundStatus: queued },
```

## Remediation

Preserve STOP precedence until the phone holder completes verified re-consent, and exclude attacker links from automatic acknowledgments.
