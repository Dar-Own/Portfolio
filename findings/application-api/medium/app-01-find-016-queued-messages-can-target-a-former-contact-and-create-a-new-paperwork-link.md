# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Medium (conditional document-confidentiality variant)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Queued messages can target a former contact and create a new paperwork link

## Description

Queued job, proposal, direct-email, and paperwork messages can still target a former contact after correction. The paperwork path can issue a new public COI link that serves the document.

Prerequisite and affected workflow: A primary contact changes from A to B while an A message is queued, selected, or retried before provider handoff.

Established observation: After staff corrected a contact from A to B, an unsent paperwork email was still accepted for A by captured mail. That path minted a new A-addressed document link after the correction, and the public link returned the exact synthetic COI bytes.

Limits: Real email and SMS delivery were not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A primary contact changes from A to B while an A message is queued, selected, or retried before provider handoff.

Source: `apps/api/src/features/paperwork/routes.ts:303-325` (1d5d98db9acf).

```typescript
      const link = await createDocumentLink(ctx, {
        fileId: document.file_id,
        fileName: documentFileName(job.brandName, item.docType, extension),
        jobDocumentId: item.id,
      });
      // The download link is minted by the worker as it sends the email (migration 0106 stage 2d): the
      // stored letter holds its placeholder, tied to the link's row.
      const email = companyDocumentEmail({
        firstName: firstNameOf(job.contactName, job.customerName),
        brand: { name: job.brandName, phone: job.brandPhone ?? '' },
        kind: item.docType,
        address: job.address,
        downloadUrl: CUSTOMER_LINK_PLACEHOLDERS.document,
        linkDays: DOCUMENT_LINK_DAYS,
      });
      const queued = await queueEmail(ctx, {
        to: job.email,
        body: email.text,
        templateKey: email.templateKey,
        leadId: job.leadId,
        businessUnitId: job.brandId,
      });
      await awaitLink(ctx, queued.id, 'document', link.id);
```

#### Step 2 — Cross the missing state or authority check

Producers and retries keep the stored A address, and final handoff does not check current contact authority.

Source: `apps/api/src/features/customers/routes.ts:354-375` (1d5d98db9acf).

```typescript
    app.patch('/:id/contacts/:contactId', { schema: { params: contactParams, body: contactPatch } }, async (request) => {
      requireEdit(request.session);
      const body = request.body;
      const set: [string, unknown][] = [];
      if (body.fullName !== undefined) set.push(['full_name', body.fullName]);
      if (body.title !== undefined) set.push(['title', body.title || null]);
      if (body.email !== undefined) set.push(['email', emailOrNull(body.email)]);
      if (body.phone !== undefined) set.push(['phone_e164', phoneOrNull(body.phone)]);
      const { id, contactId } = request.params;
      return app.db.withUser(request.session, async (ctx) => {
        const exists = await ctx.client.query('select 1 from app.contacts where tenant_id = $1 and id = $2 and customer_id = $3', [ctx.tenantId, contactId, id]);
        if (exists.rowCount !== 1) throw notFound("We couldn't find that person on this customer.");
        if (set.length > 0) {
          const result = await ctx.client.query(
            `update app.contacts set ${set.map(([column], i) => `${column} = $${i + 4}`).join(', ')}
              where tenant_id = $1 and id = $2 and customer_id = $3`,
            [ctx.tenantId, contactId, id, ...set.map(([, value]) => value)],
          );
          if (result.rowCount !== 1) throw notFound("We couldn't find that person on this customer.");
        }
        if (body.isPrimary === true) await makePrimary(ctx, id, contactId);
        return { id: contactId };
```

Source: `apps/worker/src/jobs/outbox-delivery.ts:637-680` (1d5d98db9acf).

```typescript
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

Source: `apps/api/src/features/paperwork/routes.ts:453-465` (1d5d98db9acf).

```typescript
  publicRoutes: {
    prefix: '/api/public/paperwork',
    register(app) {
      /** The document a customer was emailed. The link works for 30 days and says nothing else. */
      app.get('/documents/:token', { schema: { params: tokenParams }, config: { rateLimit: { max: 20, timeWindow: 60_000 } } }, async (request, reply) => {
        const { token } = request.params;
        if (!isWellFormedPublicToken(token)) throw notFound("This link isn't valid anymore.");
        // The link door hands over the file it names with the company's wrapped key (app.document_link_file_key,
        // 0106); the bytes are opened with the master key, on no transaction and as nobody.
        const shared = await documentByLinkToken(app.db, token);
        if (!shared) throw notFound("This link isn't valid anymore.");
        const bytes = await readLinkedFile(app.config, shared);
        return reply.headers(attachmentHeaders(shared.fileName, shared.contentType)).send(bytes);
```


#### Step 3 — Observe the durable effect

After staff corrected a contact from A to B, an unsent paperwork email was still accepted for A by captured mail. That path minted a new A-addressed document link after the correction, and the public link returned the exact synthetic COI bytes.

Real email and SMS delivery were not tested.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/attack-led/public-files/run-message-retry-recipient.mjs:177-200` (test source).

```javascript
  const retryAgain = await inject('POST', `/api/messages/${messageId}/retry`, owner.headers, {});
  assert.equal(retryAgain.statusCode, 422, retryAgain.body);
  out.controls.doubleRetry = { status: retryAgain.statusCode, message: message(messageId) };
  failTarget = false;
  const j19 = await deliverOutbox(ctx);
  out.timeline.push({ step: 'J19 fake SMTP handoff after B correction and manual retry', result: j19, state: message(messageId) });
  out.providerAttempts = attempts.filter(x => x.marker);
  out.effects = { before, after: effects(tenant), foreign: effects(rival) };
  const acceptedOld = attempts.filter(x => x.marker && x.accepted && x.to === oldEmail);
  const acceptedCurrent = attempts.filter(x => x.marker && x.accepted && x.to === correctedEmail);
  assert.equal(acceptedCurrent.length, 1, 'fresh B control must reach fake provider');
  assert.equal(message(messageId).status, 'sent');
  assert.equal(acceptedOld.length, 1, 'manual retry reached old A after B correction');
  out.result = 'PROVED-LOCAL-OLD-RECIPIENT-RETRY';
  out.limitations = ['A fake SMTP acceptance proves application handoff, not customer receipt.', 'The client has stated B-only after invoice and proposal contact corrections; a generic manual-message correction rule was not separately answered.', 'No spatial, production build, AWS, or real provider claim.'];
} catch (error) {
  out.error = String(error?.stack ?? error);
  throw error;
} finally {
  try { if (app) await app.close(); } catch (error) { out.cleanupError = `app: ${String(error)}`; }
  try { if (ctx) await ctx.close(); } catch (error) { out.cleanupError = `worker: ${String(error)}`; }
  let stopped = !started;
  try { if (started) { command(path.join(pgBin, 'pg_ctl'), ['-D', data, '-m', 'immediate', '-w', 'stop']); stopped = true; } }
  catch (error) { out.cleanupError = `PostgreSQL: ${String(error)}`; }
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A primary contact changes from A to B while an A message is queued, selected, or retried before provider handoff.

Recorded result: After staff corrected a contact from A to B, an unsent paperwork email was still accepted for A by captured mail. That path minted a new A-addressed document link after the correction, and the public link returned the exact synthetic COI bytes.

Scope: Real email and SMS delivery were not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/attack-led/public-files/run-message-retry-recipient.mjs:177-200` (test source).

```javascript
  const retryAgain = await inject('POST', `/api/messages/${messageId}/retry`, owner.headers, {});
  assert.equal(retryAgain.statusCode, 422, retryAgain.body);
  out.controls.doubleRetry = { status: retryAgain.statusCode, message: message(messageId) };
  failTarget = false;
  const j19 = await deliverOutbox(ctx);
  out.timeline.push({ step: 'J19 fake SMTP handoff after B correction and manual retry', result: j19, state: message(messageId) });
  out.providerAttempts = attempts.filter(x => x.marker);
  out.effects = { before, after: effects(tenant), foreign: effects(rival) };
  const acceptedOld = attempts.filter(x => x.marker && x.accepted && x.to === oldEmail);
  const acceptedCurrent = attempts.filter(x => x.marker && x.accepted && x.to === correctedEmail);
  assert.equal(acceptedCurrent.length, 1, 'fresh B control must reach fake provider');
  assert.equal(message(messageId).status, 'sent');
  assert.equal(acceptedOld.length, 1, 'manual retry reached old A after B correction');
  out.result = 'PROVED-LOCAL-OLD-RECIPIENT-RETRY';
  out.limitations = ['A fake SMTP acceptance proves application handoff, not customer receipt.', 'The client has stated B-only after invoice and proposal contact corrections; a generic manual-message correction rule was not separately answered.', 'No spatial, production build, AWS, or real provider claim.'];
} catch (error) {
  out.error = String(error?.stack ?? error);
  throw error;
} finally {
  try { if (app) await app.close(); } catch (error) { out.cleanupError = `app: ${String(error)}`; }
  try { if (ctx) await ctx.close(); } catch (error) { out.cleanupError = `worker: ${String(error)}`; }
  let stopped = !started;
  try { if (started) { command(path.join(pgBin, 'pg_ctl'), ['-D', data, '-m', 'immediate', '-w', 'stop']); stopped = true; } }
  catch (error) { out.cleanupError = `PostgreSQL: ${String(error)}`; }
```

## Remediation

Cancel or readdress stale messages on correction and recheck current contact authority at final handoff and retry.
