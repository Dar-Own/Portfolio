# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Medium (conditional)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Stripe account switch can orphan an open checkout and lose its late payment event

## Description

A payment-account switch can leave an existing checkout open under the former account while marking it canceled locally. A later payment event for that checkout can be consumed without recording payment.

Prerequisite and affected workflow: An owner changes account A to B while an A checkout remains open and a customer or callback later uses A.

Established observation: After the application switched Stripe accounts, a checkout created under account A remained open in the account-aware test provider while local state marked it canceled. A later paid event for that checkout was consumed without recording the payment in the local ledger.

Limits: Real Stripe payment and account-isolation behavior were not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

An owner changes account A to B while an A checkout remains open and a customer or callback later uses A.

Source: `apps/api/src/features/connections/routes.ts:89-122` (1d5d98db9acf).

```typescript
    app.put('/:key', { schema: { params: keyParams, body: saveBody } }, async (request) => {
      requirePermission(request.session, 'connections.manage');
      const session = request.session;
      const key = request.params.key;
      if (!isConnectionKey(key) || !(CONNECTION_KEYS as readonly string[]).includes(key)) throw notFound("That setting doesn't exist.");
      const value = request.body.value === null ? null : request.body.value.trim();
      if (value !== null) {
        const problem = connectionValueProblem(key, value);
        if (problem) throw invalid(problem);
        if (isSecretConnectionKey(key) && !app.config.secretsKey) throw new HttpError(409, 'no_master_key', MASTER_KEY_MISSING_MESSAGE);
      }

      // A fresh password check, the same as signing in (guesses count toward the lockout).
      const check = await confirmPassword(app.db, session, request.body.password, { ip: request.ip, userAgent: request.headers['user-agent'] });
      if (check === 'locked') throw locked();
      if (check !== 'ok') throw new HttpError(422, 'wrong_password', WRONG_PASSWORD_MESSAGE);

      // The owner's settings (0080, 0089): anyone else asks the owner, who gets a task with a one-click
      // approve (one request waits per setting: asking again replaces it, so nothing piles up). Which
      // settings those are is the database's answer, asked here, never a list kept in the API.
      const ownerOnly = await app.db.withUser(session, (ctx) => isOwnerOnlyChange(ctx, key, value));
      if (ownerOnly && !session.role.isOwner) {
        const id = await app.db.withUser(session, (ctx) => requestConnectionChange(ctx, key as ConnectionKey, value, app.config.secretsKey));
        request.log.info({ setting: key, removed: value === null, requestId: id }, "A company connection change was sent for the owner's approval");
        // waitingForOwner: the answer says what happened, so the page doesn't have to guess from the list of
        // waiting changes — which told the owner his own save was "sent to the owner" whenever somebody
        // else had already asked for that same setting (the review of this fix, finding 8).
        return { ...(await view(app, session)), waitingForOwner: true };
      }
      await app.db.withUser(session, (ctx) => saveConnection(ctx, key as ConnectionKey, value, app.config.secretsKey));
      request.log.info({ setting: key, removed: value === null }, 'A company connection setting was changed');
      return { ...(await view(app, session)), waitingForOwner: false };
    });

```

#### Step 2 — Cross the missing state or authority check

The connection route replaces the current credential without binding or migrating existing sessions (`apps/api/src/features/connections/routes.ts:89-122`). Local `payment_sessions` store provider/session ID but not the originating account (`packages/db/migrations/0023_invoicing_payments_reminders.sql:204-228`). Checkout jobs resolve current credentials (`apps/worker/src/jobs/checkout-requests.ts:70-95`); pending-session closure runs with that provider (`apps/api/src/features/invoices/checkout.ts:353-385,480-505`) and treats account-mismatched 404 as ended/absent (`packages/integrations/src/stripe.ts:337-357`). The webhook worker also retrieves through the current account and can consume the callback as `checkout_gone` (`apps/worker/src/jobs/webhook-inbox.ts:437-484`).

Source: `packages/db/migrations/0023_invoicing_payments_reminders.sql:204-228` (1d5d98db9acf).

```sql
create table app.payment_sessions (
  id                  uuid primary key default gen_random_uuid(),
  tenant_id           uuid not null references app.tenants(id) on delete cascade,
  invoice_id          uuid not null,
  provider            text not null check (provider ~ '^[a-z][a-z0-9_]{1,30}$'),
  provider_session_id text not null check (length(provider_session_id) between 8 and 200),
  amount_cents        bigint not null check (amount_cents > 0),
  status              text not null default 'pending' check (status in ('pending', 'paid', 'failed', 'canceled')),
  method              text check (method in ('card', 'ach')),
  created_at          timestamptz not null default now(),
  completed_at        timestamptz,
  unique (tenant_id, id),
  unique (provider, provider_session_id),
  check ((status = 'pending') = (completed_at is null)),
  check (status <> 'paid' or method is not null),
  foreign key (tenant_id, invoice_id) references app.invoices (tenant_id, id)
);
create index payment_sessions_invoice_idx on app.payment_sessions (tenant_id, invoice_id, created_at desc);

comment on table app.payment_sessions is
  'Hosted checkout attempts. No card data is ever stored: the provider takes the payment on its own page.';

alter table app.payment_sessions enable row level security;
alter table app.payment_sessions force row level security;
create policy tenant_isolation on app.payment_sessions
```

Source: `apps/worker/src/jobs/checkout-requests.ts:70-95` (1d5d98db9acf).

```typescript
export async function paymentProviderOf(options: Pick<CheckoutJobOptions, 'masterKey' | 'config' | 'logger'>, client: Queryable, tenantId: string, tokenHash?: Buffer): Promise<PaymentProvider | null> {
  const { rows } = await client.query<StoredConnection>('select key, plain_value, value_ciphertext from app.company_connection_values()');
  const values = connectionValuesFrom(rows, tenantId, options.masterKey, (key) =>
    // The company and the setting only: never the value, never the master key.
    options.logger.error(`Company ${tenantId}: its saved ${key} can't be opened with this server's master key (AOS_SECRETS_KEY); it has to be entered again on the Connections page.`),
  );
  const stripe = stripeSettingsFrom(values);
  if (stripe) return createStripePaymentProvider(stripe, { apiBase: options.config.stripeApiBase });
  if (options.config.devTools && !options.config.production) return createTestCheckoutProvider(client, { baseUrl: options.config.publicWebUrl, tenantId, tokenHash });
  return null;
}

/** How a request failed, in the customer's words, with the code the API turns into a status. */
function failureOf(error: unknown): { code: 'invalid' | 'not_found' | 'unavailable'; message: string; refusedKey: boolean } {
  if (error instanceof HttpError) return { code: error.statusCode === 404 ? 'not_found' : error.statusCode === 422 ? 'invalid' : 'unavailable', message: error.message, refusedKey: false };
  if (error instanceof StripeCallError) {
    const refusedKey = error.status === 401 || error.status === 403;
    return refusedKey ? { code: 'invalid', message: PROVIDER_REFUSED_KEY, refusedKey } : { code: 'unavailable', message: PROVIDER_UNREACHABLE, refusedKey };
  }
  return { code: 'unavailable', message: PROVIDER_UNREACHABLE, refusedKey: false };
}

/** One request, in its company's Automation transaction: the answer goes on the row with it. */
async function fulfil(options: CheckoutJobOptions, row: RequestRow): Promise<'done' | 'failed'> {
  let provider: PaymentProvider | null = null;
  try {
```


#### Step 3 — Observe the durable effect

After the application switched Stripe accounts, a checkout created under account A remained open in the account-aware test provider while local state marked it canceled. A later paid event for that checkout was consumed without recording the payment in the local ledger.

Real Stripe payment and account-isolation behavior were not tested.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/pocs/run-stripe-account-switch-proof.mjs:194-216` (synthetic test).

```javascript
  const send = await request('POST', `/api/invoices/${company.invoice}/send`, { payload: { to: recipient } });
  expect(send, 200);
  await waitForEvents(app);
  const mail = await deliverOutbox(app);
  assert.ok(mail.emails.some((message) => message.to === recipient));
  const token = sentLink(app, 'pay', recipient).token;
  record('issued link', { recipient, tokenFingerprint: fingerprint(token), state: snapshot() });
  const summary = await request('GET', `/api/public/pay/${token}`, { actor: 'none' });
  assert.equal(summary.statusCode, 200, summary.body);
  const checkoutFirst = await request('POST', `/api/public/pay/${token}/checkout`, { actor: 'none' });
  const checkoutA = expect(checkoutFirst, 200);
  const idA = sessionId(checkoutA.checkoutUrl);
  assert.equal(sessionAccounts.get(idA), 'account-A');
  assert.equal(fake.sessions.get(idA)?.status, 'open');
  record('A checkout baseline', { sessionId: idA, remoteAccount: sessionAccounts.get(idA), remoteStatus: fake.sessions.get(idA)?.status, state: snapshot() });
  // Negative control: same account key can expire its own checkout and open a replacement.
  const sameKey = await request('POST', `/api/public/pay/${token}/checkout`, { actor: 'none' });
  const checkoutA2 = expect(sameKey, 200);
  const idA2 = sessionId(checkoutA2.checkoutUrl);
  assert.equal(fake.sessions.get(idA)?.status, 'expired');
  assert.equal(fake.sessions.get(idA2)?.status, 'open');
  record('unchanged-key expiry control', { oldRemoteStatus: fake.sessions.get(idA)?.status, replacementRemoteStatus: fake.sessions.get(idA2)?.status, state: snapshot() });
  // Supported owner + password API change, with an old A session still open.
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: An owner changes account A to B while an A checkout remains open and a customer or callback later uses A.

Recorded result: After the application switched Stripe accounts, a checkout created under account A remained open in the account-aware test provider while local state marked it canceled. A later paid event for that checkout was consumed without recording the payment in the local ledger.

Scope: Real Stripe payment and account-isolation behavior were not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/pocs/run-stripe-account-switch-proof.mjs:194-216` (synthetic test).

```javascript
  const send = await request('POST', `/api/invoices/${company.invoice}/send`, { payload: { to: recipient } });
  expect(send, 200);
  await waitForEvents(app);
  const mail = await deliverOutbox(app);
  assert.ok(mail.emails.some((message) => message.to === recipient));
  const token = sentLink(app, 'pay', recipient).token;
  record('issued link', { recipient, tokenFingerprint: fingerprint(token), state: snapshot() });
  const summary = await request('GET', `/api/public/pay/${token}`, { actor: 'none' });
  assert.equal(summary.statusCode, 200, summary.body);
  const checkoutFirst = await request('POST', `/api/public/pay/${token}/checkout`, { actor: 'none' });
  const checkoutA = expect(checkoutFirst, 200);
  const idA = sessionId(checkoutA.checkoutUrl);
  assert.equal(sessionAccounts.get(idA), 'account-A');
  assert.equal(fake.sessions.get(idA)?.status, 'open');
  record('A checkout baseline', { sessionId: idA, remoteAccount: sessionAccounts.get(idA), remoteStatus: fake.sessions.get(idA)?.status, state: snapshot() });
  // Negative control: same account key can expire its own checkout and open a replacement.
  const sameKey = await request('POST', `/api/public/pay/${token}/checkout`, { actor: 'none' });
  const checkoutA2 = expect(sameKey, 200);
  const idA2 = sessionId(checkoutA2.checkoutUrl);
  assert.equal(fake.sessions.get(idA)?.status, 'expired');
  assert.equal(fake.sessions.get(idA2)?.status, 'open');
  record('unchanged-key expiry control', { oldRemoteStatus: fake.sessions.get(idA)?.status, replacementRemoteStatus: fake.sessions.get(idA2)?.status, state: snapshot() });
  // Supported owner + password API change, with an old A session still open.
```

## Remediation

Bind each session to its originating provider account and reconcile or expire A sessions under A before completing the switch.
