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

# Queued internal alert sends after user disable or membership removal

## Description

Queued internal alerts and enrollment-code emails can still be handed to the email adapter for a disabled or removed member.

Prerequisite and affected workflow: A recipient loses active user or tenant membership status after message creation and before mail handoff.

Established observation: Internal alerts reached captured mail after their recipient was disabled or removed from the company. A separate enrollment-code letter was also handed off after membership removal, although the removed member could not use the code.

Limits: Real delivery of either message type was not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A recipient loses active user or tenant membership status after message creation and before mail handoff.

Source: `packages/db/migrations/0077_alerts_profiles_and_ssn.sql:426-458` (1d5d98db9acf).

```sql
  for r in
    select c.*, u.email::text as user_email
      from app.member_contacts c
      join app.users u on u.id = c.user_id and u.disabled_at is null and not u.is_system
     where c.tenant_id = new.tenant_id and c.user_id = any(new.recipient_ids)
     order by c.user_id
  loop
    if r.alert_channel in ('text', 'text_email') then
      phone := case r.sms_phone when 'work' then r.work_phone else r.personal_phone end;
      if phone is not null then
        local_ts := now() at time zone tz;
        local_t := local_ts::time;
        in_quiet := case when r.quiet_start < r.quiet_end then local_t >= r.quiet_start and local_t < r.quiet_end
                         when r.quiet_start > r.quiet_end then local_t >= r.quiet_start or local_t < r.quiet_end
                         else false end;
        after_ts := null;
        if in_quiet and not (new.urgent and r.urgent_anyway) then
          after_ts := (local_ts::date + r.quiet_end) at time zone tz;
          if after_ts <= now() then
            after_ts := after_ts + interval '1 day';
          end if;
        end if;
        insert into app.messages (tenant_id, channel, direction, to_address, from_address, template_key, body, status,
                                  recipient_user_id, notification_id, send_after)
        values (new.tenant_id, 'sms', 'outbound', phone, coalesce(brand.phone, ''), 'internal_alert',
                line || ' Reply STOP to opt out.', 'queued', r.user_id, new.id, after_ts);
      end if;
    end if;
    if r.alert_channel in ('email', 'text_email') then
      insert into app.messages (tenant_id, channel, direction, to_address, from_address, template_key, body, status,
                                recipient_user_id, notification_id)
      values (new.tenant_id, 'email', 'outbound', r.user_email, coalesce(brand.email, ''), 'internal_alert',
              line, 'queued', r.user_id, new.id);
```

#### Step 2 — Cross the missing state or authority check

Queued-alert and direct enrollment send paths use an earlier recipient decision without an effect-time membership check.

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

Source: `apps/worker/src/jobs/outbox-delivery.ts:615-680` (1d5d98db9acf).

```typescript
    const picked = await withAutomation(ctx.pool, tenantId, async (client) => {
      const message = await pickQueued(client, tenantId, 'email', tried);
      if (!message) return false;
      tried.push(message.id);
      const invitation = await isInvitationLetter(client, tenantId, message);
      const way = await emailWayOut(client, tenantId, message, outbound, run.install, invitation);
      // That way out is down this run: leave this one queued and go on to the next message.
      if (run.serversDown.has(way.server)) return true;

      // Live, and the company has no mail service set up: say so on the message rather than pretend.
      if (!way.sender) {
        const noted = await client.query(
          `update app.messages set status = 'failed', sent_at = null, blocked_reason = $3
            where tenant_id = $1 and id = $2 and status = 'queued'`,
          [tenantId, message.id, emailNotSetUpReason(way.problem ?? 'no mail service is set up.')],
        );
        if (noted.rowCount !== 1) throw new Error(`Email ${message.id} could not be marked as not sent.`);
        tally.emailsNotSent++;
        return true;
      }
      const sender = way.sender;

      const brand = await brandOf(client, tenantId, message);
      // The letter's links are minted only for a letter that leaves: rolled back with the savepoint otherwise.
      await client.query('savepoint letter');
      let letter: MintedLetter;
      try {
        letter = await mintLinks(client, tenantId, message);
      } catch (error) {
        if (!(error instanceof LinkGoneError)) throw error;
        await client.query('rollback to savepoint letter');
        await markLinkGone(client, ctx, tenantId, message, 'queued', notSentReason(error), tally);
        return true;
      }
      // A reply from the shared inbox keeps its own subject; every other letter's is rebuilt from its template.
      const subject = invitation
        ? way.viaInstall
          ? INSTALL_INVITE_SUBJECT
          : inviteEmailSubject(letter.inviterName ?? brand.company_name, brand.company_name)
        : (message.subject ?? emailSubject(message.template_key, brand.name));
      const fromAddress = brand.sending_email ?? message.from_address;
      // Only a reply from the shared inbox that really leaves is copied into the Sent folder.
      const copied = copies !== null && message.template_key === REPLY_TEMPLATE_KEY && !way.viaInstall && !way.toTestMailboxOnly;
      let providerMessageId: string;
      let letterCopy: SentLetter | null = null;
      try {
        const attachments = await loadAttachments(client, tenantId, message.id, files);
        const cc = await replyCc(client, tenantId, message.id);
        const messageId = appMessageIdHeader(message.id, fromAddress);
        const references = message.references_header?.split(/\s+/).filter((id) => id !== '') ?? null;
        const result = await sender.send({
          from: { name: invitation ? brand.company_name : brand.name, address: fromAddress },
          to: message.to_address,
          cc,
          subject,
          text: letter.text,
          attachments,
          // The threading headers of a reply, and on every letter the app's own Message-ID and its mark: the
          // mailbox sync files a customer's answer into the conversation by the one, and knows the Sent
          // folder's copy is the app's own by the other (docs/email-manager.md section 3.3).
          inReplyTo: message.in_reply_to,
          references,
          messageId,
          headers: { [APP_MESSAGE_HEADER]: message.id },
          keepRaw: copied && copies.mailbox !== null,
        });
```


#### Step 3 — Observe the durable effect

Internal alerts reached captured mail after their recipient was disabled or removed from the company. A separate enrollment-code letter was also handed off after membership removal, although the removed member could not use the code.

Real delivery of either message type was not tested.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/enrolment-codes/run-enrolment-removal.mjs:202-222` (synthetic test).

```javascript
    const codeAfterRemoval = await app.inject({ method: 'POST', url: '/api/auth/two-factor/setup', headers: target.headers,
      payload: { password: TEST_DEMO_PASSWORD, emailCode: targetLetter.code } });
    record.controls.targetCodeAfterRemoval = responseShape(codeAfterRemoval);
    assert.equal(codeAfterRemoval.statusCode, 401, 'removed target used code to set up MFA');
    tick('removed-session-and-code-denied');
  } finally {
    releaseResolve();
  }
  const result = await job;
  record.controls.job = result;
  record.controls.workerLogs = logs;
  record.controls.lettersAfter = letterRows().map(({ id, to, code, ...rest }) => ({ id, to, hasCode: code !== null, ...rest }));
  assert.equal(record.fakeProviderAttempts.length, 2, 'expected target and active-owner fake SMTP handoffs');
  assert.ok(record.fakeProviderAttempts.find((attempt) => attempt.to === targetEmail)?.containsExpectedCode, 'target code not in outgoing text');
  assert.ok(record.fakeProviderAttempts.find((attempt) => attempt.to === owner.session.user.email)?.containsExpectedCode, 'positive owner code not in outgoing text');
  assert.equal(record.controls.lettersAfter.length, 0, 'accepted letters not removed');
  assert.equal(result.processed, 2);
  const ownerAfter = await app.inject({ method: 'GET', url: '/api/auth/session', headers: owner.headers });
  record.controls.activeOwnerSessionAfter = responseShape(ownerAfter);
  assert.equal(ownerAfter.statusCode, 200);
  record.outcome = 'TEST-CONFIRMED: J07 fake-accepted removed member code after owner removal; removed session/code route denied';
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A recipient loses active user or tenant membership status after message creation and before mail handoff.

Recorded result: Internal alerts reached captured mail after their recipient was disabled or removed from the company. A separate enrollment-code letter was also handed off after membership removal, although the removed member could not use the code.

Scope: Real delivery of either message type was not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/enrolment-codes/run-enrolment-removal.mjs:202-222` (synthetic test).

```javascript
    const codeAfterRemoval = await app.inject({ method: 'POST', url: '/api/auth/two-factor/setup', headers: target.headers,
      payload: { password: TEST_DEMO_PASSWORD, emailCode: targetLetter.code } });
    record.controls.targetCodeAfterRemoval = responseShape(codeAfterRemoval);
    assert.equal(codeAfterRemoval.statusCode, 401, 'removed target used code to set up MFA');
    tick('removed-session-and-code-denied');
  } finally {
    releaseResolve();
  }
  const result = await job;
  record.controls.job = result;
  record.controls.workerLogs = logs;
  record.controls.lettersAfter = letterRows().map(({ id, to, code, ...rest }) => ({ id, to, hasCode: code !== null, ...rest }));
  assert.equal(record.fakeProviderAttempts.length, 2, 'expected target and active-owner fake SMTP handoffs');
  assert.ok(record.fakeProviderAttempts.find((attempt) => attempt.to === targetEmail)?.containsExpectedCode, 'target code not in outgoing text');
  assert.ok(record.fakeProviderAttempts.find((attempt) => attempt.to === owner.session.user.email)?.containsExpectedCode, 'positive owner code not in outgoing text');
  assert.equal(record.controls.lettersAfter.length, 0, 'accepted letters not removed');
  assert.equal(result.processed, 2);
  const ownerAfter = await app.inject({ method: 'GET', url: '/api/auth/session', headers: owner.headers });
  record.controls.activeOwnerSessionAfter = responseShape(ownerAfter);
  assert.equal(ownerAfter.statusCode, 200);
  record.outcome = 'TEST-CONFIRMED: J07 fake-accepted removed member code after owner removal; removed session/code route denied';
```

## Remediation

Revalidate active recipient authority before both queued-alert and direct enrollment-letter sends.
