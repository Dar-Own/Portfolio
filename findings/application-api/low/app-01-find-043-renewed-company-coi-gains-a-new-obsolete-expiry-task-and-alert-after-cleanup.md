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

# Renewed company COI gains a new obsolete expiry task and alert after cleanup

## Description

A renewed company COI can still acquire an open obsolete expiry task and false alert.

Prerequisite and affected workflow: The worker selects an expiring certificate, then a renewed certificate is saved and cleanup runs before the worker creates its reminder.

Established observation: A genuine renewed insurance document was uploaded and its cleanup event completed before the reminder worker resumed. The worker then created a new open expiry task and false alert for the obsolete document, and a retry left the task open.

Limits: Natural overlap, deployment, and insurance consequences remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

The worker selects an expiring certificate, then a renewed certificate is saved and cleanup runs before the worker creates its reminder.

Source: `apps/worker/src/jobs/paperwork-reminders.ts:88-123` (1d5d98db9acf).

```typescript
  const scan = await forEachActiveTenant(ctx.pool, async (tenantId) => {
    await withAutomation(ctx.pool, tenantId, async (client) => {
      const hours = await businessHours(client, tenantId);
      const today = localDate(now, hours.timezone);
      const { rows } = await client.query<CoiRow>(
        `select id, label, expires_on::text as expires_on from app.company_documents
          where tenant_id = $1 and doc_type = 'coi'
          order by created_at desc, id desc
          limit 1`,
        [tenantId],
      );
      const coi = rows[0];
      if (!coi?.expires_on) {
        tally.quiet++;
        return;
      }
      const expiry = expiryState(coi.expires_on, today, COI_WARNING_DAYS);
      if (expiry.state !== 'expiring' && expiry.state !== 'expired') {
        tally.quiet++;
        return;
      }
      const ownerId = await reminderOwner(client, tenantId);
      if (!ownerId) {
        throw new Error('Nobody holds the office routing rule, so the COI reminder has no owner.');
      }
      const created = await createReminder(client, {
        tenantId,
        ownerId,
        documentId: coi.id,
        state: expiry.state,
        expiresOn: coi.expires_on,
        // An expired certificate is due now; one that is running out is due tomorrow.
        dueAt: expiry.state === 'expired' ? now : addBusinessMinutes(now, 540, hours),
        timeZone: hours.timezone,
        daysLeft: expiry.daysLeft ?? 0,
      });
```

#### Step 2 — Cross the missing state or authority check

The insurance-expiry worker uses an old certificate snapshot after replacement cleanup has already removed the existing reminders.

Source: `apps/api/src/features/paperwork/routes.ts:362-378` (1d5d98db9acf).

```typescript
  app.post('/company-documents', { schema: { body: companyDocumentBody }, bodyLimit: UPLOAD_BODY_LIMIT }, async (request, reply) => {
    requirePermission(request.session, 'paperwork.manage');
    const body = request.body;
    const result = await app.db.withUser(request.session, async (ctx) => {
      const id = await addCompanyDocument(ctx, store, {
        docType: body.docType,
        label: body.label ?? null,
        effectiveOn: body.effectiveOn ?? null,
        expiresOn: body.expiresOn ?? null,
        contentType: body.contentType,
        dataBase64: body.dataBase64,
      });
      const today = await companyToday(ctx);
      const documents = await listCompanyDocuments(ctx, today);
      return { id, documents, today };
    });
    return reply.status(201).send(result);
```


#### Step 3 — Observe the durable effect

A genuine renewed insurance document was uploaded and its cleanup event completed before the reminder worker resumed. The worker then created a new open expiry task and false alert for the obsolete document, and a retry left the task open.

Natural overlap, deployment, and insurance consequences remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/paperwork-reminder-race/run-coi-replacement-after-select.mjs:225-247` (synthetic test).

```javascript
  const afterCleanup = ledger(peach, oldPeachId);
  assert.equal(afterCleanup.current_coi, newPeachId);
  assert.equal(afterCleanup.events, 0);
  assert.equal(afterCleanup.old_open_tasks, 0);
  releasePause(); releasePause = null;
  const job = await within(pending, 'J20 resumed task creation');
  const after = { peach: ledger(peach, oldPeachId), rival: ledger(rival, oldRivalId) };
  const stale = taskDetail(peach, oldPeachId);
  assert.equal(job.errors, 0, JSON.stringify(job));
  assert.equal(after.peach.old_open_tasks, 1, 'stale old-COI reminder did not appear');
  assert.equal(after.peach.coi_alerts, afterCleanup.coi_alerts + 1, 'stale alert missing');
  assert.equal(stale.status, 'open');
  assert.match(stale.title, /Upload the renewed company COI/);
  assert.equal(after.rival.old_open_tasks, 1, 'unchanged Rival positive did not get reminder');
  assert.equal(after.peach.messages, afterCleanup.messages, 'J20 unexpectedly queued customer message');
  assert.equal(after.peach.document_links, afterCleanup.document_links, 'J20 unexpectedly minted document link');
  assert.equal(after.peach.paperwork_files, afterCleanup.paperwork_files, 'J20 unexpectedly wrote file bytes');
  const oldFile = documentFile(peach, oldPeachId);
  const newFile = documentFile(peach, newPeachId);
  assert.equal(oldFile.sha256, createHash('sha256').update(pdf('J20 OLD PEACH')).digest('hex'));
  assert.equal(newFile.sha256, createHash('sha256').update(pdf('J20 NEW PEACH')).digest('hex'));
  assert.notEqual(oldFile.sha256, newFile.sha256);
  const retry = await remindAboutPaperwork(ctx);
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: The worker selects an expiring certificate, then a renewed certificate is saved and cleanup runs before the worker creates its reminder.

Recorded result: A genuine renewed insurance document was uploaded and its cleanup event completed before the reminder worker resumed. The worker then created a new open expiry task and false alert for the obsolete document, and a retry left the task open.

Scope: Natural overlap, deployment, and insurance consequences remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/paperwork-reminder-race/run-coi-replacement-after-select.mjs:225-247` (synthetic test).

```javascript
  const afterCleanup = ledger(peach, oldPeachId);
  assert.equal(afterCleanup.current_coi, newPeachId);
  assert.equal(afterCleanup.events, 0);
  assert.equal(afterCleanup.old_open_tasks, 0);
  releasePause(); releasePause = null;
  const job = await within(pending, 'J20 resumed task creation');
  const after = { peach: ledger(peach, oldPeachId), rival: ledger(rival, oldRivalId) };
  const stale = taskDetail(peach, oldPeachId);
  assert.equal(job.errors, 0, JSON.stringify(job));
  assert.equal(after.peach.old_open_tasks, 1, 'stale old-COI reminder did not appear');
  assert.equal(after.peach.coi_alerts, afterCleanup.coi_alerts + 1, 'stale alert missing');
  assert.equal(stale.status, 'open');
  assert.match(stale.title, /Upload the renewed company COI/);
  assert.equal(after.rival.old_open_tasks, 1, 'unchanged Rival positive did not get reminder');
  assert.equal(after.peach.messages, afterCleanup.messages, 'J20 unexpectedly queued customer message');
  assert.equal(after.peach.document_links, afterCleanup.document_links, 'J20 unexpectedly minted document link');
  assert.equal(after.peach.paperwork_files, afterCleanup.paperwork_files, 'J20 unexpectedly wrote file bytes');
  const oldFile = documentFile(peach, oldPeachId);
  const newFile = documentFile(peach, newPeachId);
  assert.equal(oldFile.sha256, createHash('sha256').update(pdf('J20 OLD PEACH')).digest('hex'));
  assert.equal(newFile.sha256, createHash('sha256').update(pdf('J20 NEW PEACH')).digest('hex'));
  assert.notEqual(oldFile.sha256, newFile.sha256);
  const retry = await remindAboutPaperwork(ctx);
```

## Remediation

Before saving an expiry task or alert, confirm that the selected certificate is still the latest effective document.
