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

# Old payment link stays valid after invoice recipient correction

## Description

A former invoice recipient's old payment link still exposes later balance changes and authorizes checkout creation after correction to B. A matching payment-confirmation event can also update local invoice state.

Prerequisite and affected workflow: A retains a genuine unexpired token after staff resend the still-open invoice to B.

Established observation: A genuine payment link issued to A remained usable after the invoice recipient changed to B: it returned the updated invoice summary and started checkout. A matching simulated paid callback then settled the invoice locally; wrong or foreign payment sessions were denied.

Limits: No real Stripe charge was made. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A retains a genuine unexpired token after staff resend the still-open invoice to B.

Source: `apps/api/src/features/invoices/routes.ts:218-244` (1d5d98db9acf).

```typescript
    // The body is optional: "Send" with no address uses the one on file.
    app.post('/:id/send', { schema: { params: idParams, body: z.strictObject({ to: email.optional() }).nullish() } }, async (request) => {
      requirePermission(request.session, 'invoices.create');
      const { id } = request.params;
      const to = request.body?.to?.toLowerCase();
      const session = request.session;

      const prepared = await app.db.withUser(session, async (ctx) => {
        const { now, today } = await clock(ctx, session.company.timezone);
        const before = await getInvoiceDetail(ctx, id, today);
        if (before.status === 'void') throw invalid('This invoice is void. Create a new invoice instead.');
        if (before.status === 'paid') throw invalid('This invoice is already paid in full.');
        const address = (to ?? before.sentToEmail ?? before.job.apEmail ?? before.customer.contactEmail ?? before.customer.email ?? '').trim().toLowerCase();
        if (!address) throw invalid("Add the customer's email address, then send the invoice.");
        if (before.status === 'draft') await markInvoiceSent(ctx, id, today, now, address);
        else await recordResend(ctx, id, address);
        // The letter itself is the door's (app.queue_invoice_email, 0106: the people who invoice hold no
        // messages.send, and the door writes the letter from the invoice, so that is all they can send),
        // and so is the pay link, only where it leads somewhere (the company's own Stripe, which the door
        // knows, or the test checkout on a developer PC, which the API says), minted by the worker as it
        // sends; the outbox delivers it, and the event lets features/pdf attach the invoice PDF to that
        // email (the outbox waits for it).
        await ctx.client.query('select app.queue_invoice_email($1, $2, $3)', [id, address, testCheckoutAvailable(app.config)]);
        ctx.emit('invoice.sent', id);
        return { invoice: await getInvoiceDetail(ctx, id, today), address };
      });
      return { invoice: prepared.invoice, to: prepared.address, emailQueued: true };
```

#### Step 2 — Cross the missing state or authority check

`POST /api/invoices/:id/send` calls `recordResend` and queues a new letter (`apps/api/src/features/invoices/routes.ts:218-244`). `recordResend` updates only `sent_to_email` (`apps/api/src/features/invoices/service.ts:905-912`). The queue door adds a new pay-token row without revoking earlier ones (`packages/db/migrations/0106_automation_leaves_the_api.sql:837-869`), and worker minting tests expiry rather than latest recipient (`:1291-1339`). The public summary joins the old token to the current invoice balance, with hash, expiry, active tenant and status checks but no latest-recipient/revocation condition (`packages/db/migrations/0023_invoicing_payments_reminders.sql:490-520`). The checkout door likewise accepts an unexpired old token when due remains (`:537-560`); public summary/checkout routes are `apps/api/src/features/invoices/routes.ts:361-372`. confirmation follows the old token/current invoice and matching checkout through `apps/api/src/features/invoices/routes.ts:377` and `apps/api/src/features/invoices/checkout.ts:372-390`.

Source: `apps/api/src/features/invoices/service.ts:905-912` (1d5d98db9acf).

```typescript
/** Records the address a re-sent invoice went to (the invoice itself doesn't change). */
export async function recordResend(ctx: DbContext, id: string, to: string): Promise<void> {
  const result = await ctx.client.query(
    `update app.invoices set sent_to_email = $2 where id = $1 and status in ('sent', 'partially_paid')`,
    [id, to],
  );
  if (result.rowCount !== 1) throw notFound(INVOICE_NOT_FOUND);
}
```

Source: `packages/db/migrations/0106_automation_leaves_the_api.sql:837-869` (1d5d98db9acf).

```sql
  -- A pay link only where it leads somewhere (0078's rule, the same one the pay page and the worker's
  -- reminders follow): its row now, its token when the worker sends.
  pay_online := exists (select 1 from app.company_connections c where c.tenant_id = company and c.key = 'stripe.secret_key')
                or coalesce(p_test_checkout, false);
  if pay_online then
    insert into app.invoice_pay_tokens (tenant_id, invoice_id, token_hash, purpose, expires_at, created_by)
    values (company, p_invoice, sha256(public.gen_random_bytes(32)), 'invoice', now() + interval '180 days', app.current_user_id())
    returning id into pay_link;
  end if;
  -- The brand's phone as people write it, (470) 555-0100, when it is a US number (templates.ts displayPhone).
  phone := inv.brand_phone;
  if phone ~ '^\+?1?[\s().-]*\d{3}[\s().-]*\d{3}[\s.-]*\d{4}$' then
    digits := regexp_replace(phone, '\D', '', 'g');
    if length(digits) = 11 and left(digits, 1) = '1' then digits := substr(digits, 2); end if;
    if length(digits) = 10 then
      phone := '(' || substr(digits, 1, 3) || ') ' || substr(digits, 4, 3) || '-' || substr(digits, 7);
    end if;
  end if;
  letter := 'Hi ' || inv.first_name || ', here is invoice ' || inv.number || ' from ' || inv.brand_name
         || case when inv.street <> '' then ' for ' || inv.street else '' end || '.' || E'\n\n'
         || 'Amount due: ' || app.money_text(inv.due_now_cents) || E'\n'
         || 'Due: ' || coalesce(to_char(inv.due_on, 'Dy, Mon FMDD, YYYY'), '—') || E'\n\n'
         || case when pay_online then 'Pay online: (your pay link)' || E'\n\n' else '' end
         || case when pay_online then 'A check or bank transfer is just as welcome' else 'To pay by check or bank transfer' end
         || ': reply to this email and we''ll send the details.'
         || case when inv.po_number <> '' then ' Your PO number ' || inv.po_number || ' is on it.' else '' end
         || case when inv.retainage_held_cents > 0 then ' ' || app.money_text(inv.retainage_held_cents) || ' retainage is held until the job is signed off.' else '' end
         || E'\n\n' || 'Thank you for your business.' || E'\n'
         || inv.brand_name || case when phone <> '' then ' · ' || phone else '' end;
  insert into app.messages (id, tenant_id, customer_id, lead_id, channel, direction, to_address, from_address, template_key, body, status, created_by)
  values (message_id, company, inv.customer_id, inv.lead_id, 'email', 'outbound', lower(btrim(p_to)), inv.from_address, 'invoice', letter, 'queued', app.current_user_id());
  if pay_online then
    insert into app.message_links (tenant_id, message_id, kind, link_id) values (company, message_id, 'pay', pay_link);
```


#### Step 3 — Observe the durable effect

A genuine payment link issued to A remained usable after the invoice recipient changed to B: it returned the updated invoice summary and started checkout. A matching simulated paid callback then settled the invoice locally; wrong or foreign payment sessions were denied.

No real Stripe charge was made.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/attack-led/public-files/run-r238-r201-issued-token-byte-door.mjs:270-297` (synthetic test).

```javascript
  const pendingB = expect(await request('POST', `/api/public/pay/${tokenB}/confirm`, { payload: { session: matching } }), 200);
  assert.equal(pendingA.outcome, 'pending'); assert.equal(pendingB.outcome, 'pending');
  out.payment.controls.pending = { oldA: pendingA.outcome, currentB: pendingB.outcome };
  fake.markPaid(wrong); fake.markPaid(crossTenant);
  const targetBeforeDenied = paymentRows(main.invoice);
  const wrongReply = await request('POST', `/api/public/pay/${tokenA}/confirm`, { payload: { session: wrong } });
  const foreignReply = await request('POST', `/api/public/pay/${tokenA}/confirm`, { payload: { session: crossTenant } });
  assert.equal(wrongReply.statusCode, 404, wrongReply.body); assert.equal(foreignReply.statusCode, 404, foreignReply.body);
  assert.deepEqual(paymentRows(main.invoice), targetBeforeDenied, 'wrong/foreign checkout changed target payment state');
  out.payment.controls.denied = { wrongInvoice: describe(wrongReply), foreignSession: describe(foreignReply), before: targetBeforeDenied, after: paymentRows(main.invoice), checkoutAnswers: checkoutAnswers(tokenA) };
  const ownWrong = expect(await request('POST', `/api/public/pay/${otherToken}/confirm`, { payload: { session: wrong } }), 200);
  const ownForeign = expect(await request('POST', `/api/public/pay/${foreignToken}/confirm`, { payload: { session: crossTenant } }), 200);
  assert.equal(ownWrong.outcome, 'recorded'); assert.equal(ownForeign.outcome, 'recorded');
  assert.equal(paymentRows(main.otherInvoice).payments, 1); assert.equal(paymentRows(foreign.invoice).payments, 1);
  assert.deepEqual(paymentRows(main.invoice), targetBeforeDenied, 'other invoices changed target payment state');
  out.payment.controls.ownInvoice = { wrongWithOwnToken: ownWrong.outcome, foreignWithOwnToken: ownForeign.outcome,
    sameTenant: paymentRows(main.otherInvoice), foreign: paymentRows(foreign.invoice), unchangedTarget: paymentRows(main.invoice) };
  out.stage = 'wrong-and-foreign-controls-complete';
  fake.markPaid(matching);
  const recorded = expect(await request('POST', `/api/public/pay/${tokenA}/confirm`, { payload: { session: matching } }), 200);
  assert.equal(recorded.outcome, 'recorded');
  const replay = expect(await request('POST', `/api/public/pay/${tokenA}/confirm`, { payload: { session: matching } }), 200);
  assert.equal(replay.outcome, 'already_recorded');
  assert.equal(paymentRows(main.invoice).payments, 1);
  const webhook = fake.checkoutCompleted(whsecMain, matching, { id: 'evt_test_r238_oldatoken000001' });
  const posted = await request('POST', '/api/public/pay/stripe/audit-r238-main', { headers: { 'content-type': 'application/json', 'stripe-signature': webhook.header }, payload: webhook.payload });
  assert.equal(posted.statusCode, 200, posted.body);
  const webhookWorker = await webhookInbox.handler(workerCtx);
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A retains a genuine unexpired token after staff resend the still-open invoice to B.

Recorded result: A genuine payment link issued to A remained usable after the invoice recipient changed to B: it returned the updated invoice summary and started checkout. A matching simulated paid callback then settled the invoice locally; wrong or foreign payment sessions were denied.

Scope: No real Stripe charge was made. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/attack-led/public-files/run-r238-r201-issued-token-byte-door.mjs:270-297` (synthetic test).

```javascript
  const pendingB = expect(await request('POST', `/api/public/pay/${tokenB}/confirm`, { payload: { session: matching } }), 200);
  assert.equal(pendingA.outcome, 'pending'); assert.equal(pendingB.outcome, 'pending');
  out.payment.controls.pending = { oldA: pendingA.outcome, currentB: pendingB.outcome };
  fake.markPaid(wrong); fake.markPaid(crossTenant);
  const targetBeforeDenied = paymentRows(main.invoice);
  const wrongReply = await request('POST', `/api/public/pay/${tokenA}/confirm`, { payload: { session: wrong } });
  const foreignReply = await request('POST', `/api/public/pay/${tokenA}/confirm`, { payload: { session: crossTenant } });
  assert.equal(wrongReply.statusCode, 404, wrongReply.body); assert.equal(foreignReply.statusCode, 404, foreignReply.body);
  assert.deepEqual(paymentRows(main.invoice), targetBeforeDenied, 'wrong/foreign checkout changed target payment state');
  out.payment.controls.denied = { wrongInvoice: describe(wrongReply), foreignSession: describe(foreignReply), before: targetBeforeDenied, after: paymentRows(main.invoice), checkoutAnswers: checkoutAnswers(tokenA) };
  const ownWrong = expect(await request('POST', `/api/public/pay/${otherToken}/confirm`, { payload: { session: wrong } }), 200);
  const ownForeign = expect(await request('POST', `/api/public/pay/${foreignToken}/confirm`, { payload: { session: crossTenant } }), 200);
  assert.equal(ownWrong.outcome, 'recorded'); assert.equal(ownForeign.outcome, 'recorded');
  assert.equal(paymentRows(main.otherInvoice).payments, 1); assert.equal(paymentRows(foreign.invoice).payments, 1);
  assert.deepEqual(paymentRows(main.invoice), targetBeforeDenied, 'other invoices changed target payment state');
  out.payment.controls.ownInvoice = { wrongWithOwnToken: ownWrong.outcome, foreignWithOwnToken: ownForeign.outcome,
    sameTenant: paymentRows(main.otherInvoice), foreign: paymentRows(foreign.invoice), unchangedTarget: paymentRows(main.invoice) };
  out.stage = 'wrong-and-foreign-controls-complete';
  fake.markPaid(matching);
  const recorded = expect(await request('POST', `/api/public/pay/${tokenA}/confirm`, { payload: { session: matching } }), 200);
  assert.equal(recorded.outcome, 'recorded');
  const replay = expect(await request('POST', `/api/public/pay/${tokenA}/confirm`, { payload: { session: matching } }), 200);
  assert.equal(replay.outcome, 'already_recorded');
  assert.equal(paymentRows(main.invoice).payments, 1);
  const webhook = fake.checkoutCompleted(whsecMain, matching, { id: 'evt_test_r238_oldatoken000001' });
  const posted = await request('POST', '/api/public/pay/stripe/audit-r238-main', { headers: { 'content-type': 'application/json', 'stripe-signature': webhook.header }, payload: webhook.payload });
  assert.equal(posted.statusCode, 200, posted.body);
  const webhookWorker = await webhookInbox.handler(workerCtx);
```

## Remediation

Revoke earlier recipient tokens on correction and enforce that revocation in public summary, checkout, and confirmation paths.
