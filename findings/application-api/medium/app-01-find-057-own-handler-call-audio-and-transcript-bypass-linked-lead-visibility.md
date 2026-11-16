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

# Own-handler call audio and transcript bypass linked-lead visibility

## Description

A call handler can read exact audio bytes and transcript tied to a lead they cannot open.

Prerequisite and affected workflow: A same-company call handler requests the audio or transcript of a call linked to a lead they cannot view.

Established observation: A call handler was denied the linked lead but received the exact 812-byte call recording and its transcript through the call routes. Peer, lesser-role, and foreign-company requests were denied.

Limits: Disclosure of real customer content and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A same-company call handler requests the audio or transcript of a call linked to a lead they cannot view.

Source: `apps/api/src/features/calls/routes.ts:107-110` (1d5d98db9acf).

```typescript
    app.get('/:id', { schema: { params: idParams } }, async (request) => {
      requirePermission(request.session, 'calls.view');
      return app.db.withUser(request.session, async (ctx) => ({ call: await requireCall(ctx, request.params.id) }));
    });
```

#### Step 2 — Cross the missing state or authority check

Call media and transcript reads allow the assigned handler but do not apply the linked lead's visibility rule.

Source: `apps/api/src/features/calls/routes.ts:137-153` (1d5d98db9acf).

```typescript
    app.get('/:id/audio', { schema: { params: idParams } }, async (request, reply) => {
      requirePermission(request.session, 'calls.view');
      const { id } = request.params;
      const { audio, name } = await app.db.withUser(request.session, async (ctx): Promise<{ audio: AudioToStream; name: string }> => {
        const call = await requireCall(ctx, id);
        const fileId = await callAudioFileId(ctx, id);
        if (!fileId) throw invalid('This call has no recording yet.');
        return { audio: await openAudio(ctx, { fileId, store: app.fileStore }), name: call.startedAt.slice(0, 10) };
      });
      // inline, so the browser's player plays it; nosniff, so the browser takes our word for the type.
      return reply
        .header('content-type', audio.contentType)
        .header('content-length', String(audio.sizeBytes))
        .header('content-disposition', contentDisposition('inline', `call-${name}.${audio.extension}`))
        .header('x-content-type-options', 'nosniff')
        .header('cache-control', 'private, no-store')
        .send(audio.stream);
```


#### Step 3 — Observe the durable effect

A call handler was denied the linked lead but received the exact 812-byte call recording and its transcript through the call routes. Peer, lesser-role, and foreign-company requests were denied.

Disclosure of real customer content and deployed effect remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/attack-led/public-files/run-call-audio-owned-handler-lead-denied.mjs:250-275` (synthetic test).

```javascript
    assert.deepEqual(kimAudio.rawPayload, wav);
    assert.equal(kimAudio.headers['content-type'], 'audio/wav');
    assert.equal(kimAudio.headers['cache-control'], 'private, no-store');
    assert.equal(Number(kimAudio.headers['content-length']), wav.length);
    out.result = 'OWN-HANDLED-HIDDEN-LEAD-EXACT-AUDIO-DELIVERED-POLICY-OPEN';
  } else {
    assert.ok([403, 404].includes(kimAudio.statusCode), `unexpected own-handler audio status: ${kimAudio.statusCode}`);
    noWav(kimAudio, 'Kim own call');
    out.result = 'OWN-HANDLED-HIDDEN-LEAD-AUDIO-DENIED';
  }

  // Andre has calls.view but did not handle this hidden intake call; Tyler lacks calls.view; Bob is foreign.
  const andreCard = await request('R031-same-tenant-peer-card-denied', 'GET', `/api/calls/${callId}`, andre);
  assert.equal(andreCard.statusCode, 404, andreCard.body);
  const andreAudio = await request('R033-same-tenant-peer-byte-denied', 'GET', `/api/calls/${callId}/audio`, andre);
  assert.equal(andreAudio.statusCode, 404, andreAudio.body); noWav(andreAudio, 'Andre peer');
  const tylerAudio = await request('R033-no-calls-view-byte-denied', 'GET', `/api/calls/${callId}/audio`, tyler);
  assert.equal(tylerAudio.statusCode, 403, tylerAudio.body); noWav(tylerAudio, 'Tyler lesser');
  const foreignAudio = await request('R033-foreign-known-id-byte-denied', 'GET', `/api/calls/${callId}/audio`, foreign);
  assert.equal(foreignAudio.statusCode, 404, foreignAudio.body); noWav(foreignAudio, 'Bob foreign');
  const afterReads = state(callId);
  assert.deepEqual(afterReads, beforeReads, 'read requests changed call/file/event/audit rows');
  const diskAfter = { length: readFileSync(diskPath).length, sha256: sha(readFileSync(diskPath)) };
  assert.deepEqual(diskAfter, diskBefore, 'read requests changed encrypted stored bytes');
  out.effects = { beforeReads, afterReads, encryptedDiskBefore: diskBefore, encryptedDiskAfter: diskAfter,
    kimLeadDeniedStatus: leadDenied.statusCode, kimAudioStatus: kimAudio.statusCode,
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A same-company call handler requests the audio or transcript of a call linked to a lead they cannot view.

Recorded result: A call handler was denied the linked lead but received the exact 812-byte call recording and its transcript through the call routes. Peer, lesser-role, and foreign-company requests were denied.

Scope: Disclosure of real customer content and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/attack-led/public-files/run-call-audio-owned-handler-lead-denied.mjs:250-275` (synthetic test).

```javascript
    assert.deepEqual(kimAudio.rawPayload, wav);
    assert.equal(kimAudio.headers['content-type'], 'audio/wav');
    assert.equal(kimAudio.headers['cache-control'], 'private, no-store');
    assert.equal(Number(kimAudio.headers['content-length']), wav.length);
    out.result = 'OWN-HANDLED-HIDDEN-LEAD-EXACT-AUDIO-DELIVERED-POLICY-OPEN';
  } else {
    assert.ok([403, 404].includes(kimAudio.statusCode), `unexpected own-handler audio status: ${kimAudio.statusCode}`);
    noWav(kimAudio, 'Kim own call');
    out.result = 'OWN-HANDLED-HIDDEN-LEAD-AUDIO-DENIED';
  }

  // Andre has calls.view but did not handle this hidden intake call; Tyler lacks calls.view; Bob is foreign.
  const andreCard = await request('R031-same-tenant-peer-card-denied', 'GET', `/api/calls/${callId}`, andre);
  assert.equal(andreCard.statusCode, 404, andreCard.body);
  const andreAudio = await request('R033-same-tenant-peer-byte-denied', 'GET', `/api/calls/${callId}/audio`, andre);
  assert.equal(andreAudio.statusCode, 404, andreAudio.body); noWav(andreAudio, 'Andre peer');
  const tylerAudio = await request('R033-no-calls-view-byte-denied', 'GET', `/api/calls/${callId}/audio`, tyler);
  assert.equal(tylerAudio.statusCode, 403, tylerAudio.body); noWav(tylerAudio, 'Tyler lesser');
  const foreignAudio = await request('R033-foreign-known-id-byte-denied', 'GET', `/api/calls/${callId}/audio`, foreign);
  assert.equal(foreignAudio.statusCode, 404, foreignAudio.body); noWav(foreignAudio, 'Bob foreign');
  const afterReads = state(callId);
  assert.deepEqual(afterReads, beforeReads, 'read requests changed call/file/event/audit rows');
  const diskAfter = { length: readFileSync(diskPath).length, sha256: sha(readFileSync(diskPath)) };
  assert.deepEqual(diskAfter, diskBefore, 'read requests changed encrypted stored bytes');
  out.effects = { beforeReads, afterReads, encryptedDiskBefore: diskBefore, encryptedDiskAfter: diskAfter,
    kimLeadDeniedStatus: leadDenied.statusCode, kimAudioStatus: kimAudio.statusCode,
```

## Remediation

Require permission to view the linked lead before returning call audio or transcript, while preserving access to authorized unlinked calls.
