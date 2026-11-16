# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Medium (conditional provider-bound confidentiality)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Call audio is handed off after transcription is disabled

## Description

After transcription is turned off, the application can still submit a pending call-audio request for transcription and save its returned text.

Prerequisite and affected workflow: A transcription request reads audio while enabled, then the owner commits OFF before the pending provider call begins.

Established observation: The owner turned transcription off after audio selection but before the pending provider handoff. The application still passed the exact test recording to its transcription adapter and stored the returned text; requests started after OFF, lesser users, and foreign-company users were denied.

Limits: Real provider receipt and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A transcription request reads audio while enabled, then the owner commits OFF before the pending provider call begins.

Source: `apps/api/src/features/calls/routes.ts:206-230` (1d5d98db9acf).

```typescript
    app.post('/:id/transcribe', { schema: { params: idParams } }, async (request) => {
      requirePermission(request.session, 'calls.manage');
      const { id } = request.params;
      // The recording is read in one transaction, transcribed outside any transaction (the service may
      // take a while), and the transcript written in a second one. The call comes first, so a call that
      // isn't there (or is another company's) is "not found" whatever is connected.
      const provider = await providerFor(request.session);
      const { audio, speech } = await app.db.withUser(request.session, async (ctx) => {
        await requireCall(ctx, id);
        if (!provider) throw new HttpError(409, 'no_transcription_provider', NO_TRANSCRIPTION_PROVIDER_MESSAGE);
        const fileId = await callAudioFileId(ctx, id);
        if (!fileId) throw invalid('Upload the recording first, then transcribe it.');
        return { audio: await readAudio(ctx, { fileId, store: app.fileStore }), speech: provider };
      });
      let text: string;
      try {
        text = (await speech.transcribe({ bytes: audio.bytes, contentType: audio.contentType, fileName: `call.${audio.extension}` })).text;
      } catch (error) {
        request.log.warn({ err: error, provider: speech.name }, 'Transcription failed');
        throw new HttpError(502, 'transcription_failed', TRANSCRIPTION_FAILED_MESSAGE);
      }
      return app.db.withUser(request.session, async (ctx) => {
        await setCallTranscript(ctx, id, cleanTranscript(text), 'provider');
        return { call: await requireCall(ctx, id), simulated: speech.simulated };
      });
```

#### Step 2 — Cross the missing state or authority check

The transcription request continues using the earlier enabled setting when submitting the audio and saving the result.

Source: `apps/api/src/features/connections/routes.ts:89-120` (1d5d98db9acf).

```typescript
    app.put('/:key', { schema: { params: keyParams, body: saveBody } }, async (request) => {
      requirePermission(request.session, 'connections.manage');
      const session = request.session;
      const key = request.params.key;
      if (!isConnectionKey(key) || !(CONNECTION_KEYS as readonly string[]).includes(key)) throw notFound("That setting doesn't exist.");
      const value = request.body.value === null ? null : request.body.value.trim();
      if (value !== null) {
        const problem = connectionValueProblem(key, value);
        if (problem) throw invalid(problem);
        if (isSecretConnectionKey(key) && !app.config.secretsKey) throw new HttpError(409, 'no_master_key', MASTER_KEY_MISSING_MESSAGE);
      }

      // A fresh password check, the same as signing in (guesses count toward the lockout).
      const check = await confirmPassword(app.db, session, request.body.password, { ip: request.ip, userAgent: request.headers['user-agent'] });
      if (check === 'locked') throw locked();
      if (check !== 'ok') throw new HttpError(422, 'wrong_password', WRONG_PASSWORD_MESSAGE);

      // The owner's settings (0080, 0089): anyone else asks the owner, who gets a task with a one-click
      // approve (one request waits per setting: asking again replaces it, so nothing piles up). Which
      // settings those are is the database's answer, asked here, never a list kept in the API.
      const ownerOnly = await app.db.withUser(session, (ctx) => isOwnerOnlyChange(ctx, key, value));
      if (ownerOnly && !session.role.isOwner) {
        const id = await app.db.withUser(session, (ctx) => requestConnectionChange(ctx, key as ConnectionKey, value, app.config.secretsKey));
        request.log.info({ setting: key, removed: value === null, requestId: id }, "A company connection change was sent for the owner's approval");
        // waitingForOwner: the answer says what happened, so the page doesn't have to guess from the list of
        // waiting changes — which told the owner his own save was "sent to the owner" whenever somebody
        // else had already asked for that same setting (the review of this fix, finding 8).
        return { ...(await view(app, session)), waitingForOwner: true };
      }
      await app.db.withUser(session, (ctx) => saveConnection(ctx, key as ConnectionKey, value, app.config.secretsKey));
      request.log.info({ setting: key, removed: value === null }, 'A company connection setting was changed');
      return { ...(await view(app, session)), waitingForOwner: false };
```

Source: `packages/integrations/src/transcription.ts:80-107` (1d5d98db9acf).

```typescript
export function createOpenAiTranscriptionProvider(options: OpenAiTranscriptionOptions): TranscriptionProvider {
  const apiKey = options.apiKey.trim();
  if (apiKey === '') throw new Error('The OpenAI transcription service needs an API key on the Connections page.');
  const model = options.model ?? 'whisper-1';
  const endpoint = options.endpoint ?? '[redacted URL]';
  const fetchFn = options.fetchFn ?? fetch;
  return {
    name: 'openai',
    simulated: false,
    async transcribe(input) {
      const form = new FormData();
      form.append('file', new Blob([input.bytes], { type: input.contentType }), input.fileName);
      form.append('model', model);
      form.append('response_format', 'json');
      if (input.language) form.append('language', input.language);
      const response = await fetchFn(endpoint, {
        method: 'POST',
        headers: { Authorization: `Bearer ${apiKey}` },
        body: form,
      });
      if (!response.ok) {
        // The body may describe the key or the account; only the status is kept.
        throw new Error(`The transcription service answered ${response.status}.`);
      }
      const body = (await response.json()) as { text?: unknown };
      if (typeof body.text !== 'string') throw new Error('The transcription service answered without a transcript.');
      return { text: body.text };
    },
```


#### Step 3 — Observe the durable effect

The owner turned transcription off after audio selection but before the pending provider handoff. The application still passed the exact test recording to its transcription adapter and stored the returned text; requests started after OFF, lesser users, and foreign-company users were denied.

Real provider receipt and deployed effect remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/worker-authority-tail/run-call-transcription-effect-time.mjs:223-245` (synthetic test).

```javascript
  releasePause();
  const resumed = await withDeadline(pending, 'R037 resumed request');
  restoreWithUser(); restoreWithUser = null;
  const after = state(peach, raceId);
  result.race = { timeline: ['R037 read provider choice and decrypted inert audio in first transaction',
    'owner HTTP removed transcription.provider and committed', 'R037 handed audio to fake endpoint and wrote transcript'],
    read, before: raceBefore, atBarrier, ownerOff: summary(off), afterOff, resumed: summary(resumed), after,
    attemptsBefore: attemptsBeforeRace, attemptsAfter: result.fakeProviderAttempts.length };
  assert.equal(resumed.statusCode, 200, resumed.body);
  assert.equal(result.fakeProviderAttempts.length, attemptsBeforeRace + 1);
  assert.equal(result.fakeProviderAttempts.at(-1).audioSha256, read.audioSha256);
  assert.equal(after.provider, null);
  assert.equal(after.transcript, expectedNextText);
  assert.equal(after.transcript_source, 'provider');
  assert.equal(after.recording_file, raceBefore.recording_file);
  const afterOffAttempts = result.fakeProviderAttempts.length;
  const freshOff = await post(raceId, owner);
  result.controls.freshOff = { response: summary(freshOff), before: after, after: state(peach, raceId),
    attemptsBefore: afterOffAttempts, attemptsAfter: result.fakeProviderAttempts.length };
  assert.equal(freshOff.statusCode, 409, freshOff.body);
  assert.equal(result.fakeProviderAttempts.length, afterOffAttempts);
  assert.deepEqual(result.controls.freshOff.after, after);
  result.state = 'STALE-CONFIG-AUDIO-HANDOFF-PROVED-IN-LOCAL-FAKE-PROVIDER-SCOPE';
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A transcription request reads audio while enabled, then the owner commits OFF before the pending provider call begins.

Recorded result: The owner turned transcription off after audio selection but before the pending provider handoff. The application still passed the exact test recording to its transcription adapter and stored the returned text; requests started after OFF, lesser users, and foreign-company users were denied.

Scope: Real provider receipt and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/worker-authority-tail/run-call-transcription-effect-time.mjs:223-245` (synthetic test).

```javascript
  releasePause();
  const resumed = await withDeadline(pending, 'R037 resumed request');
  restoreWithUser(); restoreWithUser = null;
  const after = state(peach, raceId);
  result.race = { timeline: ['R037 read provider choice and decrypted inert audio in first transaction',
    'owner HTTP removed transcription.provider and committed', 'R037 handed audio to fake endpoint and wrote transcript'],
    read, before: raceBefore, atBarrier, ownerOff: summary(off), afterOff, resumed: summary(resumed), after,
    attemptsBefore: attemptsBeforeRace, attemptsAfter: result.fakeProviderAttempts.length };
  assert.equal(resumed.statusCode, 200, resumed.body);
  assert.equal(result.fakeProviderAttempts.length, attemptsBeforeRace + 1);
  assert.equal(result.fakeProviderAttempts.at(-1).audioSha256, read.audioSha256);
  assert.equal(after.provider, null);
  assert.equal(after.transcript, expectedNextText);
  assert.equal(after.transcript_source, 'provider');
  assert.equal(after.recording_file, raceBefore.recording_file);
  const afterOffAttempts = result.fakeProviderAttempts.length;
  const freshOff = await post(raceId, owner);
  result.controls.freshOff = { response: summary(freshOff), before: after, after: state(peach, raceId),
    attemptsBefore: afterOffAttempts, attemptsAfter: result.fakeProviderAttempts.length };
  assert.equal(freshOff.statusCode, 409, freshOff.body);
  assert.equal(result.fakeProviderAttempts.length, afterOffAttempts);
  assert.deepEqual(result.controls.freshOff.after, after);
  result.state = 'STALE-CONFIG-AUDIO-HANDOFF-PROVED-IN-LOCAL-FAKE-PROVIDER-SCOPE';
```

## Remediation

Recheck the transcription setting immediately before provider handoff and suppress new calls when OFF has already committed. Already-handed work may finish under the client rule.
