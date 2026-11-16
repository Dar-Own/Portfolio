# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Medium (conditional)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Acknowledged, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Import commits a lead after import.run is revoked

## Description

An in-flight import can create a lead and batch after its operator's import permission is revoked.

Prerequisite and affected workflow: The import passes its initial authorization check and the owner revokes import permission before the business write.

Established observation: An import that had passed its initial permission check committed a new lead and batch after the owner revoked the operator's import permission. Fresh requests by the revoked operator and foreign-company controls were denied.

Limits: A deployed NAS import race was not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

The import passes its initial authorization check and the owner revokes import permission before the business write.

Source: `apps/api/src/features/import/routes.ts:160-174` (1d5d98db9acf).

```typescript

/** The import as this person: plan and write in the same importer transaction (so the plan can't go stale in between), then the activity feed. */
export async function runAs(db: Db, session: Session, csv: string): Promise<RunResponse> {
  requirePermission(session, 'import.run');
  const parsed = parseGhlExport(csv);
  if (!parsed.ok) throw invalid(parsed.error);
  const runBy = session.user.id;
  const result = await db.withImporter(session, async (ctx) => {
    const { plan, setup } = await rehearseImport(ctx, parsed.records);
    return runImport(ctx, plan, setup, runBy);
  });
  const c = result.counts;
  await db.withUser(session, (ctx) =>
    addActivityNote(ctx, `Imported from GoHighLevel: ${c.create} created, ${c.update} updated, ${c.duplicate} duplicates skipped, ${c.error} errors`),
  );
```

#### Step 2 — Cross the missing state or authority check

The importer checks permission once at transaction entry and its privileged writes do not recheck the grant.

The same source excerpt above contains this boundary; the report does not supply a second distinct implementation location.

The same production boundary is reproduced here for this step:

Source: `apps/api/src/features/import/routes.ts:160-174` (1d5d98db9acf).

```typescript

/** The import as this person: plan and write in the same importer transaction (so the plan can't go stale in between), then the activity feed. */
export async function runAs(db: Db, session: Session, csv: string): Promise<RunResponse> {
  requirePermission(session, 'import.run');
  const parsed = parseGhlExport(csv);
  if (!parsed.ok) throw invalid(parsed.error);
  const runBy = session.user.id;
  const result = await db.withImporter(session, async (ctx) => {
    const { plan, setup } = await rehearseImport(ctx, parsed.records);
    return runImport(ctx, plan, setup, runBy);
  });
  const c = result.counts;
  await db.withUser(session, (ctx) =>
    addActivityNote(ctx, `Imported from GoHighLevel: ${c.create} created, ${c.update} updated, ${c.duplicate} duplicates skipped, ${c.error} errors`),
  );
```

#### Step 3 — Observe the durable effect

An import that had passed its initial permission check committed a new lead and batch after the owner revoked the operator's import permission. Fresh requests by the revoked operator and foreign-company controls were denied.

A deployed NAS import race was not tested.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/pocs/run-after-door-import-session.mjs:138-172` (synthetic test).

```javascript
  const beforePaused = importCounts();
  const cliHold = timed('cli-after-importer-door', originalImporter, (s) => s.user.id === rita.session.user.id);
  app.db.withImporter = cliHold.wrap;
  const pendingCli = runCli('cli-revoked');
  const entered = await cliHold.entered();
  const revoked = await grant('DELETE');
  assert.equal(revoked.currentlyGranted, false);
  const afterAdmin = importCounts();
  const released = cliHold.release();
  const cliOutcome = await pendingCli.then((lines) => ({ ok: true, lines }), (e) => ({ ok: false, error: String(e.message ?? e) }));
  app.db.withImporter = originalImporter;
  const afterPaused = importCounts();
  assert.equal(afterPaused.peach_leads, afterAdmin.peach_leads + 1, 'post-revocation CLI wrote a lead');
  result.scenarios.push({ name: 'import-cli-revoked-post-door', login: 'poc_after_importer', timing: { entered, adminCommitted: revoked.at, released }, revoked, before: beforePaused, afterAdmin, after: afterPaused, outcome: cliOutcome });
  assert.ok(entered <= revoked.at && revoked.at <= released, 'CLI ordering');
  const freshCli = await runCli('cli-fresh-denied').then(() => ({ ok: true }), (e) => ({ ok: false, error: String(e.message ?? e) }));
  assert.equal(freshCli.ok, false, 'fresh revoked CLI must deny');
  assert.deepEqual(importCounts(), afterPaused, 'fresh denied CLI business effects');
  result.scenarios.push({ name: 'import-cli-fresh-revoked', outcome: freshCli, before: afterPaused, after: importCounts() });

  await grant('PUT');
  const beforeHttp = importCounts();
  const httpHold = timed('http-after-importer-door', originalImporter, (s) => s.user.id === rita.session.user.id);
  app.db.withImporter = httpHold.wrap;
  const pendingHttp = send('POST', '/api/import/ghl/run', rita.headers, { csv: csv('http-revoked') });
  const httpEntered = await httpHold.entered();
  const httpRevoke = await grant('DELETE');
  const httpAdmin = importCounts();
  const httpReleased = httpHold.release();
  const httpOutcome = await pendingHttp;
  app.db.withImporter = originalImporter;
  const afterHttp = importCounts();
  assert.equal(afterHttp.peach_leads, httpAdmin.peach_leads + 1, 'post-revocation HTTP wrote a lead');
  assert.ok(httpEntered <= httpRevoke.at && httpRevoke.at <= httpReleased, 'HTTP ordering');
  result.scenarios.push({ name: 'import-http-revoked-post-door', login: 'poc_after_importer', timing: { entered: httpEntered, adminCommitted: httpRevoke.at, released: httpReleased }, before: beforeHttp, afterAdmin: httpAdmin, after: afterHttp, response: brief(httpOutcome) });
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: The import passes its initial authorization check and the owner revokes import permission before the business write.

Recorded result: An import that had passed its initial permission check committed a new lead and batch after the owner revoked the operator's import permission. Fresh requests by the revoked operator and foreign-company controls were denied.

Scope: A deployed NAS import race was not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/pocs/run-after-door-import-session.mjs:138-172` (synthetic test).

```javascript
  const beforePaused = importCounts();
  const cliHold = timed('cli-after-importer-door', originalImporter, (s) => s.user.id === rita.session.user.id);
  app.db.withImporter = cliHold.wrap;
  const pendingCli = runCli('cli-revoked');
  const entered = await cliHold.entered();
  const revoked = await grant('DELETE');
  assert.equal(revoked.currentlyGranted, false);
  const afterAdmin = importCounts();
  const released = cliHold.release();
  const cliOutcome = await pendingCli.then((lines) => ({ ok: true, lines }), (e) => ({ ok: false, error: String(e.message ?? e) }));
  app.db.withImporter = originalImporter;
  const afterPaused = importCounts();
  assert.equal(afterPaused.peach_leads, afterAdmin.peach_leads + 1, 'post-revocation CLI wrote a lead');
  result.scenarios.push({ name: 'import-cli-revoked-post-door', login: 'poc_after_importer', timing: { entered, adminCommitted: revoked.at, released }, revoked, before: beforePaused, afterAdmin, after: afterPaused, outcome: cliOutcome });
  assert.ok(entered <= revoked.at && revoked.at <= released, 'CLI ordering');
  const freshCli = await runCli('cli-fresh-denied').then(() => ({ ok: true }), (e) => ({ ok: false, error: String(e.message ?? e) }));
  assert.equal(freshCli.ok, false, 'fresh revoked CLI must deny');
  assert.deepEqual(importCounts(), afterPaused, 'fresh denied CLI business effects');
  result.scenarios.push({ name: 'import-cli-fresh-revoked', outcome: freshCli, before: afterPaused, after: importCounts() });

  await grant('PUT');
  const beforeHttp = importCounts();
  const httpHold = timed('http-after-importer-door', originalImporter, (s) => s.user.id === rita.session.user.id);
  app.db.withImporter = httpHold.wrap;
  const pendingHttp = send('POST', '/api/import/ghl/run', rita.headers, { csv: csv('http-revoked') });
  const httpEntered = await httpHold.entered();
  const httpRevoke = await grant('DELETE');
  const httpAdmin = importCounts();
  const httpReleased = httpHold.release();
  const httpOutcome = await pendingHttp;
  app.db.withImporter = originalImporter;
  const afterHttp = importCounts();
  assert.equal(afterHttp.peach_leads, httpAdmin.peach_leads + 1, 'post-revocation HTTP wrote a lead');
  assert.ok(httpEntered <= httpRevoke.at && httpRevoke.at <= httpReleased, 'HTTP ordering');
  result.scenarios.push({ name: 'import-http-revoked-post-door', login: 'poc_after_importer', timing: { entered: httpEntered, adminCommitted: httpRevoke.at, released: httpReleased }, before: beforeHttp, afterAdmin: httpAdmin, after: afterHttp, response: brief(httpOutcome) });
```

## Remediation

Recheck import permission before writes or cancel the in-flight transaction when the grant is revoked.
