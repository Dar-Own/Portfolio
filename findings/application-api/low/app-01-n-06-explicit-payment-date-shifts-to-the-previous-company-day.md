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

# Explicit payment date shifts to the previous company day

## Description

A manually selected payment date can fall on the previous company calendar day, affecting period grouping and business reporting.

Prerequisite: A supplied calendar date is converted under a different timezone from the company’s business timezone.

Recorded result: Original and final APIs reproduced the date shift. Patched controls cover timezone differences, daylight-saving and calendar boundaries, omitted dates, retainage, amounts and unchanged online-provider timestamps.

Evidence limits: The correction applies to future writes. A timestamp alone cannot establish the original user-selected date for historical records, so historical errors remain unconfirmed; no historical data repair is established.

### Mechanism step by step

#### Step 1 — A user supplies a calendar date for a manual payment

A supplied calendar date is converted under a different timezone from the company’s business timezone.

Source: `apps/api/src/features/invoices/service.ts:830-855` (1d5d98db9acf).

```typescript
export async function recordPayment(ctx: DbContext, invoiceId: string, input: RecordPaymentInput, today: string): Promise<void> {
  const invoice = await requireInvoice(ctx, invoiceId, today);
  if (invoice.status === 'draft') throw invalid('Send the invoice before recording a payment.');
  if (invoice.status === 'void') throw invalid('This invoice is void.');
  const amount = Math.trunc(input.amountCents);
  if (!Number.isFinite(amount) || amount <= 0) throw invalid('Enter the amount received.');
  // The day the money came in: a calendar date in the company's own time zone, inside the window
  // between the invoice date and today. Left out, it is today.
  const receivedOn = input.receivedOn?.trim() || null;
  if (receivedOn !== null) {
    if (!isCalendarDate(receivedOn)) throw invalid('Pick the day the money came in.');
    if (receivedOn > today) throw invalid("Pick the day the money came in; it can't be in the future.");
    if (invoice.issuedOn && receivedOn < invoice.issuedOn) {
      throw invalid(
        `Invoice ${invoice.number} is dated ${formatInvoiceDate(invoice.issuedOn, { weekday: false })}, so a payment can't be dated before that.`,
      );
    }
  }
  await ctx.client.query(
    `insert into app.payments (id, tenant_id, invoice_id, amount_cents, method, is_retainage, reference, received_at)
     values ($1, $2, $3, $4, $5, $6, $7, coalesce($8::date::timestamptz, $9::timestamptz))`,
    [
      randomUUID(),
      ctx.tenantId,
      invoiceId,
      amount,
```

#### Step 2 — Timestamp conversion and company-local rendering disagree

The explicit-date branch uses the wrong timezone interpretation when writing the payment timestamp.

Source: `apps/api/src/features/invoices/service.ts:379-390` (1d5d98db9acf).

```typescript
    method: string;
    is_retainage: boolean;
    provider_payment_id: string | null;
    reference: string | null;
    received_at: Date;
    received_on: string;
    recorded_by_name: string | null;
  }>(
    `select p.id, p.amount_cents, p.method, p.is_retainage, p.provider_payment_id, p.reference, p.received_at,
            (p.received_at at time zone t.timezone)::date::text as received_on,
            u.full_name as recorded_by_name
       from app.payments p
```

#### Step 3 — Observe the effect and its correction

Original and final APIs reproduced the date shift. Patched controls cover timezone differences, daylight-saving and calendar boundaries, omitted dates, retainage, amounts and unchanged online-provider timestamps.

Convert explicit dates using trusted tenant timezone data in recordPayment. Keep the actual-timestamp path for omitted dates and expose historical candidates through a read-only inventory.

Regression test for the corrected behavior:

Source: `apps/api/src/features/invoices/payment-dates.test.ts:24-43` (corrected implementation).

```typescript
  test(`N-06: explicit calendar dates round trip in ${companyZone}, database ${serverZone}`, async () => {
    query(superuserLogin(), 'postgres', `alter database ${ident(database)} set timezone to ${literal(serverZone!)}`);
    sql(`update app.tenants set timezone = ${literal(companyZone!)} where id = '${tenant}'`);
    const onlineBefore = sql("select coalesce(jsonb_agg(to_jsonb(p) order by id)::text,'[]') from app.payments p where provider_payment_id is not null");
    const app = await createTestApp({ database });
    try {
      const { headers } = await login(app, '[redacted email]', TEST_DEMO_PASSWORD);
      for (const receivedOn of ['2026-01-01', '2026-03-08', '2026-03-31', '2026-04-01', '2026-11-01', '2026-12-31']) {
        const reference = `N06 ${serverZone}/${companyZone}/${receivedOn}`;
        const res = await app.inject({ method: 'POST', url: `/api/invoices/${invoice}/payments`, headers: { ...headers, 'x-[redacted name]': '1' }, remoteAddress: testIp(), payload: { method: 'check', amountCents: 1, receivedOn, reference } });
        assert.equal(res.statusCode, 200, res.body);
        const payment = res.json<{ payments: Payment[] }>().payments.find(p => p.reference === reference)!;
        assert.equal(payment.receivedOn, receivedOn);
        assert.equal(payment.amountCents, 1);
      }
      const noDate = await app.inject({ method: 'POST', url: `/api/invoices/${invoice}/payments`, headers: { ...headers, 'x-[redacted name]': '1' }, remoteAddress: testIp(), payload: { method: 'cash', amountCents: 1, reference: `N06 omitted ${serverZone}/${companyZone}` } });
      assert.equal(noDate.statusCode, 200, noDate.body);
      const actual = noDate.json<{ payments: Payment[] }>().payments.find(p => p.reference === `N06 omitted ${serverZone}/${companyZone}`)!;
      assert.ok(Math.abs(Date.parse(actual.receivedAt) - Date.parse('2027-01-05T12:00:00Z')) < 120_000, 'omitted date retains the actual request instant');
      assert.equal(sql("select coalesce(jsonb_agg(to_jsonb(p) order by id)::text,'[]') from app.payments p where provider_payment_id is not null"), onlineBefore);
```

## Proof of concept

The excerpts below test the corrected behavior. They support the remediation within the tested conditions, rather than independently demonstrating the original vulnerability.

Setup: A supplied calendar date is converted under a different timezone from the company’s business timezone.

Recorded result: Original and final APIs reproduced the date shift. Patched controls cover timezone differences, daylight-saving and calendar boundaries, omitted dates, retainage, amounts and unchanged online-provider timestamps.

Scope: The correction applies to future writes. A timestamp alone cannot establish the original user-selected date for historical records, so historical errors remain unconfirmed; no historical data repair is established.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Regression test: `apps/api/src/features/invoices/payment-dates.test.ts`.

Source: `apps/api/src/features/invoices/payment-dates.test.ts:24-43` (corrected implementation).

```typescript
  test(`N-06: explicit calendar dates round trip in ${companyZone}, database ${serverZone}`, async () => {
    query(superuserLogin(), 'postgres', `alter database ${ident(database)} set timezone to ${literal(serverZone!)}`);
    sql(`update app.tenants set timezone = ${literal(companyZone!)} where id = '${tenant}'`);
    const onlineBefore = sql("select coalesce(jsonb_agg(to_jsonb(p) order by id)::text,'[]') from app.payments p where provider_payment_id is not null");
    const app = await createTestApp({ database });
    try {
      const { headers } = await login(app, '[redacted email]', TEST_DEMO_PASSWORD);
      for (const receivedOn of ['2026-01-01', '2026-03-08', '2026-03-31', '2026-04-01', '2026-11-01', '2026-12-31']) {
        const reference = `N06 ${serverZone}/${companyZone}/${receivedOn}`;
        const res = await app.inject({ method: 'POST', url: `/api/invoices/${invoice}/payments`, headers: { ...headers, 'x-[redacted name]': '1' }, remoteAddress: testIp(), payload: { method: 'check', amountCents: 1, receivedOn, reference } });
        assert.equal(res.statusCode, 200, res.body);
        const payment = res.json<{ payments: Payment[] }>().payments.find(p => p.reference === reference)!;
        assert.equal(payment.receivedOn, receivedOn);
        assert.equal(payment.amountCents, 1);
      }
      const noDate = await app.inject({ method: 'POST', url: `/api/invoices/${invoice}/payments`, headers: { ...headers, 'x-[redacted name]': '1' }, remoteAddress: testIp(), payload: { method: 'cash', amountCents: 1, reference: `N06 omitted ${serverZone}/${companyZone}` } });
      assert.equal(noDate.statusCode, 200, noDate.body);
      const actual = noDate.json<{ payments: Payment[] }>().payments.find(p => p.reference === `N06 omitted ${serverZone}/${companyZone}`)!;
      assert.ok(Math.abs(Date.parse(actual.receivedAt) - Date.parse('2027-01-05T12:00:00Z')) < 120_000, 'omitted date retains the actual request instant');
      assert.equal(sql("select coalesce(jsonb_agg(to_jsonb(p) order by id)::text,'[]') from app.payments p where provider_payment_id is not null"), onlineBefore);
```
