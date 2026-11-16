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

# Twilio number rotation drops old-number STOP and receipts

## Description

After number rotation, signed STOP and receipt callbacks to the retired number are acknowledged then dropped, leaving consent or delivery status stale.

Prerequisite and affected workflow: A company switches Twilio number N to M while valid callbacks or queued N messages remain within the required grace period.

Established observation: After a company changed its Twilio number, signed STOP and delivery callbacks addressed to the retired number were acknowledged but discarded. The resulting consent and delivery records stayed stale during the transition.

Limits: Real Twilio gateway behavior after number rotation was not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A company switches Twilio number N to M while valid callbacks or queued N messages remain within the required grace period.

Source: `apps/api/src/features/automations/routes.ts:219-239` (1d5d98db9acf).

```typescript
  // A brand's sending address and phone number are the from-address of every email and text the company
  // sends, the owner's own security notices included (core/domain/messages.ts, migration 0080), so changing
  // one is a dangerous action: the password again first. It was outside the old route-walking test's fence,
  // and a stolen owner session could repoint every outbound message (the review of this fix, finding 6).
  app.patch('/brands/:id', { schema: { params: z.object({ id: z.uuid("We couldn't find that brand.") }), body: brandBody }, config: { permission: 'automations.edit', dangerous: true } }, async (request) =>
    change(request, async (ctx) => {
      let phoneE164: string | undefined;
      if (request.body.phone !== undefined) {
        const e164 = toE164(request.body.phone);
        if (!e164) throw invalid('Phone must look like (470) 555-0100.');
        phoneE164 = e164;
      }
      let sendingEmail: string | undefined;
      if (request.body.sendingEmail !== undefined) {
        const parsed = z.email().safeParse(request.body.sendingEmail);
        if (!parsed.success) throw invalid('Enter a valid email address, like [redacted email].');
        sendingEmail = parsed.data.toLowerCase();
      }
      await updateBrand(ctx, request.params.id, { name: request.body.name, phoneE164, sendingEmail });
    }),
  );
```

#### Step 2 — Cross the missing state or authority check

The owner route writes only the current `phone_e164` (`apps/api/src/features/automations/routes.ts:219-239`; `automations/automations.ts:275-293`). `app.company_for_texting_number` matches current brand numbers, not a retired-number mapping (`packages/db/migrations/0047_texting.sql:24-35`). The public routes persist callbacks and return `200`/`204` (`apps/api/src/features/texting/routes.ts:34-40,70-87`), but the worker drops an N row when current lookup fails before signature, STOP or receipt processing (`apps/worker/src/jobs/webhook-inbox.ts:181-210`). On a resolved number, STOP and receipts would update consent/status (`:239-288,304-327`). Outbox consent is rechecked, yet the persisted `from_address` is passed to the SMS adapter without current-number validation (`apps/worker/src/jobs/outbox-delivery.ts:855-908`).

Source: `packages/db/migrations/0047_texting.sql:24-35` (1d5d98db9acf).

```sql
create or replace function app.company_for_texting_number(p_phone text)
returns uuid
language sql stable security definer set search_path = app, pg_temp as $$
  select case when count(distinct b.tenant_id) = 1 then min(b.tenant_id::text)::uuid else null end
    from app.business_units b
    join app.tenants t on t.id = b.tenant_id
   where b.phone_e164 = p_phone
     and p_phone ~ '^\+[1-9][0-9]{7,14}$'
     and t.suspended_at is null
$$;
comment on function app.company_for_texting_number(text) is
  'The active company whose brand owns this phone number (E.164), or null when none, several, or suspended. For the texting webhook, which has no session.';
```

Source: `apps/api/src/features/texting/routes.ts:34-40` (1d5d98db9acf).

```typescript
/** Stores a webhook request for the worker, exactly as it came (the signature covers the path and every field). */
async function receive(app: ApiApp, provider: 'twilio' | 'twilio_status', request: FastifyRequest): Promise<void> {
  if (typeof request.body !== 'string') throw new HttpError(403, 'bad_signature', NOT_FROM_TWILIO);
  const signature = request.headers['x-twilio-signature'];
  await app.db.withAuth(({ client }) =>
    client.query('select app.receive_webhook($1, $2, $3, $4)', [provider, request.url, typeof signature === 'string' ? signature.slice(0, 2000) : null, request.body]),
  );
```


#### Step 3 — Observe the durable effect

After a company changed its Twilio number, signed STOP and delivery callbacks addressed to the retired number were acknowledged but discarded. The resulting consent and delivery records stayed stale during the transition.

Real Twilio gateway behavior after number rotation was not tested.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/pocs/run-twilio-old-number.mjs:180-214` (synthetic test).

```javascript

  // Supported owner transition: the app requires step-up and updates the current brand number N -> M.
  const owner = await login(app, '[redacted email]', TEST_DEMO_PASSWORD);
  const changeUrl = `/api/automations/brands/${brand}`;
  const noStepUp = await app.inject({ method: 'PATCH', url: changeUrl, headers: owner.headers, payload: { phone: newNumber } });
  assert.equal(noStepUp.statusCode, 403);
  await stepUp(app, owner.headers, TEST_DEMO_PASSWORD);
  const changed = await app.inject({ method: 'PATCH', url: changeUrl, headers: owner.headers, payload: { phone: newNumber } });
  assert.equal(changed.statusCode, 200, changed.body);
  const numberState = { brandPhone: psql(`select phone_e164 from app.business_units where id='${brand}'`),
    oldLookup: psql(`select coalesce(app.company_for_texting_number('${oldNumber}')::text,'none')`),
    newLookup: psql(`select coalesce(app.company_for_texting_number('${newNumber}')::text,'none')`) };
  assert.equal(numberState.brandPhone, newNumber);
  assert.equal(numberState.oldLookup, 'none');
  assert.equal(numberState.newLookup, peach);

  // Effect-time outbound control: a queued N text existed before the supported N -> M update.
  // Send after that update through a fake gateway, alongside a newly queued M positive control.
  const queuedNewId = seedQueued(newNumber, 'post-switch');
  const providerAttempts = [];
  const fakeOutbound = { mode: 'live', texting: { provider: 'twilio' },
    sms: { async send(input) { providerAttempts.push(input); return { providerMessageId: sid(100 + providerAttempts.length) }; } } };
  const outboundTally = emptyTally();
  await sendQueuedTexts(ctx, peach, new Date(), outboundTally, fakeOutbound);
  const queuedState = rows(`select id, status, from_address, to_address, provider_message_id from app.messages
    where id in ('${queuedOldId}', '${queuedNewId}') order by id`);
  const outbound = { queuedOldId, queuedNewId, queuedState, providerAttempts, tally: outboundTally };

  // After switch: API still acknowledges signed callbacks to N, but worker resolves only current M.
  const afterStopSid = sid(4);
  const afterStopIngress = await post(inboundRoute, incoming(oldNumber, afterStopSid));
  const afterStopDrain = await drain();
  const afterStop = { ingress: afterStopIngress, drain: afterStopDrain, consent: consent(), message: message(afterStopSid), inbox: inbox() };
  const queuedAfterIgnoredStopId = seedQueued(newNumber, 'after-ignored-STOP');
  const attemptsBeforeIgnoredStopSend = providerAttempts.length;
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A company switches Twilio number N to M while valid callbacks or queued N messages remain within the required grace period.

Recorded result: After a company changed its Twilio number, signed STOP and delivery callbacks addressed to the retired number were acknowledged but discarded. The resulting consent and delivery records stayed stale during the transition.

Scope: Real Twilio gateway behavior after number rotation was not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/pocs/run-twilio-old-number.mjs:180-214` (synthetic test).

```javascript

  // Supported owner transition: the app requires step-up and updates the current brand number N -> M.
  const owner = await login(app, '[redacted email]', TEST_DEMO_PASSWORD);
  const changeUrl = `/api/automations/brands/${brand}`;
  const noStepUp = await app.inject({ method: 'PATCH', url: changeUrl, headers: owner.headers, payload: { phone: newNumber } });
  assert.equal(noStepUp.statusCode, 403);
  await stepUp(app, owner.headers, TEST_DEMO_PASSWORD);
  const changed = await app.inject({ method: 'PATCH', url: changeUrl, headers: owner.headers, payload: { phone: newNumber } });
  assert.equal(changed.statusCode, 200, changed.body);
  const numberState = { brandPhone: psql(`select phone_e164 from app.business_units where id='${brand}'`),
    oldLookup: psql(`select coalesce(app.company_for_texting_number('${oldNumber}')::text,'none')`),
    newLookup: psql(`select coalesce(app.company_for_texting_number('${newNumber}')::text,'none')`) };
  assert.equal(numberState.brandPhone, newNumber);
  assert.equal(numberState.oldLookup, 'none');
  assert.equal(numberState.newLookup, peach);

  // Effect-time outbound control: a queued N text existed before the supported N -> M update.
  // Send after that update through a fake gateway, alongside a newly queued M positive control.
  const queuedNewId = seedQueued(newNumber, 'post-switch');
  const providerAttempts = [];
  const fakeOutbound = { mode: 'live', texting: { provider: 'twilio' },
    sms: { async send(input) { providerAttempts.push(input); return { providerMessageId: sid(100 + providerAttempts.length) }; } } };
  const outboundTally = emptyTally();
  await sendQueuedTexts(ctx, peach, new Date(), outboundTally, fakeOutbound);
  const queuedState = rows(`select id, status, from_address, to_address, provider_message_id from app.messages
    where id in ('${queuedOldId}', '${queuedNewId}') order by id`);
  const outbound = { queuedOldId, queuedNewId, queuedState, providerAttempts, tally: outboundTally };

  // After switch: API still acknowledges signed callbacks to N, but worker resolves only current M.
  const afterStopSid = sid(4);
  const afterStopIngress = await post(inboundRoute, incoming(oldNumber, afterStopSid));
  const afterStopDrain = await drain();
  const afterStop = { ingress: afterStopIngress, drain: afterStopDrain, consent: consent(), message: message(afterStopSid), inbox: inbox() };
  const queuedAfterIgnoredStopId = seedQueued(newNumber, 'after-ignored-STOP');
  const attemptsBeforeIgnoredStopSend = providerAttempts.length;
```

## Remediation

Keep retired numbers mapped for signed inbound STOP and receipts during the grace period and block or rebind outbound N messages.
