# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low (conditional financial-offer integrity)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Queued older quote email carries the revised quote PDF

## Description

An older queued quote email can attach a newer quote PDF with a conflicting price and an expired older link.

Prerequisite and affected workflow: Two authorized quote versions are sent before the queued email events generate their PDF attachments.

Established observation: Two valid quote sends were queued before attachment generation. The earlier email body quoted $9,363, but both emails received the later $9,961.70 quote PDF; the earlier signing link had expired.

Limits: Natural event overlap, real email delivery, and which document would bind the offer remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

Two authorized quote versions are sent before the queued email events generate their PDF attachments.

Source: `apps/api/src/features/estimates/estimates.ts:1003-1101` (1d5d98db9acf).

```typescript
export async function sendEstimate(ctx: DbContext, id: string): Promise<SendEstimateResult> {
  const estimate = await requireEstimate(ctx, id, { lines: false });
  if (!SENDABLE_STATUSES.includes(estimate.status)) {
    throw ruleViolation(`This estimate is ${estimateStatuses.label(estimate.status).toLowerCase()}, so it can't be sent again.`);
  }
  // The same rule the database enforces, with the reasons people can read.
  if (estimate.approvalReasons.length > 0) throw needsApprovalError(estimate.approvalReasons);
  const actor = await loadActor(ctx);

  // Sent. A Viewed estimate (migration 0076) stays Viewed only while the link the customer opened
  // still shows this version: after a change a fresh link goes out, and the customer has not seen
  // that one yet.
  const linkStillLive = await quoteLinkLive(ctx, id);
  if (estimate.status !== 'sent' && !(estimate.status === 'viewed' && linkStillLive)) {
    const result = await ctx.client.query(`update app.estimates set status = 'sent' where id = $1`, [id]);
    if (result.rowCount === 0) throw estimateNotFound();
  }

  // The lead moves to Approved (proposal sent). The stage gates may refuse (a missing screenshot,
  // tier or size): the estimate still goes out, and the screen says why the lead stayed put.
  let stageWarning: string | null = null;
  const stage = estimate.lead.stage;
  if (stage === 'new' || stage === 'sizing' || stage === 'estimating') {
    await ctx.client.query('savepoint move_stage');
    try {
      await setStage(ctx, estimate.lead.id, 'proposal_sent');
      await ctx.client.query('release savepoint move_stage');
    } catch (error) {
      await ctx.client.query('rollback to savepoint move_stage');
      stageWarning = error instanceof HttpError ? error.message : 'The lead could not move to Approved.';
    }
  }

  const brand = { name: estimate.lead.brandName, phone: estimate.lead.brandPhone ?? '' };
  const proposalFor = (url: string) =>
    proposalEmail({ brand, estimateNumber: estimate.number, address: estimate.lead.address ?? '', totalCents: estimate.totalCents, estimateUrl: url });
  const textFor = (url: string) =>
    estimateReadySms({
      firstName: firstNameOf(estimate.lead.contactName, estimate.lead.name),
      brand,
      service: estimate.service?.name ?? 'the work',
      address: estimate.lead.address ?? '',
      areaSqFt: estimate.areaSqft ?? 0,
      lowCents: estimate.lowCents,
      highCents: estimate.highCents,
      addOn: estimate.addOn ? { service: estimate.addOn.name, totalCents: estimate.addOn.totalCents } : null,
      estimateUrl: url,
    });
  // The customer's quote page (migration 0072). Each message goes out with a fresh link to the estimate
  // as it is now, minted by the worker as it sends the message (migration 0106 stage 2d): the stored
  // body holds the link's placeholder, and the link's row waits for the worker with a hash no token
  // matches. A message that already went out, with a link that still works, is not sent again.
  const proposal = proposalFor(QUOTE_LINK_PLACEHOLDER);
  const text = textFor(QUOTE_LINK_PLACEHOLDER);
  const linkLive = actor.canMessage && linkStillLive;
  // "Text estimates automatically" (Settings): the estimate text goes out only when it is on.
  const textToo = (await getPricingSettings(ctx)).autoSmsEstimates;

  const email: SendEstimateResult['email'] = { queued: false, to: null, alreadySent: false };
  const sms: SendEstimateResult['sms'] = { queued: false, to: null, status: null, blockedReason: null, alreadySent: false, textingOff: false };
  if (actor.canMessage) {
    if (estimate.lead.email) {
      email.to = estimate.lead.email;
      email.alreadySent = linkLive && (await messageAlreadySent(ctx, estimate, 'email', proposal.templateKey, proposal.text));
      if (!email.alreadySent) {
        const queued = await queueEmail(ctx, {
          to: estimate.lead.email,
          body: proposal.text,
          templateKey: proposal.templateKey,
          leadId: estimate.lead.id,
        });
        await awaitLink(ctx, queued.id, 'quote', await pendingQuoteLink(ctx, id));
        email.queued = true;
      }
    }
    if (estimate.lead.phone && !textToo) sms.textingOff = true;
    if (estimate.lead.phone && textToo) {
      sms.to = estimate.lead.phone;
      sms.alreadySent = linkLive && (await messageAlreadySent(ctx, estimate, 'sms', text.templateKey, ensureSmsOptOut(text.body)));
      if (!sms.alreadySent) {
        const queued = await queueSms(ctx, {
          to: estimate.lead.phone,
          body: text.body,
          templateKey: text.templateKey,
          leadId: estimate.lead.id,
        });
        // A text blocked for want of consent never leaves, so no link waits for it.
        if (queued.status === 'queued') await awaitLink(ctx, queued.id, 'quote', await pendingQuoteLink(ctx, id));
        sms.queued = queued.status === 'queued';
        sms.status = queued.status;
        sms.blockedReason = queued.blockedReason;
      }
    }
  }

  // The proposal PDF is attached to the email by the worker (features/pdf/events.ts).
  ctx.emit('estimate.sent', id);
  const alreadySent = !email.queued && !sms.queued && (email.alreadySent || sms.alreadySent);
  return { estimate: await requireEstimate(ctx, id), email, sms, stageWarning, alreadySent };
```

#### Step 2 — Cross the missing state or authority check

The queued send retains the earlier email body while PDF generation reads the later current quote revision.

Source: `apps/worker/src/jobs/outbox-delivery.ts:390-411` (1d5d98db9acf).

```typescript
 * Picks the oldest queued outbound message on a channel that no other run holds, skipping ones already
 * tried in this run. An email that should carry a PDF (a proposal or an invoice) is left alone until its
 * attachment is linked, or until it has waited ATTACHMENT_GRACE_SECONDS. An internal alert text held
 * through the person's quiet hours (send_after, migration 0077) waits until they end.
 */
async function pickQueued(client: pg.PoolClient, tenantId: string, channel: 'email' | 'sms', tried: string[]): Promise<PickedMessage | null> {
  const { rows } = await client.query<PickedMessage>(
    `select m.id, m.lead_id, m.customer_id, m.to_address, m.from_address, m.template_key, m.body, m.blocked_reason, m.recipient_user_id,
            m.subject, m.in_reply_to, m.references_header
       from app.messages m
      where m.tenant_id = $1 and m.channel = $2::app.channel and m.direction = 'outbound' and m.status = 'queued'
        and m.id <> all($3::uuid[])
        and (m.send_after is null or m.send_after <= now())
        and (m.channel <> 'email' or m.template_key is null or m.template_key <> all($4::text[])
             or m.created_at < now() - make_interval(secs => $5)
             or exists (select 1 from app.message_attachments a where a.tenant_id = m.tenant_id and a.message_id = m.id))
      order by m.created_at, m.id
      limit 1
      for update of m skip locked`,
    [tenantId, channel, tried, [...PDF_TEMPLATE_KEYS], ATTACHMENT_GRACE_SECONDS],
  );
  return rows[0] ?? null;
```


#### Step 3 — Observe the durable effect

Two valid quote sends were queued before attachment generation. The earlier email body quoted $9,363, but both emails received the later $9,961.70 quote PDF; the earlier signing link had expired.

Natural event overlap, real email delivery, and which document would bind the offer remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/attack-led/public-files/run-estimate-pdf-correlation.mjs:175-198` (test source).

```javascript
  // A separate unchanged-version control keeps the main V1/V2 chain at exactly two events.
  const controlCreated = await request('control-create', 'POST', '/api/estimates', andre.headers,
    { leadId, serviceId, areaSqft: 28_000, source: 'takeoff' }, 201);
  const controlId = controlCreated.json().id;
  await request('control-send-once', 'POST', `/api/estimates/${controlId}/send`, andre.headers, undefined, 200);
  await waitForEvents(app);
  const controlDelivered = await deliverOutbox(app);
  assert.equal(controlDelivered.errors.length, 0);
  assert.ok(controlDelivered.emails.some((e) => e.headers?.['X-[redacted name]-Message']), 'control email not fake-delivered');
  const controlRepeat = await request('control-send-unchanged-again', 'POST', `/api/estimates/${controlId}/send`, andre.headers, undefined, 200);
  raw.controls.unchanged = { alreadySent: controlRepeat.json().alreadySent, queued: proposalRows(controlCreated.json().number).length };
  assert.equal(raw.controls.unchanged.alreadySent, true);
  assert.equal(raw.controls.unchanged.queued, 1);
  raw.verdict = 'OBSERVED';
  raw.conclusion = 'Inspect two queued message bodies against each fake-handed attachment text and the current quote page; classify V1 separately from V2.';
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

Setup: Two authorized quote versions are sent before the queued email events generate their PDF attachments.

Recorded result: Two valid quote sends were queued before attachment generation. The earlier email body quoted $9,363, but both emails received the later $9,961.70 quote PDF; the earlier signing link had expired.

Scope: Natural event overlap, real email delivery, and which document would bind the offer remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/attack-led/public-files/run-estimate-pdf-correlation.mjs:175-198` (test source).

```javascript
  // A separate unchanged-version control keeps the main V1/V2 chain at exactly two events.
  const controlCreated = await request('control-create', 'POST', '/api/estimates', andre.headers,
    { leadId, serviceId, areaSqft: 28_000, source: 'takeoff' }, 201);
  const controlId = controlCreated.json().id;
  await request('control-send-once', 'POST', `/api/estimates/${controlId}/send`, andre.headers, undefined, 200);
  await waitForEvents(app);
  const controlDelivered = await deliverOutbox(app);
  assert.equal(controlDelivered.errors.length, 0);
  assert.ok(controlDelivered.emails.some((e) => e.headers?.['X-[redacted name]-Message']), 'control email not fake-delivered');
  const controlRepeat = await request('control-send-unchanged-again', 'POST', `/api/estimates/${controlId}/send`, andre.headers, undefined, 200);
  raw.controls.unchanged = { alreadySent: controlRepeat.json().alreadySent, queued: proposalRows(controlCreated.json().number).length };
  assert.equal(raw.controls.unchanged.alreadySent, true);
  assert.equal(raw.controls.unchanged.queued, 1);
  raw.verdict = 'OBSERVED';
  raw.conclusion = 'Inspect two queued message bodies against each fake-handed attachment text and the current quote page; classify V1 separately from V2.';
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

Bind each queued email and PDF render to the immutable quote version captured when that send was requested.
