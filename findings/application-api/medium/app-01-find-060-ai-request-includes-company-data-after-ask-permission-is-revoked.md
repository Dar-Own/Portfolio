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

# AI request includes company data after Ask permission is revoked

## Description

After Ask access is revoked, a company data snapshot can still be handed to the model request path.

Prerequisite and affected workflow: Ask builds a company snapshot, then the owner revokes the requester's permission before the model request is submitted.

Established observation: The owner revoked Ask access after the application assembled a company snapshot but before model handoff. An exact synthetic bill marker still reached the model adapter, even though the final answer was denied.

Limits: Natural overlap and real Anthropic receipt remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

Ask builds a company snapshot, then the owner revokes the requester's permission before the model request is submitted.

Source: `apps/api/src/features/ask/routes.ts:206-246` (1d5d98db9acf).

```typescript
    app.post('/threads/:id/messages', { schema: { params: idParams, body: messageBody }, config: { rateLimit: questionLimit } }, async (request, reply) => {
      requirePermission(request.session, 'ask.use');
      const client = await claudeClientFor(app, request.session);
      requireConnected(client);
      const { id } = request.params;
      const question = request.body.body;

      // 1. Store the question and read the snapshot.
      const prepared = await app.db.withUser(request.session, async (ctx) => {
        await requireThread(ctx, id);
        const history = await listMessages(ctx, id);
        const setup = await loadCompanySetup(ctx);
        const snapshot: BusinessSnapshot = await buildSnapshot(ctx, setup, request.session);
        const userMessageId = await insertQuestion(ctx, id, question);
        return { history, snapshot, userMessageId };
      });

      // 2. Ask the model, outside any transaction. conversationTurns puts the conversation in the
      //    shape the service needs: it starts with the person and the two sides take turns, so
      //    questions the service never answered travel as one turn with the new question.
      const turns = conversationTurns(prepared.history.slice(-HISTORY_LIMIT), question);
      let answer: string;
      try {
        const result = await client.complete({ system: systemPrompt(prepared.snapshot.company, renderSnapshot(prepared.snapshot)), messages: turns });
        answer = result.text.trim() === '' ? NO_ANSWER_TEXT : result.text.trim();
      } catch (error) {
        request.log.warn(serviceFailureLog(error), '[redacted name]: the AI service failed');
        throw serviceError(error);
      }

      // 3. Store the answer in the person's own conversation (app.record_ask_answer, 0106), then read it back.
      const stored = await app.db.withUser(request.session, async (ctx) => {
        const assistantId = await recordAnswer(ctx, id, answer, new Date(prepared.snapshot.takenAt));
        const messages = await listMessages(ctx, id);
        return {
          thread: await requireThread(ctx, id),
          question: messages.find((message) => message.id === prepared.userMessageId) ?? null,
          answer: messages.find((message) => message.id === assistantId) ?? null,
        };
      });
      return reply.status(201).send(stored);
```

#### Step 2 — Cross the missing state or authority check

Ask checks permission when building the data snapshot but does not check it again before submitting the model request.

Source: `packages/db/migrations/0106_automation_leaves_the_api.sql:678-695` (1d5d98db9acf).

```sql
-- The answer in [redacted name] (0065's function, one change): written into the conversation of the person
-- who asked, by that person's own transaction (ask.use), instead of by the Automation member.
create or replace function app.record_ask_answer(p_thread uuid, p_body text, p_snapshot_at timestamptz) returns uuid
language plpgsql security definer set search_path = app, pg_temp as $$
declare
  answer_id uuid;
begin
  if app.current_user_id() is null or not app.is_member() or not app.can('ask.use') then
    raise exception 'An answer is written only into the conversation of the person who asked';
  end if;
  if not exists (select 1 from app.ask_threads t
                  where t.tenant_id = app.current_tenant() and t.id = p_thread and t.created_by = app.current_user_id()) then
    raise exception 'We couldn''t find that conversation';
  end if;
  insert into app.ask_messages (tenant_id, thread_id, role, body, snapshot_at)
  values (app.current_tenant(), p_thread, 'assistant', p_body, p_snapshot_at)
  returning id into answer_id;
  return answer_id;
```


#### Step 3 — Observe the durable effect

The owner revoked Ask access after the application assembled a company snapshot but before model handoff. An exact synthetic bill marker still reached the model adapter, even though the final answer was denied.

Natural overlap and real Anthropic receipt remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/ask-revocation/run-ask-post-snapshot-revocation.mjs:204-228` (synthetic test).

```javascript
  releaseResolve();
  const raceResponse = await timeout(pending, 'resumed R005 request');
  const afterRace = state(peach, role.id);
  output.race = { before: raceBefore, atPause, ownerRevocation: summary(revokeResponse), afterRevoke,
    response: summary(raceResponse), afterRace, modelCallsBefore: raceModelBefore,
    modelCallsAfter: fake.requests.length, modelRequest: fake.requests.at(-1) };
  assert.equal(fake.requests.length - raceModelBefore, 1, 'revoked snapshot did not reach fake model');
  assert.ok(output.race.modelRequest.system.includes('Rapid Haul Trucking'), 'revoked snapshot lacked sensitive seeded bill');
  assert.equal(afterRace.messages.count, afterRevoke.messages.count, 'revoked request stored assistant answer');
  assert.equal(afterRace.threads.count, afterRevoke.threads.count);
  assert.equal(afterRace.outbox.count, afterRevoke.outbox.count);
  assert.equal(afterRace.events.count, afterRevoke.events.count);
  assert.notEqual(raceResponse.statusCode, 201, 'revoked request returned stored answer');
  restoreWithUser(); restoreWithUser = null;

  const preRevokedBefore = state(peach, role.id);
  const preRevokedCalls = fake.requests.length;
  const preRevoked = await app.inject({ method: 'POST', url: `/api/ask/threads/${raceThread}/messages`, headers: vp.headers,
    payload: { body: 'Try after revocation' } });
  output.controls.preRequestRevoked = { response: summary(preRevoked), before: preRevokedBefore,
    after: state(peach, role.id), modelCallsBefore: preRevokedCalls, modelCallsAfter: fake.requests.length };
  assert.equal(preRevoked.statusCode, 403, preRevoked.body);
  assert.deepEqual(output.controls.preRequestRevoked.after, preRevokedBefore);
  assert.equal(fake.requests.length, preRevokedCalls);
  output.status = 'PROVED-LOCAL-POST-COMMIT-MODEL-HANDOFF';
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: Ask builds a company snapshot, then the owner revokes the requester's permission before the model request is submitted.

Recorded result: The owner revoked Ask access after the application assembled a company snapshot but before model handoff. An exact synthetic bill marker still reached the model adapter, even though the final answer was denied.

Scope: Natural overlap and real Anthropic receipt remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/ask-revocation/run-ask-post-snapshot-revocation.mjs:204-228` (synthetic test).

```javascript
  releaseResolve();
  const raceResponse = await timeout(pending, 'resumed R005 request');
  const afterRace = state(peach, role.id);
  output.race = { before: raceBefore, atPause, ownerRevocation: summary(revokeResponse), afterRevoke,
    response: summary(raceResponse), afterRace, modelCallsBefore: raceModelBefore,
    modelCallsAfter: fake.requests.length, modelRequest: fake.requests.at(-1) };
  assert.equal(fake.requests.length - raceModelBefore, 1, 'revoked snapshot did not reach fake model');
  assert.ok(output.race.modelRequest.system.includes('Rapid Haul Trucking'), 'revoked snapshot lacked sensitive seeded bill');
  assert.equal(afterRace.messages.count, afterRevoke.messages.count, 'revoked request stored assistant answer');
  assert.equal(afterRace.threads.count, afterRevoke.threads.count);
  assert.equal(afterRace.outbox.count, afterRevoke.outbox.count);
  assert.equal(afterRace.events.count, afterRevoke.events.count);
  assert.notEqual(raceResponse.statusCode, 201, 'revoked request returned stored answer');
  restoreWithUser(); restoreWithUser = null;

  const preRevokedBefore = state(peach, role.id);
  const preRevokedCalls = fake.requests.length;
  const preRevoked = await app.inject({ method: 'POST', url: `/api/ask/threads/${raceThread}/messages`, headers: vp.headers,
    payload: { body: 'Try after revocation' } });
  output.controls.preRequestRevoked = { response: summary(preRevoked), before: preRevokedBefore,
    after: state(peach, role.id), modelCallsBefore: preRevokedCalls, modelCallsAfter: fake.requests.length };
  assert.equal(preRevoked.statusCode, 403, preRevoked.body);
  assert.deepEqual(output.controls.preRequestRevoked.after, preRevokedBefore);
  assert.equal(fake.requests.length, preRevokedCalls);
  output.status = 'PROVED-LOCAL-POST-COMMIT-MODEL-HANDOFF';
```

## Remediation

Recheck current Ask authority at the provider handoff and discard the pending snapshot after revocation.
