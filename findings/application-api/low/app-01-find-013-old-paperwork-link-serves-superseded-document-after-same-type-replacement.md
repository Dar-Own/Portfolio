# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Old paperwork link serves superseded document after same-type replacement

## Description

An old public paperwork link continues to serve the previous document after staff upload a newer document of the same type.

Prerequisite and affected workflow: A recipient reuses an unexpired issued paperwork link after same-type company-library replacement.

Established observation: A public paperwork link issued for a synthetic document still returned the old file bytes after staff uploaded a newer company-library document of the same type. The replacement did not revoke the earlier bearer link.

Limits: Real customer use of an Internet-facing link was not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A recipient reuses an unexpired issued paperwork link after same-type company-library replacement.

Source: `apps/api/src/features/paperwork/routes.ts:294-307` (1d5d98db9acf).

```typescript
      const document = await currentCompanyDocument(ctx, item.docType);
      if (!document) {
        throw invalid(`Upload the company's ${companyDocTypes.label(item.docType)} under Company documents first, then send it.`);
      }
      const today = await companyToday(ctx);
      if (expiryState(document.expires_on, today).state === 'expired') {
        throw invalid(`The ${companyDocTypes.label(item.docType)} on file expired on ${shortDate(document.expires_on ?? today)}. Upload the current one first.`);
      }
      const extension = UPLOAD_TYPES[document.content_type] ?? 'pdf';
      const link = await createDocumentLink(ctx, {
        fileId: document.file_id,
        fileName: documentFileName(job.brandName, item.docType, extension),
        jobDocumentId: item.id,
      });
```

#### Step 2 — Cross the missing state or authority check

The send route binds its link to the then-current company-document file (`apps/api/src/features/paperwork/routes.ts:294-307`); a newer same-type upload appends another library row without revoking old links (`:362-378`; `apps/api/src/features/paperwork/company-documents.ts:92-123`). The public route returns the file named by that link (`apps/api/src/features/paperwork/routes.ts:453-465`). The final bearer door checks token hash, expiry, active tenant, and file key, not whether a newer same-type company document superseded it (`packages/db/migrations/0081_files_encrypted_at_rest.sql:139-153`; `0107_a_key_per_file.sql:118-129`). The old file remains referenced by its link.

Source: `apps/api/src/features/paperwork/company-documents.ts:92-123` (1d5d98db9acf).

```typescript
/** The document a "Send" would use: the newest one of that type. */
export async function currentCompanyDocument(ctx: DbContext, docType: CompanyDocType): Promise<DocumentRow | null> {
  const { rows } = await ctx.client.query<DocumentRow>(
    `${DOCUMENT_SELECT} where d.doc_type = $1 order by d.created_at desc, d.id desc limit 1`,
    [docType],
  );
  return rows[0] ?? null;
}

export interface AddCompanyDocumentInput extends UploadInput {
  docType: CompanyDocType;
  label?: string | null;
  effectiveOn?: string | null;
  expiresOn?: string | null;
}

/** Stores the upload and adds it to the library. The COI needs an expiry date (the warning needs it). */
export async function addCompanyDocument(ctx: DbContext, store: FileStore, input: AddCompanyDocumentInput): Promise<string> {
  if (!companyDocTypes.is(input.docType)) throw invalid('Pick what kind of document this is.');
  if (input.docType === 'coi' && !input.expiresOn) throw invalid('Add the expiry date on the certificate, so we can warn you before it runs out.');
  if (input.effectiveOn && input.expiresOn && input.expiresOn < input.effectiveOn) {
    throw invalid('The expiry date must come after the effective date.');
  }
  const stored = await storePaperworkFile(ctx, store, input);
  const id = randomUUID();
  await ctx.client.query(
    `insert into app.company_documents (id, tenant_id, doc_type, label, file_id, effective_on, expires_on)
     values ($1, $2, $3, $4, $5, $6, $7)`,
    [id, ctx.tenantId, input.docType, input.label?.trim() || companyDocTypes.label(input.docType), stored.fileId, input.effectiveOn ?? null, input.expiresOn ?? null],
  );
  ctx.emit('paperwork.company_document_added', id);
  return id;
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

A public paperwork link issued for a synthetic document still returned the old file bytes after staff uploaded a newer company-library document of the same type. The replacement did not revoke the earlier bearer link.

Real customer use of an Internet-facing link was not tested.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/pocs/run-high-risk-async-files.mjs:295-322` (synthetic test).

```javascript
  const baselineSecond = await getBytes(secondLink.token);
  assert.deepEqual([baselineSecond.code, baselineSecond.marker], [200, true]);
  const replacement = await app.inject({ method: 'POST', url: `/api/paperwork/items/${paperItem}/file`, headers: sofia.headers,
    payload: { contentType: 'application/pdf', dataBase64: pdf('NEW-JOB-COI').toString('base64') } });
  assert.equal(replacement.statusCode, 200, replacement.body);
  const afterChecklistReplacement = await getBytes(secondLink.token);

  const thirdLink = await sendLink();
  const baselineThird = await getBytes(thirdLink.token);
  assert.deepEqual([baselineThird.code, baselineThird.marker], [200, true]);
  const newLibrary = await postDoc('NEW-LIBRARY-COI');
  assert.equal(newLibrary.statusCode, 201, newLibrary.body);
  const libraryRows = rows(`select label, file_id, created_at from app.company_documents where tenant_id='${paperTenant}' and doc_type='coi' order by created_at desc, id desc limit 2`);
  const latestLibraryRowIsNew = libraryRows[0]?.label === 'NEW-LIBRARY-COI';
  const afterLibraryReplacement = await getBytes(thirdLink.token);
  const fourthLink = await sendLink();
  const newLinkIsDistinct = fourthLink.token !== thirdLink.token;
  const currentLibraryBytes = await getBytes(fourthLink.token);
  const expiredControlToken = firstLink.token;
  psql(`update app.document_links set expires_at=now()-interval '1 second'
    where tenant_id='${paperTenant}' and token_hash=sha256(convert_to('${expiredControlToken}','UTF8'))`);
  const expiredControl = await getBytes(expiredControlToken);
  assert.equal(expiredControl.code, 404, 'expiry still revokes a document link');
  const oldFileStillRecorded = psql(`select count(*) from app.files where id='${oldFile}'`);
  const linkCount = psql(`select count(*) from app.document_links where tenant_id='${paperTenant}' and file_id='${oldFile}'`);
  psql(`update app.tenants set suspended_at=now() where id='${paperTenant}'`);
  const suspendedTenantControl = await getBytes(thirdLink.token);
  assert.equal(suspendedTenantControl.code, 404, 'tenant suspension still revokes a document link');
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A recipient reuses an unexpired issued paperwork link after same-type company-library replacement.

Recorded result: A public paperwork link issued for a synthetic document still returned the old file bytes after staff uploaded a newer company-library document of the same type. The replacement did not revoke the earlier bearer link.

Scope: Real customer use of an Internet-facing link was not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/pocs/run-high-risk-async-files.mjs:295-322` (synthetic test).

```javascript
  const baselineSecond = await getBytes(secondLink.token);
  assert.deepEqual([baselineSecond.code, baselineSecond.marker], [200, true]);
  const replacement = await app.inject({ method: 'POST', url: `/api/paperwork/items/${paperItem}/file`, headers: sofia.headers,
    payload: { contentType: 'application/pdf', dataBase64: pdf('NEW-JOB-COI').toString('base64') } });
  assert.equal(replacement.statusCode, 200, replacement.body);
  const afterChecklistReplacement = await getBytes(secondLink.token);

  const thirdLink = await sendLink();
  const baselineThird = await getBytes(thirdLink.token);
  assert.deepEqual([baselineThird.code, baselineThird.marker], [200, true]);
  const newLibrary = await postDoc('NEW-LIBRARY-COI');
  assert.equal(newLibrary.statusCode, 201, newLibrary.body);
  const libraryRows = rows(`select label, file_id, created_at from app.company_documents where tenant_id='${paperTenant}' and doc_type='coi' order by created_at desc, id desc limit 2`);
  const latestLibraryRowIsNew = libraryRows[0]?.label === 'NEW-LIBRARY-COI';
  const afterLibraryReplacement = await getBytes(thirdLink.token);
  const fourthLink = await sendLink();
  const newLinkIsDistinct = fourthLink.token !== thirdLink.token;
  const currentLibraryBytes = await getBytes(fourthLink.token);
  const expiredControlToken = firstLink.token;
  psql(`update app.document_links set expires_at=now()-interval '1 second'
    where tenant_id='${paperTenant}' and token_hash=sha256(convert_to('${expiredControlToken}','UTF8'))`);
  const expiredControl = await getBytes(expiredControlToken);
  assert.equal(expiredControl.code, 404, 'expiry still revokes a document link');
  const oldFileStillRecorded = psql(`select count(*) from app.files where id='${oldFile}'`);
  const linkCount = psql(`select count(*) from app.document_links where tenant_id='${paperTenant}' and file_id='${oldFile}'`);
  psql(`update app.tenants set suspended_at=now() where id='${paperTenant}'`);
  const suspendedTenantControl = await getBytes(thirdLink.token);
  assert.equal(suspendedTenantControl.code, 404, 'tenant suspension still revokes a document link');
```

## Remediation

Revoke links to the superseded file on replacement and enforce revocation in the public bearer door.
