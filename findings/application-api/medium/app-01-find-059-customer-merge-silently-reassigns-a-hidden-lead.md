# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Medium (same-tenant integrity)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Customer merge silently reassigns a hidden lead

## Description

Customer merge can silently reassign a lead hidden from the acting editor, changing the owner's view of that lead.

Prerequisite and affected workflow: A stepped-up customer editor merges customers while a linked same-company lead is outside the editor's lead scope.

Established observation: A stepped-up customer editor was denied a linked lead, then merged its customer into another record. The hidden lead moved and its authorized owner's view changed, while the merge response reported zero hidden leads moved; missing step-up and foreign-company controls were denied.

Limits: Downstream finance or provider effects and deployed behavior remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A stepped-up customer editor merges customers while a linked same-company lead is outside the editor's lead scope.

Source: `apps/api/src/features/customers/merge.ts:225-270` (1d5d98db9acf).

```typescript
export function registerMergeRoutes(app: ApiApp): void {
  app.get('/duplicates', async (request): Promise<DuplicatesPage> => {
    requireEdit(request.session);
    return app.db.withUser(request.session, listDuplicates);
  });

  app.get('/:id/merge/:withId', { schema: { params: duplicateParams } }, async (request): Promise<MergePreview> => {
    const session = request.session;
    requireEdit(session);
    const { id, withId } = request.params;
    if (id === withId) throw invalid('That is the same customer: pick the duplicate to merge into it.');
    const money = canSeeCustomerMoney(session);
    return app.db.withUser(session, async (ctx) => {
      // An old merge link, or a second tab: say where the merged one went.
      await refuseMerged(ctx, [id, withId]);
      const keep = await loadSide(ctx, id, money, session.company.timezone);
      const duplicate = await loadSide(ctx, withId, money, session.company.timezone);
      const { result, refused } = await rehearse(ctx, id, withId);
      const seen = duplicate.counts;
      const hidden = result !== null
        && (result.leads > seen.leads || result.messages > seen.messages || result.tickets > seen.tickets || (seen.invoices !== null && result.invoices > seen.invoices));
      return { keep, duplicate, plan: result?.plan ?? null, hidden, refused };
    });
  });

  // Hard to undo and it moves everything: the password again (core/auth/plugin.ts), after the edit keys.
  app.post('/:id/merge', {
    schema: { params: mergeParams, body: mergeBody },
    config: { dangerous: true, permission: WORK_LEAD_PERMISSIONS },
  }, async (request): Promise<{ id: string; moved: MergeMoved; message: string }> => {
    const session = request.session;
    requireEdit(session);
    const { id } = request.params;
    const { duplicateId, take } = request.body;
    const money = canSeeCustomerMoney(session);
    return app.db.withUser(session, async (ctx) => {
      await refuseMerged(ctx, [duplicateId, id]);
      const names = await ctx.client.query<{ id: string; display_name: string }>(
        'select id, display_name from app.customers where tenant_id = $1 and id = any($2::uuid[])',
        [ctx.tenantId, [id, duplicateId]],
      );
      // Counted before the merge, as the person sees them: the answer never counts a lead or invoice they can't open.
      const seen = await visibleCounts(ctx, duplicateId, money);
      const { rows } = await ctx.client.query<{ result: MergeResult }>('select app.merge_customers($1, $2, $3::text[]) as result', [id, duplicateId, take]);
      const { plan: _plan, ...all } = rows[0]!.result;
      const moved: MergeMoved = { ...all, leads: seen.leads, invoices: seen.invoices ?? 0, messages: seen.messages, tickets: seen.tickets };
```

#### Step 2 — Cross the missing state or authority check

Customer merge moves linked leads without checking whether the acting editor may access each lead.

Source: `packages/db/migrations/0105_merging_customers_and_import_permission.sql:109-143` (1d5d98db9acf).

```sql
  -- The verified caller's own keys (0100). Never app.is_admin_session() in a definer function.
  if tenant is null or me is null or not app.is_member()
     or not (app.can('leads.create') or app.can('leads.route') or app.can('estimates.create')) then
    raise exception 'You don''t have permission to merge customers.' using errcode = '42501';
  end if;
  if p_keep is null or p_duplicate is null then
    raise exception 'Pick the two customers to merge.';
  end if;
  if p_keep = p_duplicate then
    raise exception 'That is the same customer: pick the duplicate to merge into it.';
  end if;
  if exists (select 1 from unnest(take) f where f is null or f not in ('name', 'type', 'email', 'phone', 'billing_address', 'notes', 'notes_both')) then
    raise exception 'Only the name, type, email, phone, billing address and notes can be taken from the duplicate.';
  end if;

  -- The customer import's lock (its run and its undo take it too): a merge never lands in the middle of one.
  perform pg_advisory_xact_lock(hashtext('app.customer_import:' || tenant::text));
  perform 1 from app.customers c where c.tenant_id = tenant and c.id in (p_keep, p_duplicate) order by c.id for update;
  select * into k from app.customers c where c.tenant_id = tenant and c.id = p_keep;
  select * into d from app.customers c where c.tenant_id = tenant and c.id = p_duplicate;
  if k.id is null or d.id is null then
    raise exception 'Both customers must be in this company. We couldn''t find one of them here.';
  end if;
  if k.merged_into is not null then
    raise exception '% was already merged into %.', k.display_name, (select m.display_name from app.customers m where m.tenant_id = tenant and m.id = k.merged_into);
  end if;
  if d.merged_into is not null then
    raise exception '% was already merged into %.', d.display_name, (select m.display_name from app.customers m where m.tenant_id = tenant and m.id = d.merged_into);
  end if;
  if k.deleted_at is not null or d.deleted_at is not null then
    raise exception '% was deleted, so it can''t be merged.', case when k.deleted_at is not null then k.display_name else d.display_name end;
  end if;
  -- Nobody moves money they may not see (the Intake VA's side-by-side showed the invoices as "Hidden").
  if not app.can('invoices.view') and exists (select 1 from app.invoices x where x.tenant_id = tenant and x.customer_id = d.id) then
    raise exception '% has invoices, and you don''t have access to invoices. Someone who handles invoices can merge these two.', d.display_name;
```


#### Step 3 — Observe the durable effect

A stepped-up customer editor was denied a linked lead, then merged its customer into another record. The hidden lead moved and its authorized owner's view changed, while the merge response reported zero hidden leads moved; missing step-up and foreign-company controls were denied.

Downstream finance or provider effects and deployed behavior remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/customer-merge/run-customer-merge.mjs:197-213` (synthetic test).

```javascript
  assert.equal(hiddenPreview.body.duplicate.counts.leads, 0);
  assert.equal(hiddenPreview.body.hidden, true);
  assert.equal(hiddenPreview.body.refused, null);
  stable(await call('hidden-merge-needs-stepup', 'POST', `/api/customers/${hiddenKeep}/merge`, nina,
    { duplicateId: hiddenDup }), 403);
  await stepUp(app, nina.headers, TEST_DEMO_PASSWORD);
  const hiddenMerge = await call('hidden-lead-merge-by-estimator', 'POST', `/api/customers/${hiddenKeep}/merge`, nina,
    { duplicateId: hiddenDup });
  assert.equal(hiddenMerge.status, 200, JSON.stringify(hiddenMerge.body));
  assert.equal(hiddenMerge.body.moved.leads, 0, 'response masks hidden lead count');
  assert.equal(sql(`select customer_id from app.leads where id = '${hiddenLead}'`), hiddenKeep, 'hidden lead actually moved');
  assert.equal(sql(`select merged_into from app.customers where id = '${hiddenDup}'`), hiddenKeep);
  const andreAfter = await call('hidden-lead-owner-after', 'GET', `/api/leads/${hiddenLead}`, andre);
  stable(andreAfter, 200);
  assert.equal(andreAfter.body.lead.customerId, hiddenKeep);
  assert.equal(andreAfter.body.lead.name, 'Audit Hidden Keep');
  stable(await call('estimator-hidden-lead-still-denied-after', 'GET', `/api/leads/${hiddenLead}`, nina), 404);
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A stepped-up customer editor merges customers while a linked same-company lead is outside the editor's lead scope.

Recorded result: A stepped-up customer editor was denied a linked lead, then merged its customer into another record. The hidden lead moved and its authorized owner's view changed, while the merge response reported zero hidden leads moved; missing step-up and foreign-company controls were denied.

Scope: Downstream finance or provider effects and deployed behavior remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/customer-merge/run-customer-merge.mjs:197-213` (synthetic test).

```javascript
  assert.equal(hiddenPreview.body.duplicate.counts.leads, 0);
  assert.equal(hiddenPreview.body.hidden, true);
  assert.equal(hiddenPreview.body.refused, null);
  stable(await call('hidden-merge-needs-stepup', 'POST', `/api/customers/${hiddenKeep}/merge`, nina,
    { duplicateId: hiddenDup }), 403);
  await stepUp(app, nina.headers, TEST_DEMO_PASSWORD);
  const hiddenMerge = await call('hidden-lead-merge-by-estimator', 'POST', `/api/customers/${hiddenKeep}/merge`, nina,
    { duplicateId: hiddenDup });
  assert.equal(hiddenMerge.status, 200, JSON.stringify(hiddenMerge.body));
  assert.equal(hiddenMerge.body.moved.leads, 0, 'response masks hidden lead count');
  assert.equal(sql(`select customer_id from app.leads where id = '${hiddenLead}'`), hiddenKeep, 'hidden lead actually moved');
  assert.equal(sql(`select merged_into from app.customers where id = '${hiddenDup}'`), hiddenKeep);
  const andreAfter = await call('hidden-lead-owner-after', 'GET', `/api/leads/${hiddenLead}`, andre);
  stable(andreAfter, 200);
  assert.equal(andreAfter.body.lead.customerId, hiddenKeep);
  assert.equal(andreAfter.body.lead.name, 'Audit Hidden Keep');
  stable(await call('estimator-hidden-lead-still-denied-after', 'GET', `/api/leads/${hiddenLead}`, nina), 404);
```

## Remediation

Reject the entire merge unless the editor can access every lead that would move, with the check enforced in the database merge function.
