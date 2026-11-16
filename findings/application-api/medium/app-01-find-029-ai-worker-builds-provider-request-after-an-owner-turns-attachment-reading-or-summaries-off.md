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

# AI worker builds provider request after an owner turns attachment reading or summaries off

## Description

An AI request can still contain PDF bytes or email content after the owner turns off the corresponding attachment or summary feature.

Prerequisite and affected workflow: The worker claims eligible mail while the switch is on and the owner turns it off before model handoff.

Established observation: The AI worker built a provider-bound request containing PDF bytes after attachment reading was switched off. A separate test sent a distinctive email-content marker to the test model after summaries were switched off and saved the resulting summary.

Limits: No real Anthropic transfer occurred. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

The worker claims eligible mail while the switch is on and the owner turns it off before model handoff.

Source: `apps/worker/src/jobs/mail-ai.ts:289-304` (1d5d98db9acf).

```typescript
/** The PDFs and photos of an email that may leave, with the owner's switch alone, read through the file store. */
async function readDocuments(client: pg.PoolClient, tenantId: string, messageId: string, org: Organization, files: WorkerFileStore): Promise<AiDocument[]> {
  if (!org.ai.attachmentsAllowed) return [];
  const { rows } = await client.query<{ file_id: string; file_name: string; content_type: string; size_bytes: string | number }>(
    `select a.file_id, regexp_replace(f.storage_key, '^.*/', '') as file_name, f.content_type, f.size_bytes
       from app.email_attachments a join app.files f on f.tenant_id = a.tenant_id and f.id = a.file_id
      where a.tenant_id = $1 and a.email_id = $2 order by f.storage_key`,
    [tenantId, messageId],
  );
  const chosen = documentsToSend(rows.map((row) => ({ fileId: row.file_id, fileName: row.file_name, contentType: row.content_type, sizeBytes: Number(row.size_bytes) })), org.ai);
  const documents: AiDocument[] = [];
  for (const file of chosen) {
    const { bytes } = await files.read(client, tenantId, file.fileId);
    documents.push({ name: file.fileName, mediaType: file.mediaType, base64: bytes.toString('base64') });
  }
  return documents;
```

#### Step 2 — Cross the missing state or authority check

The model request uses claim-time settings and content without rechecking the switches before provider initiation.

Source: `apps/api/src/features/inbox/routes.ts:436-443` (1d5d98db9acf).

```typescript
    // More of the company's mail leaves the building: the owner's decision, with the password again (0096's reasoning).
    app.patch(
      '/ai-settings/attachments',
      { schema: { body: attachmentsBody }, config: { permission: 'automations.edit', ownerOnly: AI_OWNER_ONLY_TEXT, dangerous: true } },
      async (request): Promise<MailAiSettingsView> => {
        await app.db.withUser(request.session, (ctx) => saveReadAttachments(ctx, request.body.readAttachments));
        return aiView(request.session);
      },
```

Source: `apps/api/src/features/inbox/ai.ts:286-292` (1d5d98db9acf).

```typescript
/** "Read attachments too": the owner's alone (the route says so first; the database refuses anybody else). */
export async function saveReadAttachments(ctx: DbContext, on: boolean): Promise<void> {
  await ctx.client.query(
    `insert into app.mail_settings (tenant_id, ai_read_attachments) values ($1, $2)
     on conflict (tenant_id) do update set ai_read_attachments = excluded.ai_read_attachments where mail_settings.ai_read_attachments is distinct from excluded.ai_read_attachments`,
    [ctx.tenantId, on],
  );
```


#### Step 3 — Observe the durable effect

The AI worker built a provider-bound request containing PDF bytes after attachment reading was switched off. A separate test sent a distinctive email-content marker to the test model after summaries were switched off and saved the resulting summary.

No real Anthropic transfer occurred.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/attack-led/workers-bulk/run-summary-off-after-claim.mjs:257-288` (synthetic test).

```javascript
        timeline.push({ step: 'claim_commit_finished_before_provider', setting: summarySetting(), providerCalls: fake.requests.length, atBarrier });
        switched = await switchSummaries(false);
        timeline.push({ step: 'real_http_off_committed_before_provider', setting: 'f', providerCalls: fake.requests.length });
      } finally { releaseGate(); releaseGate = null; }
      result = await bounded(running, 30_000, `${name} worker completion`);
      running = null;
    } else {
      result = await runMailAi(job, { provider });
    }
    assert.equal(result.errors, 0, `${name}: worker errors: ${errors.join(' | ')}`);
    const after = snapshot(clone, peach, thread);
    const capturedRequests = fake.requests.map(captured);
    const rivalRequests = rivalFake.requests.map(captured);
    assert.equal(rivalRequests.length, 0, `${name}: foreign provider leakage`);
    if (name === 'off_before_claim') {
      assert.equal(capturedRequests.length, 0, `${name}: OFF-before-claim provider leak`);
      assert.equal(after.requests[0].outcome, 'off');
      assert.equal(after.summary, null);
      assert.equal(after.run_count, 0, `${name}: OFF-before-claim recorded an AI run`);
    } else {
      assert.equal(capturedRequests.length, 1, `${name}: expected exact one requested summary call`);
      assert.ok(capturedRequests[0].messages.some(message => message.content.includes(marker)), `${name}: marker absent from model request`);
      assert.equal(after.requests[0].outcome, 'ok');
      assert.ok(after.summary?.includes(marker), `${name}: summary absent`);
      assert.equal(after.run_count, 1, `${name}: expected one summary AI run`);
    }
    const callsBeforeRetry = fake.requests.length;
    const retry = await runMailAi(job, { provider });
    assert.equal(retry.errors, 0, `${name}: retry errors: ${errors.join(' | ')}`);
    assert.equal(fake.requests.length, callsBeforeRetry, `${name}: duplicate provider call`);
    assert.equal(rivalFake.requests.length, 0, `${name}: foreign provider leakage on retry`);
    assert.deepEqual(tenantEffects(clone, rival), foreignBefore, `${name}: Rival AI effects changed`);
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: The worker claims eligible mail while the switch is on and the owner turns it off before model handoff.

Recorded result: The AI worker built a provider-bound request containing PDF bytes after attachment reading was switched off. A separate test sent a distinctive email-content marker to the test model after summaries were switched off and saved the resulting summary.

Scope: No real Anthropic transfer occurred. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/attack-led/workers-bulk/run-summary-off-after-claim.mjs:257-288` (synthetic test).

```javascript
        timeline.push({ step: 'claim_commit_finished_before_provider', setting: summarySetting(), providerCalls: fake.requests.length, atBarrier });
        switched = await switchSummaries(false);
        timeline.push({ step: 'real_http_off_committed_before_provider', setting: 'f', providerCalls: fake.requests.length });
      } finally { releaseGate(); releaseGate = null; }
      result = await bounded(running, 30_000, `${name} worker completion`);
      running = null;
    } else {
      result = await runMailAi(job, { provider });
    }
    assert.equal(result.errors, 0, `${name}: worker errors: ${errors.join(' | ')}`);
    const after = snapshot(clone, peach, thread);
    const capturedRequests = fake.requests.map(captured);
    const rivalRequests = rivalFake.requests.map(captured);
    assert.equal(rivalRequests.length, 0, `${name}: foreign provider leakage`);
    if (name === 'off_before_claim') {
      assert.equal(capturedRequests.length, 0, `${name}: OFF-before-claim provider leak`);
      assert.equal(after.requests[0].outcome, 'off');
      assert.equal(after.summary, null);
      assert.equal(after.run_count, 0, `${name}: OFF-before-claim recorded an AI run`);
    } else {
      assert.equal(capturedRequests.length, 1, `${name}: expected exact one requested summary call`);
      assert.ok(capturedRequests[0].messages.some(message => message.content.includes(marker)), `${name}: marker absent from model request`);
      assert.equal(after.requests[0].outcome, 'ok');
      assert.ok(after.summary?.includes(marker), `${name}: summary absent`);
      assert.equal(after.run_count, 1, `${name}: expected one summary AI run`);
    }
    const callsBeforeRetry = fake.requests.length;
    const retry = await runMailAi(job, { provider });
    assert.equal(retry.errors, 0, `${name}: retry errors: ${errors.join(' | ')}`);
    assert.equal(fake.requests.length, callsBeforeRetry, `${name}: duplicate provider call`);
    assert.equal(rivalFake.requests.length, 0, `${name}: foreign provider leakage on retry`);
    assert.deepEqual(tenantEffects(clone, rival), foreignBefore, `${name}: Rival AI effects changed`);
```

## Remediation

Recheck each AI data-sharing switch immediately before provider initiation and cancel claims that lost authority.
