# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Medium (same-tenant confidentiality)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Import-history detail reveals notes from a lead the reader cannot open

## Description

Import-history detail can expose notes from a same-company lead the reader cannot open.

Prerequisite and affected workflow: A Controller with own-only lead visibility reads a historical batch containing another user's imported lead and its original or mapped CSV notes.

Established observation: A reader could not open another staff member's lead, yet a historical import-batch response returned that lead's distinctive notes in its mapped and original CSV payload. Foreign-company and revoked-reader requests were denied.

Limits: Cross-tenant disclosure and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A Controller with own-only lead visibility reads a historical batch containing another user's imported lead and its original or mapped CSV notes.

Source: `apps/api/src/features/import/routes.ts:126-142` (1d5d98db9acf).

```typescript
async function listRows(ctx: DbContext, batchId: string): Promise<ImportRowRecord[]> {
  const { rows } = await ctx.client.query<{
    row_number: number;
    legacy_id: string | null;
    action: ImportAction;
    reason: string | null;
    warnings: string[];
    lead_id: string | null;
    payload: ImportRowPayload;
  }>(
    `select row_number, legacy_id, action, reason, warnings, lead_id, payload
       from app.import_rows where tenant_id = $1 and batch_id = $2 order by row_number`,
    [ctx.tenantId, batchId],
  );
  return rows.map((row) => ({
    rowNumber: row.row_number,
    legacyId: row.legacy_id,
```

#### Step 2 — Cross the missing state or authority check

The historical batch read authorizes the importer record without applying current visibility to each imported lead's payload.

Source: `apps/api/src/features/import/routes.ts:221-233` (1d5d98db9acf).

```typescript
    app.get('/ghl/batches', async (request): Promise<BatchListResponse> => {
      requirePermission(request.session, 'import.run');
      return app.db.withUser(request.session, async (ctx) => ({ items: await listBatches(ctx) }));
    });

    app.get('/ghl/batches/:id', { schema: { params: idParams } }, async (request): Promise<BatchDetailResponse> => {
      requirePermission(request.session, 'import.run');
      const { id } = request.params;
      return app.db.withUser(request.session, async (ctx) => {
        const batch = (await listBatches(ctx)).find((b) => b.id === id);
        if (!batch) throw notFound("We couldn't find that import.");
        return { batch, rows: await listRows(ctx, id) };
      });
```


#### Step 3 — Observe the durable effect

A reader could not open another staff member's lead, yet a historical import-batch response returned that lead's distinctive notes in its mapped and original CSV payload. Foreign-company and revoked-reader requests were denied.

Cross-tenant disclosure and deployed effect remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/api-frontier/run-import-batch-payload-boundary.mjs:196-214` (test source).

```javascript
  raw.limitations.push('One newly created synthetic GHL row under final migrations; no spatial behavior, customer data, provider egress, AWS deployment, or every importer route.');
} catch (error) {
  raw.error = error instanceof Error ? { message: error.message, stack: error.stack } : String(error);
} finally {
  if (app) await app.close();
  if (started) {
    const pidFile = path.join(data, 'postmaster.pid');
    if (existsSync(pidFile) && Number(readFileSync(pidFile, 'utf8').split('\n')[0]) === postmasterPid) {
      spawnSync(path.join(pgBin, 'pg_ctl'), ['-D', data, '-m', 'immediate', 'stop'], { cwd: repo, stdio: 'ignore', timeout: 30_000 });
    }
  }
  raw.isolation.privateServerStopped = !existsSync(path.join(data, 'postmaster.pid'));
  assert.ok(scratch.startsWith('/private/tmp/aos-import-batch-'));
  rmSync(scratch, { recursive: true, force: true });
  raw.isolation.scratchRemoved = !existsSync(scratch);
  writeFileSync(rawPath, JSON.stringify(raw, null, 2) + '\n');
  console.log(JSON.stringify({ result: raw.result, rawPath, error: raw.error?.message, cases: raw.cases.map((c) => c.name) }, null, 2));
  if (raw.error) process.exitCode = 1;
}
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A Controller with own-only lead visibility reads a historical batch containing another user's imported lead and its original or mapped CSV notes.

Recorded result: A reader could not open another staff member's lead, yet a historical import-batch response returned that lead's distinctive notes in its mapped and original CSV payload. Foreign-company and revoked-reader requests were denied.

Scope: Cross-tenant disclosure and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/api-frontier/run-import-batch-payload-boundary.mjs:196-214` (test source).

```javascript
  raw.limitations.push('One newly created synthetic GHL row under final migrations; no spatial behavior, customer data, provider egress, AWS deployment, or every importer route.');
} catch (error) {
  raw.error = error instanceof Error ? { message: error.message, stack: error.stack } : String(error);
} finally {
  if (app) await app.close();
  if (started) {
    const pidFile = path.join(data, 'postmaster.pid');
    if (existsSync(pidFile) && Number(readFileSync(pidFile, 'utf8').split('\n')[0]) === postmasterPid) {
      spawnSync(path.join(pgBin, 'pg_ctl'), ['-D', data, '-m', 'immediate', 'stop'], { cwd: repo, stdio: 'ignore', timeout: 30_000 });
    }
  }
  raw.isolation.privateServerStopped = !existsSync(path.join(data, 'postmaster.pid'));
  assert.ok(scratch.startsWith('/private/tmp/aos-import-batch-'));
  rmSync(scratch, { recursive: true, force: true });
  raw.isolation.scratchRemoved = !existsSync(scratch);
  writeFileSync(rawPath, JSON.stringify(raw, null, 2) + '\n');
  console.log(JSON.stringify({ result: raw.result, rawPath, error: raw.error?.message, cases: raw.cases.map((c) => c.name) }, null, 2));
  if (raw.error) process.exitCode = 1;
}
```

## Remediation

Filter historical import rows and their saved fields according to the reader's current permission to view each lead.
