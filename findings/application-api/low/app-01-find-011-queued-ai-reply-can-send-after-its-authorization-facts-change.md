# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low (conditional)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Queued AI reply can send after its authorization facts change

## Description

A queued AI reply can reach the mail adapter after its category is disabled, the recipient opts out, or newer mail makes the draft stale.

Prerequisite and affected workflow: Unattended AI mail is enabled and one of those facts changes after queueing but before SMTP handoff.

Established observation: Queued AI replies were handed to captured mail after their category was disabled, the recipient opted out, or newer inbound mail made the reply stale. The send path did not recheck those facts before handoff.

Limits: No external email receipt was proved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

Unattended AI mail is enabled and one of those facts changes after queueing but before SMTP handoff.

Source: `packages/db/migrations/0108_mail_manager.sql:1762-1830` (1d5d98db9acf).

```sql
create function app.mail_ai_may_send(p_draft uuid, p_to text, p_subject text, p_body text)
returns boolean
language sql stable set search_path = app, pg_temp as $$
  select coalesce((
    select true
      from app.mail_suggestions d
      join app.mail_threads t on t.tenant_id = d.tenant_id and t.id = d.thread_id
      join app.email_messages m on m.tenant_id = d.tenant_id and m.id = d.message_id
      join app.mail_settings s on s.tenant_id = d.tenant_id
      cross join lateral (select coalesce((select ts.business_tz from app.tenant_settings ts where ts.tenant_id = d.tenant_id), 'America/New_York') as tz) z
      cross join (select '\$|¢|€|£|¥|(?<![a-z])(usd|eur|euros?|gbp|dollars?|bucks|cents?)(?![a-z])|\d\s?(k|grand)(?![a-z])'::text as money) mp
     where d.tenant_id = app.current_tenant() and d.id = p_draft
       -- the AI's own clean draft, not sent yet, word for word
       and d.kind = 'reply' and d.status = 'suggested' and d.for_user_id = app.current_user_id() and cardinality(d.flags) = 0
       and d.payload->>'subject' = p_subject and d.payload->>'body' = p_body
       and p_subject !~* mp.money and p_body !~* mp.money
       -- a conversation the owner listed, sorted by the AI at least 90% sure, still waiting for a reply
       and t.status = 'needs_reply' and not t.suspicious
       and t.category = any(s.ai_send_categories) and t.category_set_by = 'ai' and t.category_confidence >= 0.9
       -- its latest incoming email, not one of the first sync's
       and m.direction = 'inbound' and not m.backfill
       and m.id = (select x.id from app.email_messages x
                    where x.tenant_id = t.tenant_id and x.thread_id = t.id and x.direction = 'inbound'
                    order by x.received_at desc, x.id desc limit 1)
       -- to somebody already in the conversation, in business hours, under today's cap
       and exists (select 1 from unnest(t.participants) p(address) where lower(p.address::text) = lower(btrim(p_to)))
       and not exists (select 1 from app.current_consents c
                        where c.tenant_id = t.tenant_id and c.channel = 'email' and not c.granted and lower(c.address) = lower(btrim(p_to)))
       and app.business_minutes(t.tenant_id, now(), now() + interval '1 minute') > 0
       and (select sp.cents from app.mail_ai_spend(date_trunc('day', now() at time zone z.tz) at time zone z.tz,
                                                   (date_trunc('day', now() at time zone z.tz) + interval '1 day') at time zone z.tz) sp) < s.ai_daily_cap_cents
  ), false)
$$;
revoke all on function app.mail_ai_may_send(uuid, text, text, text) from public;
grant execute on function app.mail_ai_may_send(uuid, text, text, text) to app_user;
comment on function app.mail_ai_may_send(uuid, text, text, text) is
  'Whether the shared inbox''s AI may send this reply by itself (migration 0108 stage 5, M13): its own clean draft, word for word, with no money in its words, on a conversation of a category the owner listed that it sorted at least 90% sure, to somebody in it who has not opted out of email, in business hours, under the cap. Read as the caller.';

-- Section 4's guard (0099's, 0092's, 0077's), copied forward with two clauses added: on insert, the AI's
-- reply (a mail_reply the Automation member queues) must pass app.mail_ai_may_send, and only such a letter
-- names an AI draft; on update, the draft a letter names never changes. Everything else is exactly as
-- section 4 wrote it.
create or replace function app.guard_message() returns trigger
language plpgsql as $$
begin
  if tg_op = 'INSERT' then
    -- 0108 stage 5: the AI answers by itself only as the owner's switch allows, and only its own reply names
    -- an AI draft.
    if new.ai_draft_id is not null and not app.is_admin_session()
       and (not app.is_automation_member() or lower(coalesce(new.template_key, '')) <> 'mail_reply') then
      raise exception 'Only the AI''s own reply names an AI draft';
    end if;
    if lower(coalesce(new.template_key, '')) = 'mail_reply' and not app.is_admin_session() and app.is_automation_member()
       and (new.channel <> 'email' or not app.mail_ai_may_send(new.ai_draft_id, new.to_address, new.subject, new.body)) then
      raise exception 'The AI may send a reply by itself only as the owner allowed: its own draft with no price, date or question for a person, on a conversation of a category the owner listed that it sorted at least 90%% sure, by email to somebody in the conversation, in business hours and under the daily cap';
    end if;
    if new.recipient_user_id is not null and not app.is_admin_session() then
      raise exception 'Internal alerts are queued by the database, not by hand';
    end if;
    if coalesce(lower(new.template_key) like 'security\_%', false) and not app.is_admin_session() then
      raise exception 'A security notice is written by [redacted name] itself, not by hand';
    end if;
    if new.channel = 'sms' and not app.is_admin_session() then
      if new.direction = 'inbound' and new.status <> 'received' then
        raise exception 'Incoming texts are recorded as received';
      end if;
      if new.direction = 'outbound' and new.status not in ('queued', 'blocked') then
        raise exception 'Text messages must be queued so consent can be checked';
      end if;
```

#### Step 2 — Cross the missing state or authority check

The final `app.mail_ai_may_send` predicate checks category, latest inbound, consent, business hours, and cap only when Automation inserts an AI `app.messages` row (`packages/db/migrations/0108_mail_manager.sql:1762-1830`). `mail-ai` queues that row and advances draft/thread state (`apps/worker/src/jobs/mail-ai.ts:615-672`). The outbox later selects queued mail and calls SMTP without a fresh AI-specific policy check (`apps/worker/src/jobs/outbox-delivery.ts:395-411,615-747`); the email status-update trigger does not repeat the insertion gate (`packages/db/migrations/0108_mail_manager.sql:1869-1870`).

Source: `apps/worker/src/jobs/mail-ai.ts:615-672` (1d5d98db9acf).

```typescript
async function sendOwnReply(client: pg.PoolClient, store: MailStore, tenantId: string, claimed: Claimed, call: Call, draft: Draft, runId: string, thread: OwnReplyThread): Promise<string> {
  const { org, claimedAt } = claimed;
  const reply = call.reply!;
  const draftId = randomUUID();
  await client.query(
    `insert into app.mail_suggestions (id, tenant_id, thread_id, message_id, kind, payload, evidence, confidence, flags, ai_run_id, for_user_id)
     values ($1, $2, $3, $4, 'reply', $5::jsonb, '{}', null, '{}', $6, app.current_user_id())`,
    [draftId, tenantId, call.threadId, call.messageId, JSON.stringify({ subject: draft.subject, body: draft.body, note_to_sender: draft.note_to_sender }), runId],
  );
  // Threaded under the email it answers, from the record's brand (else the company's first).
  const { rows: answered } = await client.query<{ message_id_header: string | null; references_header: string | null }>(
    'select message_id_header, references_header from app.email_messages where tenant_id = $1 and id = $2',
    [tenantId, call.messageId],
  );
  const inReplyTo = answered[0]?.message_id_header ?? null;
  const references = [...new Set([...(answered[0]?.references_header?.split(/\s+/).filter((id) => id !== '') ?? []), ...(inReplyTo ? [inReplyTo] : [])])];
  const { rows: brands } = await client.query<{ sending_email: string; name: string }>(
    `select bu.sending_email::text as sending_email, bu.name from app.business_units bu
      where bu.tenant_id = $1 and bu.sending_email is not null
      order by bu.id = (select l.business_unit_id from app.leads l where l.tenant_id = $1 and l.id = $2) desc nulls last, bu.created_at, bu.id
      limit 1`,
    [tenantId, thread.lead_id],
  );
  const from = brands[0]?.sending_email ?? thread.account ?? org.accounts[0]?.address ?? '';
  const letterId = randomUUID();
  await client.query(
    `insert into app.messages (id, tenant_id, customer_id, lead_id, channel, direction, to_address, from_address, template_key, body, status, created_by,
                               subject, in_reply_to, references_header, ai_draft_id)
     values ($1, $2, $3, $4, 'email', 'outbound', $5, $6, $7, $8, 'queued', app.current_user_id(), $9, $10, $11, $12)`,
    [letterId, tenantId, thread.customer_id, thread.lead_id, reply.to, from, REPLY_TEMPLATE_KEY, draft.body, draft.subject, inReplyTo, referencesText(references), draftId],
  );
  await store.insertMessage(
    {
      threadId: call.threadId,
      account: thread.account ?? org.accounts[0]?.address ?? '',
      folder: 'sent',
      direction: 'outbound',
      providerMessageId: null,
      providerThreadId: thread.provider_thread_id,
      messageIdHeader: appMessageIdHeader(letterId, from),
      inReplyTo,
      references,
      fromAddress: from,
      fromName: brands[0]?.name ?? org.name,
      to: [reply.to],
      cc: [],
      subject: draft.subject,
      text: draft.body,
      receivedAt: claimedAt.toISOString(),
      sender: null,
      outboxMessageId: letterId,
      sentBy: null,
      backfill: false,
    },
    [],
  );
  await store.touchThread(call.threadId, { message: { at: claimedAt.toISOString(), direction: 'outbound' }, participants: [{ address: reply.to, name: null }], status: 'waiting' });
  await client.query(`update app.mail_suggestions set status = 'accepted', applied_kind = 'reply', applied_id = $3 where tenant_id = $1 and id = $2`, [tenantId, draftId, letterId]);
```

Source: `apps/worker/src/jobs/outbox-delivery.ts:395-411` (1d5d98db9acf).

```typescript
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

Queued AI replies were handed to captured mail after their category was disabled, the recipient opted out, or newer inbound mail made the reply stale. The send path did not recheck those facts before handoff.

No external email receipt was proved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/pocs/run-high-risk-async-files.mjs:205-234` (synthetic test).

```javascript
      (select count(*) from app.email_messages e where e.thread_id=t.id and e.direction='inbound') as inbound_count,
      (select max(received_at) from app.email_messages e where e.thread_id=t.id and e.direction='inbound') as latest_inbound_at
      from app.mail_threads t join app.mail_settings m on m.tenant_id=t.tenant_id where t.id='${filed.thread.id}'`)[0];
    const newerAfterQueue = kind === 'newer_inbound'
      ? new Date(observedFacts.latest_inbound_at).getTime() > new Date(queued[0].created_at).getTime()
      : null;
    const callsBefore = fakeSmtp.length;
    const delivered = await deliverOutbox(deliveryCtx);
    assert.equal(delivered.errors, 0, loggerErrors.join(' | '));
    const firstCalls = fakeSmtp.slice(callsBefore);
    timeline.push({ event: 'fake_provider_acceptance', at: new Date().toISOString() });
    const status = rows(`select id, status, provider_message_id from app.messages where id='${queued[0].id}'`)[0];
    const retry = await deliverOutbox(deliveryCtx);
    assert.equal(retry.errors, 0, loggerErrors.join(' | '));
    const retryCalls = fakeSmtp.slice(callsBefore + firstCalls.length);
    aiResults.push({ kind, recipient, before, timeline, observedFacts, newerAfterQueue,
      delivery: { processed: delivered.processed, errors: delivered.errors },
      fakeSmtp: firstCalls, after: status, retry: { processed: retry.processed, newCalls: retryCalls.length } });
    if (kind === 'category_off') {
      psql(`alter table app.mail_settings disable trigger mail_settings_guard;
        update app.mail_settings set ai_send_categories = array['customer_reply'] where tenant_id = '${tenant}';
        alter table app.mail_settings enable trigger mail_settings_guard;`);
    }
  }
  const unchanged = aiResults.find((item) => item.kind === 'unchanged');
  assert.equal(unchanged?.fakeSmtp.length, 1, 'positive control: unchanged AI reply must reach fake SMTP');
  assert.equal(unchanged.fakeSmtp[0].to, unchanged.recipient);
  assert.equal(unchanged.after.status, 'sent');
  assert.equal(unchanged.retry.newCalls, 0);
  psql(`insert into app.consent_events (tenant_id, channel, address, granted, method, evidence)
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: Unattended AI mail is enabled and one of those facts changes after queueing but before SMTP handoff.

Recorded result: Queued AI replies were handed to captured mail after their category was disabled, the recipient opted out, or newer inbound mail made the reply stale. The send path did not recheck those facts before handoff.

Scope: No external email receipt was proved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/pocs/run-high-risk-async-files.mjs:205-234` (synthetic test).

```javascript
      (select count(*) from app.email_messages e where e.thread_id=t.id and e.direction='inbound') as inbound_count,
      (select max(received_at) from app.email_messages e where e.thread_id=t.id and e.direction='inbound') as latest_inbound_at
      from app.mail_threads t join app.mail_settings m on m.tenant_id=t.tenant_id where t.id='${filed.thread.id}'`)[0];
    const newerAfterQueue = kind === 'newer_inbound'
      ? new Date(observedFacts.latest_inbound_at).getTime() > new Date(queued[0].created_at).getTime()
      : null;
    const callsBefore = fakeSmtp.length;
    const delivered = await deliverOutbox(deliveryCtx);
    assert.equal(delivered.errors, 0, loggerErrors.join(' | '));
    const firstCalls = fakeSmtp.slice(callsBefore);
    timeline.push({ event: 'fake_provider_acceptance', at: new Date().toISOString() });
    const status = rows(`select id, status, provider_message_id from app.messages where id='${queued[0].id}'`)[0];
    const retry = await deliverOutbox(deliveryCtx);
    assert.equal(retry.errors, 0, loggerErrors.join(' | '));
    const retryCalls = fakeSmtp.slice(callsBefore + firstCalls.length);
    aiResults.push({ kind, recipient, before, timeline, observedFacts, newerAfterQueue,
      delivery: { processed: delivered.processed, errors: delivered.errors },
      fakeSmtp: firstCalls, after: status, retry: { processed: retry.processed, newCalls: retryCalls.length } });
    if (kind === 'category_off') {
      psql(`alter table app.mail_settings disable trigger mail_settings_guard;
        update app.mail_settings set ai_send_categories = array['customer_reply'] where tenant_id = '${tenant}';
        alter table app.mail_settings enable trigger mail_settings_guard;`);
    }
  }
  const unchanged = aiResults.find((item) => item.kind === 'unchanged');
  assert.equal(unchanged?.fakeSmtp.length, 1, 'positive control: unchanged AI reply must reach fake SMTP');
  assert.equal(unchanged.fakeSmtp[0].to, unchanged.recipient);
  assert.equal(unchanged.after.status, 'sent');
  assert.equal(unchanged.retry.newCalls, 0);
  psql(`insert into app.consent_events (tenant_id, channel, address, granted, method, evidence)
```

## Remediation

Recheck current category, consent, and latest-inbound state immediately before handoff and cancel invalid queued replies.
