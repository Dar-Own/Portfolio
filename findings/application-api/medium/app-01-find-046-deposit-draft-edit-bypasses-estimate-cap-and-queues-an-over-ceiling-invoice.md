# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Medium (conditional integrity)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Deposit draft edit bypasses estimate cap and queues an over-ceiling invoice

## Description

An edited deposit draft can exceed the estimate ceiling and be marked sent with an excess amount queued for the customer.

Prerequisite and affected workflow: An authorized invoice creator raises a deposit draft above the estimate limit, then sends it through the normal invoice workflow.

Established observation: An authorized invoice creator raised a valid deposit draft above the estimate ceiling through the edit route and sent it. The system accepted the over-ceiling amount and queued the invoice body and attachment work without applying the cap at the final send.

Limits: The client's override rule and any delivered email, pay link, settlement, or cash effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

An authorized invoice creator raises a deposit draft above the estimate limit, then sends it through the normal invoice workflow.

Source: `apps/api/src/features/invoices/service.ts:729-745` (1d5d98db9acf).

```typescript
  // raised: without this a job could be billed several times over.
  if (kind === 'deposit') {
    const already = job.deposited_cents;
    const left = job.estimate_subtotal_cents - already;
    const money = (cents: number) => formatCents(cents, { decimals: 2 });
    if (left <= 0) {
      throw invalid(
        `Job ${job.job_number} is already invoiced in deposits (${money(already)} of estimate ${job.estimate_number}). Invoice the job in full when the work is done.`,
      );
    }
    if (draft.subtotalCents > left) {
      throw invalid(
        `Job ${job.job_number} already has ${money(already)} of deposits invoiced, so only ${money(left)} of estimate ${job.estimate_number} is left to bill.`,
      );
    }
  }

```

#### Step 2 — Cross the missing state or authority check

Editing a deposit draft recalculates its subtotal without the original cap, and sending it does not check the aggregate cap again.

Source: `packages/db/migrations/0106_automation_leaves_the_api.sql:792-874` (1d5d98db9acf).

```sql
create function app.queue_invoice_email(p_invoice uuid, p_to text, p_test_checkout boolean) returns uuid
language plpgsql security definer set search_path = app, pg_temp as $$
declare
  company uuid := app.current_tenant();
  inv record;
  pay_online boolean;
  pay_link uuid;
  phone text;
  digits text;
  letter text;
  message_id uuid := gen_random_uuid();
begin
  if company is null or not app.is_member() then
    raise exception 'You are not a member of this company' using errcode = '42501';
  end if;
  if not app.can('invoices.create') then
    raise exception 'You don''t have permission to send invoices' using errcode = '42501';
  end if;
  if coalesce(btrim(p_to), '') !~ '^[^@\s]+@[^@\s]+\.[^@\s]+$' then
    raise exception 'Add a valid email address, like [redacted email].';
  end if;
  select i.customer_id, j.lead_id, i.number, i.status::text as status, i.due_on, btrim(coalesce(j.po_number, '')) as po_number,
         b.due_now_cents, coalesce(b.retainage_held_cents, 0) as retainage_held_cents,
         (regexp_split_to_array(coalesce(nullif(btrim(ct.full_name), ''), nullif(btrim(c.display_name), ''), ''), '\s+'))[1] as first_name,
         coalesce(bu.name, t.name) as brand_name, btrim(coalesce(bu.phone_e164, '')) as brand_phone,
         btrim(split_part(coalesce(p.address_line, ''), ',', 1)) as street,
         coalesce(bu.sending_email::text, first_brand.sending_email::text, '') as from_address
    into inv
    from app.invoices i
    join app.tenants t on t.id = i.tenant_id
    join app.customers c on c.tenant_id = i.tenant_id and c.id = i.customer_id
    join app.invoice_balances b on b.tenant_id = i.tenant_id and b.invoice_id = i.id
    left join app.jobs j on j.tenant_id = i.tenant_id and j.id = i.job_id
    left join app.leads l on l.tenant_id = j.tenant_id and l.id = j.lead_id
    left join app.properties p on p.tenant_id = l.tenant_id and p.id = l.property_id
    left join app.business_units bu on bu.tenant_id = j.tenant_id and bu.id = j.business_unit_id
    left join lateral (select x.sending_email from app.business_units x where x.tenant_id = i.tenant_id order by x.created_at, x.name limit 1) first_brand on true
    left join lateral (select x.full_name from app.contacts x where x.tenant_id = i.tenant_id and x.customer_id = i.customer_id order by x.is_primary desc, x.created_at limit 1) ct on true
   where i.tenant_id = company and i.id = p_invoice;
  if not found then
    raise exception 'That invoice doesn''t exist';
  end if;
  if inv.status not in ('sent', 'partially_paid') then
    raise exception 'Send the invoice before emailing it';
  end if;
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
  end if;
  return message_id;
end $$;
revoke all on function app.queue_invoice_email(uuid, text, boolean) from public;
grant execute on function app.queue_invoice_email(uuid, text, boolean) to app_user;
```


#### Step 3 — Observe the durable effect

An authorized invoice creator raised a valid deposit draft above the estimate ceiling through the edit route and sent it. The system accepted the over-ceiling amount and queued the invoice body and attachment work without applying the cap at the final send.

The client's override rule and any delivered email, pay link, settlement, or cash effect remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/attack-led/authority-data/run-invoice-edited-deposit-send.mjs:186-209` (test source).

```javascript
  assert.equal(sent.after.messages[0].status, 'queued');
  assert.equal(sent.after.messages[0].to_address, '[redacted email]');
  assert.match(sent.after.messages[0].body, /invoice INV-/);
  assert.ok(sent.after.messages[0].body.includes('Amount due:'));
  assert.equal(sent.after.payLinks, 0, 'provider-disabled production-mode lab must not mint a pay link');
  assert.ok(sent.after.events > sent.before.events, 'invoice.sent event must be durable');

  await waitForEvents(app);
  const afterEvent = invoiceEffects(invoiceId);
  raw.afterPdfEvent = afterEvent;
  assert.equal(afterEvent.attachments.length, 1);
  assert.equal(afterEvent.attachments[0].content_type, 'application/pdf');
  assert.equal(raw.providerAttempts, 0);
  raw.verdict = 'PROVED-TECHNICAL-BEHAVIOR';
  raw.conclusion = 'A single authorized R228 edit raised a valid deposit draft above its job estimate, and R230 sent it into a queued synthetic-address invoice email with a PDF attachment. No real SMTP acceptance or payment link was attempted.';
} catch (error) {
  raw.error = { name: error?.name ?? 'Error', message: String(error?.message ?? error), stack: String(error?.stack ?? '') };
  process.exitCode = 1;
} finally {
  try { await app?.close(); } catch (error) { raw.cleanupError = String(error); process.exitCode = 1; }
  if (started) {
    try { command(path.join(pgBin, 'pg_ctl'), ['-D', data, '-m', 'immediate', '-w', 'stop']); }
    catch (error) { raw.cleanupError = String(error); process.exitCode = 1; }
  }
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: An authorized invoice creator raises a deposit draft above the estimate limit, then sends it through the normal invoice workflow.

Recorded result: An authorized invoice creator raised a valid deposit draft above the estimate ceiling through the edit route and sent it. The system accepted the over-ceiling amount and queued the invoice body and attachment work without applying the cap at the final send.

Scope: The client's override rule and any delivered email, pay link, settlement, or cash effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/attack-led/authority-data/run-invoice-edited-deposit-send.mjs:186-209` (test source).

```javascript
  assert.equal(sent.after.messages[0].status, 'queued');
  assert.equal(sent.after.messages[0].to_address, '[redacted email]');
  assert.match(sent.after.messages[0].body, /invoice INV-/);
  assert.ok(sent.after.messages[0].body.includes('Amount due:'));
  assert.equal(sent.after.payLinks, 0, 'provider-disabled production-mode lab must not mint a pay link');
  assert.ok(sent.after.events > sent.before.events, 'invoice.sent event must be durable');

  await waitForEvents(app);
  const afterEvent = invoiceEffects(invoiceId);
  raw.afterPdfEvent = afterEvent;
  assert.equal(afterEvent.attachments.length, 1);
  assert.equal(afterEvent.attachments[0].content_type, 'application/pdf');
  assert.equal(raw.providerAttempts, 0);
  raw.verdict = 'PROVED-TECHNICAL-BEHAVIOR';
  raw.conclusion = 'A single authorized R228 edit raised a valid deposit draft above its job estimate, and R230 sent it into a queued synthetic-address invoice email with a PDF attachment. No real SMTP acceptance or payment link was attempted.';
} catch (error) {
  raw.error = { name: error?.name ?? 'Error', message: String(error?.message ?? error), stack: String(error?.stack ?? '') };
  process.exitCode = 1;
} finally {
  try { await app?.close(); } catch (error) { raw.cleanupError = String(error); process.exitCode = 1; }
  if (started) {
    try { command(path.join(pgBin, 'pg_ctl'), ['-D', data, '-m', 'immediate', '-w', 'stop']); }
    catch (error) { raw.cleanupError = String(error); process.exitCode = 1; }
  }
```

## Remediation

Enforce the aggregate deposit cap or an explicit approved override atomically at creation, edit, and final send.
