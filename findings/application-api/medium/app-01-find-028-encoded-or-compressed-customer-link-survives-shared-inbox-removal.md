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

# Encoded or compressed customer link survives shared-inbox removal

## Description

An encoded customer link remains visible in shared-inbox text, and a compressed-PDF link can be forwarded as bytes. Both preserve a working bearer beyond redaction.

Prerequisite and affected workflow: A valid customer link arrives in an encoded or compressed representation that the filing path accepts and an inbox member reads or forwards it.

Established observation: A genuinely issued customer document link survived inbox redaction when represented as encoded text or inside a compressed PDF. The PDF bytes were forwarded by the captured mail adapter, and the preserved public link still served the synthetic document.

Limits: An ordinary supported mail provider producing the crafted PDF representation was not observed. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A valid customer link arrives in an encoded or compressed representation that the filing path accepts and an inbox member reads or forwards it.

Source: `apps/api/src/features/inbox/sql-store.ts:416-455` (1d5d98db9acf).

```typescript
      const id = randomUUID();
      // No customer's working link is stored, whatever path the email came by (audit 3): not in the text (our
      // own letter quoted in a reply, a forward), and not in an attached file (the quoted letter as an .htm
      // file, a forward as an .eml), which is left out and named in the text instead. Anybody who reads the
      // inbox could otherwise open the customer's page as the customer.
      const taken = files.slice(0, MAX_MAILBOX_ATTACHMENTS).filter((file) => file.bytes.length > 0 && file.bytes.length <= MAX_MAILBOX_ATTACHMENT_BYTES);
      const leftOut = taken.filter((file) => attachmentHoldsCustomerLink(file, origins));
      const text = message.text.replace(/\r\n?/g, '\n').trim();
      const body = withoutLinks(leftOut.length === 0 ? text : `${text}\n\n${customerLinkFileNote(leftOut.map((file) => safeAttachmentFileName(file.fileName)))}`);
      await client.query(
        `insert into app.email_messages (id, tenant_id, thread_id, mailbox, direction, folder, provider_message_id, message_id_header, in_reply_to, references_header,
                                         from_address, from_name, to_addresses, cc_addresses, subject, preview, body, received_at, lead_id, job_id, outbox_message_id, sent_by, backfill)
         values ($1, $2, $3, $4::app.mailbox, $5::app.msg_direction, $6, $7, $8, $9, $10, $11, $12, $13::citext[], $14::citext[], $15, $16, $17, $18, $19, $20, $21, $22, $23)`,
        [
          id, tenantId, message.threadId, box, message.direction, message.folder,
          message.providerMessageId === null ? null : stored(message.providerMessageId), message.messageIdHeader, message.inReplyTo, referencesHeader(message.references),
          message.fromAddress.toLowerCase(), message.fromName, message.to, message.cc,
          withoutLinks(message.subject).trim() || '(no subject)', body.slice(0, 200), body, new Date(message.receivedAt), leadId, jobId, message.outboxMessageId, message.sentBy,
          message.backfill,
        ],
      );

      const written: string[] = [];
      // The names this email's files took, without case: some disks don't tell Plan.pdf from plan.pdf.
      const usedNames = new Set<string>();
      try {
        for (const file of taken) {
          if (leftOut.includes(file)) continue;
          const safeName = safeAttachmentFileName(file.fileName);
          // Two attachments with the same name in one email still need two file keys: "2-plan.pdf", "3-plan.pdf"...,
          // counted on until the name is free. A stranger's "plan.pdf", "3-plan.pdf", "plan.pdf" once gave the third
          // the second's name, the file store refused it and the whole email with it (audit 3).
          let name = safeName;
          for (let n = 2; usedNames.has(name.toLowerCase()); n++) name = `${n}-${safeName}`;
          usedNames.add(name.toLowerCase());
          const storageKey = `${tenantId}/inbox/${id}/${name}`;
          const fileId = randomUUID();
          await options.storeFile({ fileId, storageKey, bytes: file.bytes, contentType: safeContentType(file.contentType) });
          written.push(storageKey);
          await client.query('insert into app.email_attachments (tenant_id, email_id, file_id) values ($1, $2, $3)', [tenantId, id, fileId]);
```

#### Step 2 — Cross the missing state or authority check

Redaction misses percent-encoded text and compressed PDF content, while reply attachment forwarding preserves those bytes.

Source: `apps/api/src/features/inbox/reply.ts:87-107` (1d5d98db9acf).

```typescript
  // Files this person may see (the record's, or an upload made for this reply), by their rows.
  const files = fileIds.length === 0
    ? []
    : (await ctx.client.query<{ id: string; storage_key: string; content_type: string }>('select id, storage_key, content_type from app.files where tenant_id = $1 and id = any($2::uuid[])', [ctx.tenantId, fileIds])).rows;
  if (files.length !== fileIds.length) throw notFound(FILE_NOT_FOUND);

  const queued = await queueEmail(ctx, {
    to,
    body,
    templateKey: REPLY_TEMPLATE_KEY,
    leadId: thread.lead_id,
    customerId: thread.customer_id,
    subject,
    inReplyTo,
    references,
  });
  for (const file of files) {
    await ctx.client.query(
      'insert into app.message_attachments (id, tenant_id, message_id, file_id, file_name, content_type) values ($1, $2, $3, $4, $5, $6)',
      [randomUUID(), ctx.tenantId, queued.id, file.id, file.storage_key.replace(/^.*\//, ''), file.content_type],
    );
```

Source: `apps/worker/src/jobs/outbox-delivery.ts:423-443` (1d5d98db9acf).

```typescript
/** The files linked to an email, read through the file store. A linked file that isn't there, or can't be opened, is an error (the message fails). */
async function loadAttachments(client: pg.PoolClient, tenantId: string, messageId: string, files: WorkerFileStore): Promise<EmailAttachment[]> {
  const { rows } = await client.query<{ file_name: string; content_type: string; file_id: string }>(
    `select a.file_name, a.content_type, a.file_id
       from app.message_attachments a
      where a.tenant_id = $1 and a.message_id = $2
      order by a.created_at, a.id`,
    [tenantId, messageId],
  );
  const attachments: EmailAttachment[] = [];
  for (const row of rows) {
    let content: Buffer;
    try {
      ({ bytes: content } = await files.read(client, tenantId, row.file_id));
    } catch (error) {
      if ((error as NodeJS.ErrnoException).code === 'ENOENT') throw new Error(`The attachment ${row.file_name} is missing from the file store.`);
      throw new Error(`The attachment ${row.file_name} can't be opened: ${errorText(error)}`);
    }
    attachments.push({ fileName: row.file_name, contentType: row.content_type, content });
  }
  return attachments;
```


#### Step 3 — Observe the durable effect

A genuinely issued customer document link survived inbox redaction when represented as encoded text or inside a compressed PDF. The PDF bytes were forwarded by the captured mail adapter, and the preserved public link still served the synthetic document.

An ordinary supported mail provider producing the crafted PDF representation was not observed.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/campaign-2026-10-08/auth/run-dtc-002.mjs:194-215` (test source).

```javascript
    }
  } finally { await pool.end(); }
  const valid=await app.inject({method:'GET',url:new URL(canonical).pathname,remoteAddress:'198.51.100.15'});
  const encodedAction=await app.inject({method:'GET',url:new URL(encoded).pathname,remoteAddress:'198.51.100.16'});
  const wrong=await app.inject({method:'GET',url:`/api/public/paperwork/documents/${'B'.repeat(43)}`,remoteAddress:'198.51.100.17'});
  assert.equal(valid.statusCode,200,valid.body);assert.equal(encodedAction.statusCode,200,encodedAction.body);assert.equal(wrong.statusCode,404);
  assert.deepEqual(valid.rawPayload,encodedAction.rawPayload);
  const linkId=one(`select id from app.document_links where tenant_id=${quote(tenant)} and token_hash=digest(${quote(token)},'sha256')`);
  sql(`update app.document_links set expires_at=now()-interval '1 second' where id=${quote(linkId)}`);
  const expired=await app.inject({method:'GET',url:new URL(encoded).pathname,remoteAddress:'198.51.100.18'});
  assert.equal(expired.statusCode,404);
  assert.equal(filed[0].storedHasWorkingUrl,false);assert.equal(filed[1].storedHasWorkingUrl,true);
  assert.equal(filed[0].threadHasWorkingUrl,false);assert.equal(filed[1].threadHasWorkingUrl,true);
  output.lead030={issuedBy:'paperwork POST + fake outbox worker',fakeMailbox:'cleanMailboxMessage + fileIncoming + SQL store',tokenRedacted:true,origin,
    issuedRecipient:send.json().emailedTo,filed,public:{canonical:valid.statusCode,encoded:encodedAction.statusCode,wrong:wrong.statusCode,expired:expired.statusCode,equalBytes:true},
    disposition:'PROVED synthetic issued-token shared-inbox disclosure and public download; occurrence of percent-encoded link in real provider mail remains BLOCKED'};
  console.log(JSON.stringify(output,null,2));
} finally {
  if(app) await app.close();
  if(started) spawnSync(path.join(pgBin,'pg_ctl'),['-D',data,'-m','immediate','stop'],{cwd:repo,stdio:'ignore',timeout:30_000});
  rmSync(root,{recursive:true,force:true});
}
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A valid customer link arrives in an encoded or compressed representation that the filing path accepts and an inbox member reads or forwards it.

Recorded result: A genuinely issued customer document link survived inbox redaction when represented as encoded text or inside a compressed PDF. The PDF bytes were forwarded by the captured mail adapter, and the preserved public link still served the synthetic document.

Scope: An ordinary supported mail provider producing the crafted PDF representation was not observed. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/campaign-2026-10-08/auth/run-dtc-002.mjs:194-215` (test source).

```javascript
    }
  } finally { await pool.end(); }
  const valid=await app.inject({method:'GET',url:new URL(canonical).pathname,remoteAddress:'198.51.100.15'});
  const encodedAction=await app.inject({method:'GET',url:new URL(encoded).pathname,remoteAddress:'198.51.100.16'});
  const wrong=await app.inject({method:'GET',url:`/api/public/paperwork/documents/${'B'.repeat(43)}`,remoteAddress:'198.51.100.17'});
  assert.equal(valid.statusCode,200,valid.body);assert.equal(encodedAction.statusCode,200,encodedAction.body);assert.equal(wrong.statusCode,404);
  assert.deepEqual(valid.rawPayload,encodedAction.rawPayload);
  const linkId=one(`select id from app.document_links where tenant_id=${quote(tenant)} and token_hash=digest(${quote(token)},'sha256')`);
  sql(`update app.document_links set expires_at=now()-interval '1 second' where id=${quote(linkId)}`);
  const expired=await app.inject({method:'GET',url:new URL(encoded).pathname,remoteAddress:'198.51.100.18'});
  assert.equal(expired.statusCode,404);
  assert.equal(filed[0].storedHasWorkingUrl,false);assert.equal(filed[1].storedHasWorkingUrl,true);
  assert.equal(filed[0].threadHasWorkingUrl,false);assert.equal(filed[1].threadHasWorkingUrl,true);
  output.lead030={issuedBy:'paperwork POST + fake outbox worker',fakeMailbox:'cleanMailboxMessage + fileIncoming + SQL store',tokenRedacted:true,origin,
    issuedRecipient:send.json().emailedTo,filed,public:{canonical:valid.statusCode,encoded:encodedAction.statusCode,wrong:wrong.statusCode,expired:expired.statusCode,equalBytes:true},
    disposition:'PROVED synthetic issued-token shared-inbox disclosure and public download; occurrence of percent-encoded link in real provider mail remains BLOCKED'};
  console.log(JSON.stringify(output,null,2));
} finally {
  if(app) await app.close();
  if(started) spawnSync(path.join(pgBin,'pg_ctl'),['-D',data,'-m','immediate','stop'],{cwd:repo,stdio:'ignore',timeout:30_000});
  rmSync(root,{recursive:true,force:true});
}
```

## Remediation

Normalize supported link representations before filing and forwarding, and quarantine content that cannot be inspected within safe resource bounds.
