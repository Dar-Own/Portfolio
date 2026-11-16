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

# Queued invoice mail can go to a superseded recipient after corrected resend

## Description

After an invoice recipient changes to B, queued reminders or the original invoice email can still be handed to the email adapter addressed to A, with billing details and a payment link.

Prerequisite and affected workflow: An authorized resend to B commits while an A reminder or initial invoice letter remains unhanded.

Established observation: After an authorized invoice resend changed the recipient from A to B, previously queued invoice letters and reminders were still handed to captured mail for A with billing details and a payment link. Tests covered correction before both reminder claim and final outbox handoff.

Limits: External delivery to A was not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

An authorized resend to B commits while an A reminder or initial invoice letter remains unhanded.

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

The signed-in resend route accepts a corrected address and calls `recordResend` (`apps/api/src/features/invoices/routes.ts:218-244`), which updates `sent_to_email` (`apps/api/src/features/invoices/service.ts:905-912`). The reminder worker scans and caches its recipient and balance (`apps/worker/src/jobs/invoice-reminders.ts:153-157,188-225`), then passes the cached recipient to claim/queue (`:94-126,247-292`). Final `app.claim_invoice_reminder` locks/rechecks invoice state and balance but does not compare its caller-supplied `p_to` with current `sent_to_email` (`packages/db/migrations/0023_invoicing_payments_reminders.sql:671-729`). Separately, the ordinary send queues an A email and emits `invoice.sent` (`apps/api/src/features/invoices/routes.ts:218-244`); later correction queues B but does not cancel A. The PDF handler picks the newest queued invoice email (`apps/api/src/features/pdf/events.ts:53-67,82-91`), and the outbox sends persisted `to_address` without a current-recipient check (`apps/worker/src/jobs/outbox-delivery.ts:395-411,615-681`). Both branches fail at effect-time recipient authority.

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

Source: `apps/worker/src/jobs/invoice-reminders.ts:153-157` (1d5d98db9acf).

```typescript
/** Where a reminder goes: the address the invoice went to, then accounts payable, then the contact. */
export function recipient(invoice: Pick<Candidate, 'sent_to_email' | 'ap_email' | 'contact_email' | 'customer_email'>): string | null {
  const address = invoice.sent_to_email ?? invoice.ap_email ?? invoice.contact_email ?? invoice.customer_email;
  const trimmed = address?.trim().toLowerCase() ?? '';
  return trimmed === '' ? null : trimmed;
```


#### Step 3 — Observe the durable effect

After an authorized invoice resend changed the recipient from A to B, previously queued invoice letters and reminders were still handed to captured mail for A with billing details and a payment link. Tests covered correction before both reminder claim and final outbox handoff.

External delivery to A was not tested.

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

Setup: An authorized resend to B commits while an A reminder or initial invoice letter remains unhanded.

Recorded result: After an authorized invoice resend changed the recipient from A to B, previously queued invoice letters and reminders were still handed to captured mail for A with billing details and a payment link. Tests covered correction before both reminder claim and final outbox handoff.

Scope: External delivery to A was not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

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

Cancel unsent A letters, bind reminder claims to the current recipient, and check recipient authority before mail handoff.
