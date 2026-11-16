# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low (internal workflow integrity)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# End-of-day job creates a stale email task after final inbox triage

## Description

End-of-day processing can create a stale email task and ping after staff clear the last inbox item.

Prerequisite and affected workflow: The worker counts one open item, staff clear it, and the worker then saves a task for that item.

Established observation: Staff triaged the last open inbox item after the end-of-day worker counted one item but before it saved its work. The count reached zero, yet the worker created an email task and alert; pre-triaged and foreign-company controls did not.

Limits: External send and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

The worker counts one open item, staff clear it, and the worker then saves a task for that item.

Source: `apps/worker/src/jobs/eod-reminder.ts:63-73` (1d5d98db9acf).

```typescript
async function remindTenant(client: Client, tenantId: string, now: Date, tally: EodTally): Promise<void> {
  const settings = await loadAutomationSettings(client, tenantId);
  if (!flagOn(settings, AUTOMATION_KEYS.endOfDay)) return;
  if (!endOfDayReminderDue(now, settings.hours)) return;

  const waiting = await client.query<{ left: string }>(
    `select count(*)::text as left from app.email_messages where tenant_id = $1 and triage is null`,
    [tenantId],
  );
  const emailsLeft = Number(waiting.rows[0]?.left ?? '0');
  if (emailsLeft === 0) return;
```

#### Step 2 — Cross the missing state or authority check

End-of-day processing uses an earlier inbox count without checking again after staff triage.

Source: `apps/api/src/features/inbox/routes.ts:527-547` (1d5d98db9acf).

```typescript
    app.post('/emails/:id/triage', { schema: { params: emailParams, body: triageBody } }, async (request): Promise<TriageResult> => {
      requireInboxAccess(request.session);
      const body = request.body;
      return app.db.withUser(request.session, async (ctx) => {
        const setup = await loadCompanySetup(ctx);
        const email = await requireEmail(ctx, request.params.id, setup.now);
        switch (body.bucket) {
          case 'lead':
            return triageAsLead(ctx, email, setup);
          case 'bid_invite':
            return triageAsBidInvite(ctx, email, { name: body.name ?? '', dueOn: body.dueOn ?? '', address: body.address ?? null }, setup);
          case 'active_job':
            if (!body.leadId) throw invalid('Pick the job to log it on.');
            return triageAsActiveJob(ctx, email, body.leadId, setup);
          case 'vendor_admin':
            return triageAsVendorAdmin(ctx, email, setup, request.session);
          case 'noise':
            return triageAsNoise(ctx, email, setup);
        }
      });
    });
```


#### Step 3 — Observe the durable effect

Staff triaged the last open inbox item after the end-of-day worker counted one item but before it saved its work. The count reached zero, yet the worker created an email task and alert; pre-triaged and foreign-company controls did not.

External send and deployed effect remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/attack-led/workers-bulk/run-j08-last-triage.mjs:154-177` (test source).

```javascript
  const preEmail=one(`select id from app.email_messages where tenant_id=${q(peach)} and provider_message_id=${q(`j08-pre-${runId}@local.test`)}`);
  const preSort=await triage(owner.headers,preEmail);
  assert.equal(preSort.statusCode,200,preSort.body);
  const rivalSort=await triage(rivalOwner.headers,rivalEmail);
  assert.equal(rivalSort.statusCode,200,rivalSort.body);
  assert.equal(effects(peach).untriaged,0);
  assert.equal(effects(rival).untriaged,0);
  const beforeNext={peach:effects(peach),rival:effects(rival)};
  workerNow=new Date('2026-10-19T20:45:00.000Z'); // next business day, Monday 4:45 PM Eastern.
  const nextDay=await runEodReminder(ctx);
  const afterNext={peach:effects(peach),rival:effects(rival)};
  assert.equal(nextDay.processed,0,JSON.stringify(nextDay));
  assert.deepEqual(afterNext,beforeNext,'pretriaged next day created an effect');
  result.controls.pretriaged={preEmail,preSortStatus:preSort.statusCode,rivalSortStatus:rivalSort.statusCode,workerNow:workerNow.toISOString(),before:beforeNext,nextDay,after:afterNext};
  result.state='PROVED-STALE-J08-TASK-AND-PING-AFTER-LAST-HTTP-TRIAGE-LOCAL';
}catch(error){result.failure={message:error?.message??String(error),stack:error?.stack};process.exitCode=1;
}finally{
  if(releasePause)releasePause();
  try{if(app)await app.close();}catch(error){result.closeError=String(error);}
  try{if(ctx)await ctx.close();}catch(error){result.contextCloseError=String(error);}
  if(started){const stop=spawnSync(path.join(pgBin,'pg_ctl'),['-D',data,'-m','immediate','-w','stop'],{encoding:'utf8',timeout:30_000});result.boundary.stopStatus=stop.status;}
  if(scratch.startsWith('/private/tmp/aos-j08-triage-'))rmSync(scratch,{recursive:true,force:true});
  writeFileSync(resultPath,JSON.stringify(result,null,2)+'\n',{mode:0o600});
  process.stdout.write(`${result.state} ${resultPath}\n`);
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: The worker counts one open item, staff clear it, and the worker then saves a task for that item.

Recorded result: Staff triaged the last open inbox item after the end-of-day worker counted one item but before it saved its work. The count reached zero, yet the worker created an email task and alert; pre-triaged and foreign-company controls did not.

Scope: External send and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/attack-led/workers-bulk/run-j08-last-triage.mjs:154-177` (test source).

```javascript
  const preEmail=one(`select id from app.email_messages where tenant_id=${q(peach)} and provider_message_id=${q(`j08-pre-${runId}@local.test`)}`);
  const preSort=await triage(owner.headers,preEmail);
  assert.equal(preSort.statusCode,200,preSort.body);
  const rivalSort=await triage(rivalOwner.headers,rivalEmail);
  assert.equal(rivalSort.statusCode,200,rivalSort.body);
  assert.equal(effects(peach).untriaged,0);
  assert.equal(effects(rival).untriaged,0);
  const beforeNext={peach:effects(peach),rival:effects(rival)};
  workerNow=new Date('2026-10-19T20:45:00.000Z'); // next business day, Monday 4:45 PM Eastern.
  const nextDay=await runEodReminder(ctx);
  const afterNext={peach:effects(peach),rival:effects(rival)};
  assert.equal(nextDay.processed,0,JSON.stringify(nextDay));
  assert.deepEqual(afterNext,beforeNext,'pretriaged next day created an effect');
  result.controls.pretriaged={preEmail,preSortStatus:preSort.statusCode,rivalSortStatus:rivalSort.statusCode,workerNow:workerNow.toISOString(),before:beforeNext,nextDay,after:afterNext};
  result.state='PROVED-STALE-J08-TASK-AND-PING-AFTER-LAST-HTTP-TRIAGE-LOCAL';
}catch(error){result.failure={message:error?.message??String(error),stack:error?.stack};process.exitCode=1;
}finally{
  if(releasePause)releasePause();
  try{if(app)await app.close();}catch(error){result.closeError=String(error);}
  try{if(ctx)await ctx.close();}catch(error){result.contextCloseError=String(error);}
  if(started){const stop=spawnSync(path.join(pgBin,'pg_ctl'),['-D',data,'-m','immediate','-w','stop'],{encoding:'utf8',timeout:30_000});result.boundary.stopStatus=stop.status;}
  if(scratch.startsWith('/private/tmp/aos-j08-triage-'))rmSync(scratch,{recursive:true,force:true});
  writeFileSync(resultPath,JSON.stringify(result,null,2)+'\n',{mode:0o600});
  process.stdout.write(`${result.state} ${resultPath}\n`);
```

## Remediation

Recheck the current inbox count at the final task and ping write, suppressing zero-count work.
