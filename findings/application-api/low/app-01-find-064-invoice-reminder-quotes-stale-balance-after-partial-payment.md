# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low (customer communication integrity)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Invoice reminder quotes stale balance after partial payment

## Description

An invoice reminder can quote the old amount due after a partial payment reduces the balance.

Prerequisite and affected workflow: The reminder worker scans a $100 balance, then a payment reduces due to $60 before the reminder is claimed and sent.

Established observation: The reminder worker read a $100 balance, then a payment reduced the amount due to $60 before claim and message handoff. It still handed a $100 reminder to the email adapter; a fully paid invoice was suppressed.

Limits: Actual external mail and an overcharge remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

The reminder worker scans a $100 balance, then a payment reduces due to $60 before the reminder is claimed and sent.

Source: `apps/worker/src/jobs/invoice-reminders.ts:247-292` (1d5d98db9acf).

```typescript
async function claimAndQueue(client: Client, input: QueueInput): Promise<boolean> {
  const { tenantId, invoice, kind, to, today, now, onlinePayments } = input;
  const hash = createHash('sha256').update(randomBytes(32)).digest();
  const expires = new Date(now.getTime() + PAY_LINK_DAYS * 24 * 60 * 60 * 1000);

  const claim = await client.query<{ claimed: boolean }>('select app.claim_invoice_reminder($1, $2, $3::date, $4, $5, $6, $7) as claimed', [
    invoice.invoice_id,
    kind,
    today,
    now,
    to,
    hash,
    expires,
  ]);
  if (claim.rows[0]?.claimed !== true) return false;

  const brand = { name: invoice.brand_name ?? invoice.company_name, phone: invoice.brand_phone ?? '' };
  const shared = {
    firstName: firstNameOf(invoice.contact_name, invoice.customer_name),
    brand,
    invoiceNumber: invoice.number,
    address: invoice.address,
    amountDueCents: invoice.due_now_cents,
    dueOn: formatInvoiceDate(invoice.due_on),
    payUrl: onlinePayments ? PAY_LINK_PLACEHOLDER : null,
  };
  const late = daysPastDue(invoice.due_on, today) ?? 0;
  const message =
    kind === 'reminder'
      ? invoiceReminderEmail({ ...shared, daysUntilDue: Math.max(0, -late) })
      : invoiceOverdueEmail({ ...shared, daysPastDue: Math.max(1, late) });

  const messageId = randomUUID();
  const queued = await client.query(
    `insert into app.messages (id, tenant_id, customer_id, lead_id, channel, direction, to_address, from_address,
                               template_key, body, status, created_by)
     values ($1, $2, $3, $4, 'email', 'outbound', $5, $6, $7, $8, 'queued', app.current_user_id())`,
    [messageId, tenantId, invoice.customer_id, invoice.lead_id, to, invoice.brand_email ?? to, message.templateKey, message.text],
  );
  if (queued.rowCount !== 1) throw new Error(`The ${kind} email for invoice ${invoice.number} could not be queued.`);
  if (onlinePayments) {
    const link = await client.query<{ id: string | null }>('select app.invoice_pay_token_id($1) as id', [hash]);
    const linkId = link.rows[0]?.id;
    if (!linkId) throw new Error(`The pay link for invoice ${invoice.number} was not made by the claim.`);
    await client.query('select app.await_message_link($1, $2, $3)', [messageId, 'pay', linkId]);
  }
```

#### Step 2 — Cross the missing state or authority check

The reminder retains the balance from its earlier invoice scan instead of checking the current amount due before sending.

Source: `packages/integrations/src/templates-invoices.ts:79-105` (1d5d98db9acf).

```typescript
export function invoiceReminderEmail(p: InvoiceEmailInput & { daysUntilDue: number }): InvoiceEmailMessage {
  const when =
    p.daysUntilDue <= 0
      ? `is due today (${p.dueOn})`
      : `is due on ${p.dueOn}, in ${p.daysUntilDue} ${p.daysUntilDue === 1 ? 'day' : 'days'}`;
  return {
    templateKey: 'invoice_reminder',
    subject: invoiceEmailSubject('invoice_reminder', p.brand.name),
    text:
      `Hi ${p.firstName}, a friendly reminder from ${p.brand.name}: invoice ${p.invoiceNumber}${place(p.address)} ` +
      `for ${formatMoney(p.amountDueCents)} ${when}.\n\n` +
      payLine(p.payUrl) +
      `If the payment is already on its way, thank you, and please ignore this note. ` +
      `Any questions about the invoice? Just reply or give us a call.\n\n${signOff(p.brand)}`,
  };
}

/** The one overdue notice, the day after the due date (kind `Overdue notice`). Can be turned off. */
export function invoiceOverdueEmail(p: InvoiceEmailInput & { daysPastDue: number }): InvoiceEmailMessage {
  const late = `${p.daysPastDue} ${p.daysPastDue === 1 ? 'day' : 'days'}`;
  return {
    templateKey: 'invoice_overdue',
    subject: invoiceEmailSubject('invoice_overdue', p.brand.name),
    text:
      `Hi ${p.firstName}, invoice ${p.invoiceNumber} from ${p.brand.name}${place(p.address)} for ` +
      `${formatMoney(p.amountDueCents)} was due on ${p.dueOn} and is now ${late} past due.\n\n` +
      payLine(p.payUrl) +
```


#### Step 3 — Observe the durable effect

The reminder worker read a $100 balance, then a payment reduced the amount due to $60 before claim and message handoff. It still handed a $100 reminder to the email adapter; a fully paid invoice was suppressed.

Actual external mail and an overcharge remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/pocs/run-dtc-001-014-invoice-reminder-race.mjs:107-124` (test source).

```javascript
  const queued=sql(`select to_address||'|'||status::text||'|'||replace(body,E'\\n',' ') from app.messages where tenant_id='${tenant}' and template_key='invoice_reminder' and body like '%INV-INTEGRATED%';`);
  assert.match(queued,new RegExp(`^${oldTo.replaceAll('.','\\.')}\\|queued\\|`));
  assert.match(queued,/\$100\.00/);
  assert.doesNotMatch(queued,/\$60\.00/);
  const integratedCurrent=sql(`select i.sent_to_email::text||'|'||b.due_now_cents from app.invoices i join app.invoice_balances b on b.tenant_id=i.tenant_id and b.invoice_id=i.id where i.id='${integratedInvoice}';`);
  assert.equal(integratedCurrent,`${newTo}|6000`);
  const {deliverOutbox}=await import('../../apps/worker/src/jobs/outbox-delivery.ts');
  const sendResult=await deliverOutbox(ctx);
  assert.equal(sendResult.errors,0);
  assert.equal(captured.length,1);
  assert.equal(captured[0].to,oldTo);
  assert.match(captured[0].text,/\$100\.00/);
  assert.equal(sql(`select status::text from app.messages where tenant_id='${tenant}' and template_key='invoice_reminder' and body like '%INV-INTEGRATED%';`),'sent');
  console.log(JSON.stringify({revision,migrations:migrations.length,isolatedDataDirectoryVerified:true,realPostgis:false,spatialColumnsShimmed:true,
    scanned:{to:oldTo,dueNowCents:10000},current:{to:newTo,dueNowCents:6000,status:'partially_paid'},raced:{claim:raced,auditTo:oldTo,auditDueNowCents:6000},fullyPaid:{claim:paid,tokens:0},unchanged:{claim:unchanged},otherTenantAuditRows:0,
    actualWorker:{interposedAfterScanCommits:mutationCount,processed:runResult.processed,current:integratedCurrent,queuedTo:oldTo,queuedAmountCents:10000,queuedStatus:'queued'},
    fakeOutbox:{capturedCount:captured.length,to:captured[0].to,quotedAmountCents:10000,finalStatus:'sent'},externalEmailNotSent:true},null,2));
}finally{if(workerPool)await workerPool.end();if(adminPool)await adminPool.end();if(started)spawnSync(path.join(pgBin,'pg_ctl'),['-D',data,'-m','immediate','stop'],{cwd:repo,stdio:'ignore',timeout:30_000});rmSync(root,{recursive:true,force:true})}
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: The reminder worker scans a $100 balance, then a payment reduces due to $60 before the reminder is claimed and sent.

Recorded result: The reminder worker read a $100 balance, then a payment reduced the amount due to $60 before claim and message handoff. It still handed a $100 reminder to the email adapter; a fully paid invoice was suppressed.

Scope: Actual external mail and an overcharge remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/pocs/run-dtc-001-014-invoice-reminder-race.mjs:107-124` (test source).

```javascript
  const queued=sql(`select to_address||'|'||status::text||'|'||replace(body,E'\\n',' ') from app.messages where tenant_id='${tenant}' and template_key='invoice_reminder' and body like '%INV-INTEGRATED%';`);
  assert.match(queued,new RegExp(`^${oldTo.replaceAll('.','\\.')}\\|queued\\|`));
  assert.match(queued,/\$100\.00/);
  assert.doesNotMatch(queued,/\$60\.00/);
  const integratedCurrent=sql(`select i.sent_to_email::text||'|'||b.due_now_cents from app.invoices i join app.invoice_balances b on b.tenant_id=i.tenant_id and b.invoice_id=i.id where i.id='${integratedInvoice}';`);
  assert.equal(integratedCurrent,`${newTo}|6000`);
  const {deliverOutbox}=await import('../../apps/worker/src/jobs/outbox-delivery.ts');
  const sendResult=await deliverOutbox(ctx);
  assert.equal(sendResult.errors,0);
  assert.equal(captured.length,1);
  assert.equal(captured[0].to,oldTo);
  assert.match(captured[0].text,/\$100\.00/);
  assert.equal(sql(`select status::text from app.messages where tenant_id='${tenant}' and template_key='invoice_reminder' and body like '%INV-INTEGRATED%';`),'sent');
  console.log(JSON.stringify({revision,migrations:migrations.length,isolatedDataDirectoryVerified:true,realPostgis:false,spatialColumnsShimmed:true,
    scanned:{to:oldTo,dueNowCents:10000},current:{to:newTo,dueNowCents:6000,status:'partially_paid'},raced:{claim:raced,auditTo:oldTo,auditDueNowCents:6000},fullyPaid:{claim:paid,tokens:0},unchanged:{claim:unchanged},otherTenantAuditRows:0,
    actualWorker:{interposedAfterScanCommits:mutationCount,processed:runResult.processed,current:integratedCurrent,queuedTo:oldTo,queuedAmountCents:10000,queuedStatus:'queued'},
    fakeOutbox:{capturedCount:captured.length,to:captured[0].to,quotedAmountCents:10000,finalStatus:'sent'},externalEmailNotSent:true},null,2));
}finally{if(workerPool)await workerPool.end();if(adminPool)await adminPool.end();if(started)spawnSync(path.join(pgBin,'pg_ctl'),['-D',data,'-m','immediate','stop'],{cwd:repo,stdio:'ignore',timeout:30_000});rmSync(root,{recursive:true,force:true})}
```

## Remediation

Read and render the current balance at reminder claim or final handoff, suppressing stale amounts.
