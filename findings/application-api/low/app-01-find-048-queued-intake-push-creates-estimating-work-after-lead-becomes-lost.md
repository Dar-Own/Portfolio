# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low (conditional workflow integrity)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Queued Intake push creates estimating work after lead becomes Lost

## Description

A Lost lead can receive a new open estimator task and internal alert after staff have ended the intake workflow.

Prerequisite and affected workflow: Intake staff send a lead to estimating, then mark it Lost before the queued estimating work runs.

Established observation: An Intake push queued estimating work, then staff marked the lead Lost before the event handler ran. The handler nevertheless created an open estimator task and internal alert for the Lost lead.

Limits: Natural event lag and deployed effect remain unmeasured. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

Intake staff send a lead to estimating, then mark it Lost before the queued estimating work runs.

Source: `apps/api/src/features/intake/routes.ts:150-188` (1d5d98db9acf).

```typescript
    app.post('/leads/:id/push', { schema: { params: leadParams, body: pushBody } }, async (request): Promise<PushResultView> => {
      requireRouting(request.session);
      return app.db.withUser(request.session, async (ctx) => {
        const setup = await loadCompanySetup(ctx);
        const lead = await requireLead(ctx, request.params.id);
        pushableLead(lead);
        const picked = request.body.tier ?? null;
        const decision = decidePush(tierInputOf(lead, picked ?? lead.tier), setup.tiers);
        if (decision.action === 'pick_tier') throw ruleViolation(decision.message);
        if (decision.action === 'measure') throw ruleViolation(decision.message);
        if (decision.action === 'push' && decision.checkGates) {
          const missing = pushBlockers(gateInputOf(lead, decision.tier));
          if (missing.length > 0) throw ruleViolation(`Can't push yet. Missing: ${missing.join(', ')}`);
        }
        const result = await pushLead(ctx, lead.id, { tier: picked });
        switch (result.action) {
          case 'pushed': {
            const tierLabel = leadTiers.label(result.tier);
            await addActivityNote(ctx, pushActivityText(lead.name, tierLabel, result.assignedTo.fullName), lead.id);
            return {
              action: 'pushed', leadId: lead.id, tier: result.tier, tierLabel, ownerName: result.assignedTo.fullName,
              lossReason: null, message: `Pushed to ${result.assignedTo.fullName}`,
            };
          }
          case 'declined':
            // pushLead already closed it as Lost; only the loss note and the template are left.
            return finishDecline(ctx, lead, result.lossReason, true);
          case 'auto_estimate':
            await addActivityNote(ctx, `${lead.name} set to Tier 3 for an automatic estimate`, lead.id);
            ctx.emit('intake.auto_estimate', lead.id);
            return {
              action: 'auto_estimate', leadId: lead.id, tier: 'tier_3', tierLabel: leadTiers.label('tier_3'), ownerName: null,
              lossReason: null, message: 'Tier 3: auto-estimate created',
            };
          case 'measure':
            throw ruleViolation(result.message);
        }
      });
    });
```

#### Step 2 — Cross the missing state or authority check

The estimating worker trusts an earlier intake event and assignment without checking whether the lead is still active.

Source: `apps/api/src/core/domain/leads.ts:939-952` (1d5d98db9acf).

```typescript
    case 'push': {
      const ownerId = (await getRoutingRules(ctx))[decision.routingKey];
      if (!ownerId) {
        throw ruleViolation(`Pick someone for "${routingKeys.label(decision.routingKey)}" in Automations routing first.`);
      }
      const result = await ctx.client
        .query(`update app.leads set tier = $2, assigned_to = $3, stage = 'estimating', pushed_at = now() where id = $1`, [leadId, decision.tier, ownerId])
        .catch((error: unknown) => {
          throw asRuleViolation(error);
        });
      if (result.rowCount === 0) throw leadNotFound();
      const owner = (await ctx.client.query<{ full_name: string }>('select full_name from app.users where id = $1', [ownerId])).rows[0];
      ctx.emit('lead.stage_changed', leadId);
      ctx.emit('lead.pushed', leadId);
```

Source: `apps/api/src/features/intake/router.ts:134-169` (1d5d98db9acf).

```typescript
export async function onLeadPushed(ctx: DbContext, leadId: string): Promise<void> {
  const lead = await getLead(ctx, leadId);
  if (!lead?.pushedAt || !lead.tier || !lead.assignedTo) return;
  const pushed: { leadId: string; tier: LeadTier; assignedTo: string } = { leadId, tier: lead.tier, assignedTo: lead.assignedTo.id };
  const setup = await loadCompanySetup(ctx);
  const isPlanHub = pushed.tier === 'planhub';
  const created = await createTask(ctx, {
    title: isPlanHub ? PLANHUB_TASK_TITLE : ESTIMATE_TASK_TITLE,
    kind: 'estimate',
    ownerId: pushed.assignedTo,
    leadId: pushed.leadId,
    dueAt: estimateTaskDueAt(
      setup.now,
      setup.slaLimits.estimating,
      isPlanHub && lead.bidDueAt ? new Date(lead.bidDueAt) : null,
      setup.hours,
    ),
    dedupeKey: `estimate:${pushed.leadId}:${lead.pushedAt ?? lead.stageEnteredAt}`,
  });
  if (!created.created) return;

  const names = await userNames(ctx, [pushed.assignedTo, setup.routing.escalation]);
  const ownerName = names.get(pushed.assignedTo) ?? '';
  const path = `/pipelines/${lead.id}`;
  await notify(ctx, '#estimating', pushSlackText(leadTiers.label(pushed.tier), lead.name, tierArea(lead), ownerName), {
    forUsers: [pushed.assignedTo],
    path,
    textSummary: `${lead.name} (${leadTiers.label(pushed.tier)}) went to ${ownerName || 'an estimator'} to estimate.`,
  });
  if (pushed.tier === 'tier_1' && setup.routing.escalation && setup.routing.escalation !== pushed.assignedTo) {
    await notify(ctx, '#estimating', tier1HeadsUpSlackText(names.get(setup.routing.escalation) ?? '', lead.name), {
      forUsers: [setup.routing.escalation],
      path,
      textSummary: `Tier 1 heads-up: ${lead.name} went to ${ownerName}.`,
    });
  }
```


#### Step 3 — Observe the durable effect

An Intake push queued estimating work, then staff marked the lead Lost before the event handler ran. The handler nevertheless created an open estimator task and internal alert for the Lost lead.

Natural event lag and deployed effect remain unmeasured.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/attack-led/authority-data/run-intake-push-stale-event.mjs:163-186` (test source).

```javascript
  assert.equal(preLoss.lead.stage, 'estimating');
  assert.equal(estimateTasks(preLoss).length, 0);
  const lost = await call('authorized-lost-before-event-drain', 'POST', `/api/leads/${staleId}/status`, maya.headers, { status: 'lost', lossReason: 'no_response' }, staleId, staleName);
  assert.equal(lost.statusCode, 200, lost.body);
  const preDrain = state(staleId, staleName);
  assert.equal(preDrain.lead.stage, 'lost');
  assert.equal(estimateTasks(preDrain).length, 0);
  await waitForEvents(app);
  const postDrain = state(staleId, staleName);
  raw.cases.push({ label: 'lost-after-worker-drain', state: postDrain, eventErrors: eventErrors(app) });
  assert.equal(postDrain.lead.stage, 'lost');
  assert.equal(eventErrors(app).length, 0);
  assert.equal(raw.providerAttempts, 0);
  raw.verdict = estimateTasks(postDrain).some((item) => item.status === 'open') && estimatePings(postDrain).length > 0
    ? 'PROVED-TECHNICAL-BEHAVIOR' : 'FALSIFIED-IN-TESTED-SCOPE';
  raw.conclusion = raw.verdict === 'PROVED-TECHNICAL-BEHAVIOR'
    ? 'An authorized Lost correction committed before the worker effect, yet the queued push event created an open estimator task and #estimating notification; unchanged push worked and lesser/foreign pushes changed no business effects.'
    : 'The Lost correction prevented a new open estimate task or estimating notification in this ordered test.';
} catch (error) {
  raw.error = { name: error?.name ?? 'Error', message: String(error?.message ?? error), stack: String(error?.stack ?? '') };
  process.exitCode = 1;
} finally {
  try { await app?.close(); } catch (error) { raw.cleanupError = String(error); process.exitCode = 1; }
  if (started) {
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: Intake staff send a lead to estimating, then mark it Lost before the queued estimating work runs.

Recorded result: An Intake push queued estimating work, then staff marked the lead Lost before the event handler ran. The handler nevertheless created an open estimator task and internal alert for the Lost lead.

Scope: Natural event lag and deployed effect remain unmeasured. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/attack-led/authority-data/run-intake-push-stale-event.mjs:163-186` (test source).

```javascript
  assert.equal(preLoss.lead.stage, 'estimating');
  assert.equal(estimateTasks(preLoss).length, 0);
  const lost = await call('authorized-lost-before-event-drain', 'POST', `/api/leads/${staleId}/status`, maya.headers, { status: 'lost', lossReason: 'no_response' }, staleId, staleName);
  assert.equal(lost.statusCode, 200, lost.body);
  const preDrain = state(staleId, staleName);
  assert.equal(preDrain.lead.stage, 'lost');
  assert.equal(estimateTasks(preDrain).length, 0);
  await waitForEvents(app);
  const postDrain = state(staleId, staleName);
  raw.cases.push({ label: 'lost-after-worker-drain', state: postDrain, eventErrors: eventErrors(app) });
  assert.equal(postDrain.lead.stage, 'lost');
  assert.equal(eventErrors(app).length, 0);
  assert.equal(raw.providerAttempts, 0);
  raw.verdict = estimateTasks(postDrain).some((item) => item.status === 'open') && estimatePings(postDrain).length > 0
    ? 'PROVED-TECHNICAL-BEHAVIOR' : 'FALSIFIED-IN-TESTED-SCOPE';
  raw.conclusion = raw.verdict === 'PROVED-TECHNICAL-BEHAVIOR'
    ? 'An authorized Lost correction committed before the worker effect, yet the queued push event created an open estimator task and #estimating notification; unchanged push worked and lesser/foreign pushes changed no business effects.'
    : 'The Lost correction prevented a new open estimate task or estimating notification in this ordered test.';
} catch (error) {
  raw.error = { name: error?.name ?? 'Error', message: String(error?.message ?? error), stack: String(error?.stack ?? '') };
  process.exitCode = 1;
} finally {
  try { await app?.close(); } catch (error) { raw.cleanupError = String(error); process.exitCode = 1; }
  if (started) {
```

## Remediation

Check that the lead is still eligible for estimating before creating the task and alert, and close obsolete work when a lead becomes Lost.
