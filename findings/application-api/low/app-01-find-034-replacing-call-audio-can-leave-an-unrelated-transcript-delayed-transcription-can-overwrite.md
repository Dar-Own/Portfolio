# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low (conditional)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and a partial or modeled proof of the original behavior; the complete application or provider flow is not established by these excerpts.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Replacing call audio can leave an unrelated transcript; delayed transcription can overwrite newer text

## Description

A call can display a transcript from a replaced recording. An older transcription result can overwrite a later manual correction.

Prerequisite and affected workflow: A manager replaces recorded audio without clearing its transcript, or a provider result returns after an authorized transcript edit.

Established observation: The recording-replacement route leaves the existing transcript attached to the call. A controlled final-database ordering also accepted an older transcription result after a newer manual edit, replacing that edit because the write did not compare recording or text versions.

Limits: The complete HTTP and provider race, real provider behavior, and reliance on the displayed text remain untested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A manager replaces recorded audio without clearing its transcript, or a provider result returns after an authorized transcript edit.

Source: `apps/api/src/features/calls/routes.ts:112-135` (1d5d98db9acf).

```typescript
    app.post('/:id/audio', {
      schema: { params: idParams },
      bodyLimit: MAX_AUDIO_BYTES + 4096,
      // Before the body is read, so an oversized file gets our own message and not "That is too large."
      onRequest: async (request) => checkAudioSize(request.headers['content-length']),
    }, async (request) => {
      requirePermission(request.session, 'calls.manage');
      const { id } = request.params;
      const bytes = request.body;
      if (!Buffer.isBuffer(bytes)) throw invalid('Send the recording itself as the request body.');
      const { call, replacedKey } = await app.db.withUser(request.session, async (ctx) => {
        await requireCall(ctx, id);
        const previous = await callAudioFileId(ctx, id);
        const previousKey = previous ? await audioStorageKey(ctx, previous) : null;
        await storeAndAttachAudio(ctx, { bytes, store: app.fileStore }, (file) => attachCallAudio(ctx, id, file.fileId));
        // The database dropped the replaced recording's app.files row (migration 0066), unless something
        // else still uses it; only then do its bytes go too.
        const gone = previous !== null && (await audioStorageKey(ctx, previous)) === null;
        return { call: await requireCall(ctx, id), replacedKey: gone ? previousKey : null };
      });
      // After the commit, so a request that fails never loses the recording it kept.
      if (replacedKey) await discardAudio(app.fileStore, replacedKey);
      return { call };
    });
```

#### Step 2 — Cross the missing state or authority check

Replacing audio leaves the old transcript intact. A later transcription result checks the call ID but not the recording or transcript version.

Source: `packages/db/migrations/0066_calls_review_fixes.sql:75-97` (1d5d98db9acf).

```sql
-- ---------------------------------------------------------------------
-- 4. A replaced recording's file goes
-- ---------------------------------------------------------------------
-- AFTER the update, so the call no longer points at the old file. SECURITY DEFINER because only the
-- owner may delete app.files rows (0003); the guards on app.calls (0043, 0060) already decided that this
-- person may change the recording.
create or replace function app.drop_replaced_call_recording() returns trigger
language plpgsql security definer set search_path = app, pg_temp as $$
begin
  if old.recording_file is not null
     and old.recording_file is distinct from new.recording_file
     and app.feature_file_ok(old.tenant_id, old.recording_file, 'calls') then
    begin
      delete from app.files f where f.tenant_id = old.tenant_id and f.id = old.recording_file;
    exception when foreign_key_violation then
      null; -- still used elsewhere (another call, an email attachment): it stays
    end;
  end if;
  return null;
end $$;
revoke all on function app.drop_replaced_call_recording() from public;
create trigger calls_drop_replaced_recording after update of recording_file on app.calls
for each row execute function app.drop_replaced_call_recording();
```

Source: `apps/api/src/features/calls/store.ts:73-124` (1d5d98db9acf).

```typescript
         c.handled_by, hu.full_name as handled_by_name,
         c.recording_file, f.content_type as audio_type, f.size_bytes as audio_size,
         c.audio_uploaded_by, au.full_name as uploaded_by_name,
         c.transcript, c.transcript_source, c.transcribed_at,
         coalesce(cu.display_name, mine.name) as lead_name, coalesce(l.record_type::text, mine.record_type) as record_type,
         coalesce(l.stage::text, mine.stage) as stage, coalesce(l.ka_stage::text, mine.ka_stage) as ka_stage,
         coalesce(p.address_line, mine.address_line) as address_line, coalesce(p.city, mine.city) as city,
         l.id is not null as lead_readable,
         r.requested_by, ru.full_name as requested_by_name, r.reason as request_reason, r.created_at as requested_at
    from app.calls c
    left join app.leads l on l.tenant_id = c.tenant_id and l.id = c.lead_id
    left join app.customers cu on cu.tenant_id = l.tenant_id and cu.id = l.customer_id
    left join app.properties p on p.tenant_id = l.tenant_id and p.id = l.property_id
    left join lateral (select * from app.lead_card_taken_by_me(c.tenant_id, c.lead_id) where l.id is null) mine on true
    left join app.files f on f.tenant_id = c.tenant_id and f.id = c.recording_file
    left join app.users hu on hu.id = c.handled_by
    left join app.users au on au.id = c.audio_uploaded_by
    left join app.call_deletion_requests r on r.tenant_id = c.tenant_id and r.call_id = c.id
    left join app.users ru on ru.id = r.requested_by`;

function toCallRecord(row: CallRow): CallRecord {
  return {
    id: row.id,
    lead:
      row.lead_id && row.lead_name
        ? {
            id: row.lead_id,
            name: row.lead_name,
            address: row.address_line ? [row.address_line, row.city].filter(Boolean).join(', ') : null,
            stageLabel: row.record_type === 'key_account' ? kaStages.label(row.ka_stage) : leadStages.label(row.stage),
            canOpen: row.lead_readable,
          }
        : null,
    direction: row.direction,
    from: formatPhone(row.from_e164) || row.from_e164,
    fromE164: row.from_e164,
    to: formatPhone(row.to_e164) || row.to_e164,
    startedAt: row.started_at.toISOString(),
    durationS: row.duration_s,
    handledBy: row.handled_by ? { id: row.handled_by, fullName: row.handled_by_name ?? '' } : null,
    audio: row.recording_file
      ? {
          contentType: row.audio_type ?? 'application/octet-stream',
          sizeBytes: Number(row.audio_size ?? 0),
          uploadedBy: row.audio_uploaded_by ? { id: row.audio_uploaded_by, fullName: row.uploaded_by_name ?? '' } : null,
        }
      : null,
    transcript: row.transcript,
    transcriptSource: row.transcript_source,
    transcriptSourceLabel: transcriptSources.label(row.transcript_source),
    transcribedAt: row.transcribed_at ? row.transcribed_at.toISOString() : null,
    deletionRequest:
```


#### Step 3 — Observe the durable effect

The recording-replacement route leaves the existing transcript attached to the call. A controlled final-database ordering also accepted an older transcription result after a newer manual edit, replacing that edit because the write did not compare recording or text versions.

The complete HTTP and provider race, real provider behavior, and reliance on the displayed text remain untested.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/pocs/run-dtc-007-transcript-race.mjs:43-63` (synthetic test).

```javascript
      values ('${callAudio}','${tenant}','inbound','+12025550111','+12025550199',now(),'${fileA}'),
             ('${callManual}','${tenant}','inbound','+12025550111','+12025550199',now(),'${fileA}'),
             ('${foreignCall}','${otherTenant}','inbound','+12025550112','+12025550199',now(),null);`);
  const audioRead=userSql(`select 'audio='||recording_file from app.calls where tenant_id='${tenant}' and id='${callAudio}';`).match(/audio=([^\n]+)/)?.[1];
  assert.equal(audioRead,fileA);
  const original=callState(callAudio);assert.equal(original,`${fileA}|none|none`);
  userSql(`update app.calls set recording_file='${fileB}' where tenant_id='${tenant}' and id='${callAudio}';`);
  const afterReplace=callState(callAudio);assert.equal(afterReplace,`${fileB}|none|none`);
  userSql(`update app.calls set transcript='PROVIDER-FOR-A',transcript_source='provider',transcribed_at=now() where tenant_id='${tenant}' and id='${callAudio}';`);
  const staleAudio=callState(callAudio);assert.equal(staleAudio,`${fileB}|PROVIDER-FOR-A|provider`);
  const manualAudioRead=userSql(`select 'audio='||recording_file from app.calls where tenant_id='${tenant}' and id='${callManual}';`).match(/audio=([^\n]+)/)?.[1];
  assert.equal(manualAudioRead,fileA);
  userSql(`update app.calls set transcript='MANUAL-CORRECTION',transcript_source='manual',transcribed_at=now() where tenant_id='${tenant}' and id='${callManual}';`);
  const afterManual=callState(callManual);assert.equal(afterManual,`${fileA}|MANUAL-CORRECTION|manual`);
  userSql(`update app.calls set transcript='PROVIDER-OLDER',transcript_source='provider',transcribed_at=now() where tenant_id='${tenant}' and id='${callManual}';`);
  const staleManual=callState(callManual);assert.equal(staleManual,`${fileA}|PROVIDER-OLDER|provider`);
  const foreignBefore=callState(foreignCall);
  const foreignAttempt=userSql(`with changed as (update app.calls set transcript='SHOULD-NOT-WRITE',transcript_source='provider',transcribed_at=now() where tenant_id='${tenant}' and id='${foreignCall}' returning id) select 'foreign_changed='||count(*) from changed;`).match(/foreign_changed=(\d+)/)?.[1];
  assert.equal(foreignAttempt,'0');assert.equal(callState(foreignCall),foreignBefore);
  console.log(JSON.stringify({revision,migrations:migrations.length,isolatedDataDirectoryVerified:true,realPostgis:false,spatialColumnsShimmed:true,
    actor:'poc_dtc007_app via auth.act_as_session',audioRead,original,afterReplace,staleAudio,manualAudioRead,afterManual,staleManual,
```

## Proof of concept

The original proof isolates a narrower component or modeled transition. It does not demonstrate every step of the complete external workflow.

Setup: A manager replaces recorded audio without clearing its transcript, or a provider result returns after an authorized transcript edit.

Recorded result: The recording-replacement route leaves the existing transcript attached to the call. A controlled final-database ordering also accepted an older transcription result after a newer manual edit, replacing that edit because the write did not compare recording or text versions.

Scope: The complete HTTP and provider race, real provider behavior, and reliance on the displayed text remain untested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/pocs/run-dtc-007-transcript-race.mjs:43-63` (synthetic test).

```javascript
      values ('${callAudio}','${tenant}','inbound','+12025550111','+12025550199',now(),'${fileA}'),
             ('${callManual}','${tenant}','inbound','+12025550111','+12025550199',now(),'${fileA}'),
             ('${foreignCall}','${otherTenant}','inbound','+12025550112','+12025550199',now(),null);`);
  const audioRead=userSql(`select 'audio='||recording_file from app.calls where tenant_id='${tenant}' and id='${callAudio}';`).match(/audio=([^\n]+)/)?.[1];
  assert.equal(audioRead,fileA);
  const original=callState(callAudio);assert.equal(original,`${fileA}|none|none`);
  userSql(`update app.calls set recording_file='${fileB}' where tenant_id='${tenant}' and id='${callAudio}';`);
  const afterReplace=callState(callAudio);assert.equal(afterReplace,`${fileB}|none|none`);
  userSql(`update app.calls set transcript='PROVIDER-FOR-A',transcript_source='provider',transcribed_at=now() where tenant_id='${tenant}' and id='${callAudio}';`);
  const staleAudio=callState(callAudio);assert.equal(staleAudio,`${fileB}|PROVIDER-FOR-A|provider`);
  const manualAudioRead=userSql(`select 'audio='||recording_file from app.calls where tenant_id='${tenant}' and id='${callManual}';`).match(/audio=([^\n]+)/)?.[1];
  assert.equal(manualAudioRead,fileA);
  userSql(`update app.calls set transcript='MANUAL-CORRECTION',transcript_source='manual',transcribed_at=now() where tenant_id='${tenant}' and id='${callManual}';`);
  const afterManual=callState(callManual);assert.equal(afterManual,`${fileA}|MANUAL-CORRECTION|manual`);
  userSql(`update app.calls set transcript='PROVIDER-OLDER',transcript_source='provider',transcribed_at=now() where tenant_id='${tenant}' and id='${callManual}';`);
  const staleManual=callState(callManual);assert.equal(staleManual,`${fileA}|PROVIDER-OLDER|provider`);
  const foreignBefore=callState(foreignCall);
  const foreignAttempt=userSql(`with changed as (update app.calls set transcript='SHOULD-NOT-WRITE',transcript_source='provider',transcribed_at=now() where tenant_id='${tenant}' and id='${foreignCall}' returning id) select 'foreign_changed='||count(*) from changed;`).match(/foreign_changed=(\d+)/)?.[1];
  assert.equal(foreignAttempt,'0');assert.equal(callState(foreignCall),foreignBefore);
  console.log(JSON.stringify({revision,migrations:migrations.length,isolatedDataDirectoryVerified:true,realPostgis:false,spatialColumnsShimmed:true,
    actor:'poc_dtc007_app via auth.act_as_session',audioRead,original,afterReplace,staleAudio,manualAudioRead,afterManual,staleManual,
```

## Remediation

Clear or mark the transcript stale on audio replacement and accept provider results only when the captured recording and transcript versions still match.
