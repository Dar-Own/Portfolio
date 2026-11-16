# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low (same-tenant financial-workflow integrity)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Bill-blind sub-job manager can unlink an existing bill

## Description

A sub-job manager denied Bill detail can clear an existing bill association and change the board's link state.

Prerequisite and affected workflow: A sub-job manager who cannot access Bills clears the bill ID on a known sub-job item.

Established observation: A sub-job manager was denied direct access to a linked Bill, then submitted a null bill association through the sub-job route. The request succeeded and cleared the existing link; wrong and foreign bill links were denied.

Limits: A newly blocked transition, payment delay, and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A sub-job manager who cannot access Bills clears the bill ID on a known sub-job item.

Source: `apps/api/src/features/subjobs/routes.ts:185-193` (1d5d98db9acf).

```typescript
    // Link the subcontractor's bill (entered on Bills & payroll), or unlink it with null.
    app.post('/:id/bill', { schema: { params: idParams, body: billBody } }, async (request) => {
      requirePermission(request.session, 'subjobs.manage');
      const { id } = request.params;
      const { billId } = request.body;
      return app.db.withUser(request.session, async (ctx) => {
        await linkSubJobBill(ctx, id, billId);
        return { subJob: await requireSubJob(ctx, id, await viewFor(ctx)) };
      });
```

#### Step 2 — Cross the missing state or authority check

Clearing a bill link bypasses the bill permission check used when adding or changing a link.

Source: `packages/db/migrations/0038_sub_jobs.sql:354-363` (1d5d98db9acf).

```sql
create policy tenant_isolation on app.sub_jobs
  using (tenant_id = app.current_tenant()) with check (tenant_id = app.current_tenant());
create policy member_only on app.sub_jobs as restrictive to app_user
  using ((select app.is_member())) with check ((select app.is_member()));
create policy perm_select on app.sub_jobs as restrictive for select to app_user
  using ((select app.can('subjobs.view') or app.can('subjobs.manage')));
create policy perm_insert on app.sub_jobs as restrictive for insert to app_user
  with check ((select app.can('subjobs.manage')));
create policy perm_update on app.sub_jobs as restrictive for update to app_user
  using ((select app.can('subjobs.manage'))) with check ((select app.can('subjobs.manage')));
```


#### Step 3 — Observe the durable effect

A sub-job manager was denied direct access to a linked Bill, then submitted a null bill association through the sub-job route. The request succeeded and cleared the existing link; wrong and foreign bill links were denied.

A newly blocked transition, payment delay, and deployed effect remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/complementary/cases/subjob-boundaries.mjs:106-130` (test source).

```javascript
    const board=await app.inject({method:'GET',url:'/api/sub-jobs',headers:actor.headers});
    const learned=read(board)?.items?.find((item)=>item.id===ids.sub204)?.id;
    ctx.record({label:'board-subjob-id-provenance',status:board.statusCode,learned,
      billPermission:perms.bills,viewPermission:actor.session.permissions['subjobs.view']});
    if(board.statusCode!==200||learned!==ids.sub204)return{verdict:'BLOCKED',summary:'Actor did not learn linked subjob ID from authorized board.'};
    const direct=await call('direct-bill-denial',actor,'GET',`/api/bills/${ids.bill204}`);
    const wrong=await call('wrong-vendor-bill',actor,'POST',`/api/sub-jobs/${learned}/bill`,{billId:ids.wrongBill});
    const foreignBill=ids.foreignBill?await call('foreign-bill-id',actor,'POST',`/api/sub-jobs/${learned}/bill`,{billId:ids.foreignBill}):null;
    const unlink=await call('unlink-existing-bill',actor,'POST',`/api/sub-jobs/${learned}/bill`,{billId:null});
    const relink=await call('relink-known-hidden-bill',actor,'POST',`/api/sub-jobs/${learned}/bill`,{billId:ids.bill204});
    const readback=await call('owner-subjob-readback',owner,'GET',`/api/sub-jobs/${learned}`);
    const denied=wrong.r.statusCode>=400&&JSON.stringify(wrong.before)===JSON.stringify(wrong.after)
      &&(!foreignBill||foreignBill.r.statusCode>=400&&JSON.stringify(foreignBill.before)===JSON.stringify(foreignBill.after));
    if(!([403,404].includes(direct.r.statusCode)&&denied&&unlink.r.statusCode===200
      &&unlink.before.sub204?.bill_id===ids.bill204&&unlink.after.sub204?.bill_id===null&&readback.r.statusCode===200))
      return{verdict:'BLOCKED',summary:'Bill direct-door, wrong/foreign, unlink or readback control failed.'};
    const linked=relink.r.statusCode===200&&relink.after.sub204?.bill_id===ids.bill204;
    return{verdict:'TECHNICAL-BEHAVIOR-PROVED',
      summary:linked?'A bill-blind subjobs.manage member unlinked and relinked a matching bill despite denied direct Bill access.'
        :'A bill-blind subjobs.manage member unlinked an existing bill despite denied direct Bill access; relink was denied.',
      evidence:{directBillStatus:direct.r.statusCode,unlinkStatus:unlink.r.statusCode,relinkStatus:relink.r.statusCode,
        unlinkedBillId:unlink.after.sub204?.bill_id,finalBillId:relink.after.sub204?.bill_id},
      limitations:['The selected actor had a legitimate subjobs.manage grant. No payment change or foreign-tenant link was proved.']};
  }finally{await app.close()}
}
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A sub-job manager who cannot access Bills clears the bill ID on a known sub-job item.

Recorded result: A sub-job manager was denied direct access to a linked Bill, then submitted a null bill association through the sub-job route. The request succeeded and cleared the existing link; wrong and foreign bill links were denied.

Scope: A newly blocked transition, payment delay, and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/complementary/cases/subjob-boundaries.mjs:106-130` (test source).

```javascript
    const board=await app.inject({method:'GET',url:'/api/sub-jobs',headers:actor.headers});
    const learned=read(board)?.items?.find((item)=>item.id===ids.sub204)?.id;
    ctx.record({label:'board-subjob-id-provenance',status:board.statusCode,learned,
      billPermission:perms.bills,viewPermission:actor.session.permissions['subjobs.view']});
    if(board.statusCode!==200||learned!==ids.sub204)return{verdict:'BLOCKED',summary:'Actor did not learn linked subjob ID from authorized board.'};
    const direct=await call('direct-bill-denial',actor,'GET',`/api/bills/${ids.bill204}`);
    const wrong=await call('wrong-vendor-bill',actor,'POST',`/api/sub-jobs/${learned}/bill`,{billId:ids.wrongBill});
    const foreignBill=ids.foreignBill?await call('foreign-bill-id',actor,'POST',`/api/sub-jobs/${learned}/bill`,{billId:ids.foreignBill}):null;
    const unlink=await call('unlink-existing-bill',actor,'POST',`/api/sub-jobs/${learned}/bill`,{billId:null});
    const relink=await call('relink-known-hidden-bill',actor,'POST',`/api/sub-jobs/${learned}/bill`,{billId:ids.bill204});
    const readback=await call('owner-subjob-readback',owner,'GET',`/api/sub-jobs/${learned}`);
    const denied=wrong.r.statusCode>=400&&JSON.stringify(wrong.before)===JSON.stringify(wrong.after)
      &&(!foreignBill||foreignBill.r.statusCode>=400&&JSON.stringify(foreignBill.before)===JSON.stringify(foreignBill.after));
    if(!([403,404].includes(direct.r.statusCode)&&denied&&unlink.r.statusCode===200
      &&unlink.before.sub204?.bill_id===ids.bill204&&unlink.after.sub204?.bill_id===null&&readback.r.statusCode===200))
      return{verdict:'BLOCKED',summary:'Bill direct-door, wrong/foreign, unlink or readback control failed.'};
    const linked=relink.r.statusCode===200&&relink.after.sub204?.bill_id===ids.bill204;
    return{verdict:'TECHNICAL-BEHAVIOR-PROVED',
      summary:linked?'A bill-blind subjobs.manage member unlinked and relinked a matching bill despite denied direct Bill access.'
        :'A bill-blind subjobs.manage member unlinked an existing bill despite denied direct Bill access; relink was denied.',
      evidence:{directBillStatus:direct.r.statusCode,unlinkStatus:unlink.r.statusCode,relinkStatus:relink.r.statusCode,
        unlinkedBillId:unlink.after.sub204?.bill_id,finalBillId:relink.after.sub204?.bill_id},
      limitations:['The selected actor had a legitimate subjobs.manage grant. No payment change or foreign-tenant link was proved.']};
  }finally{await app.close()}
}
```

## Remediation

Require Bills permission when adding, changing, or clearing a bill link in both the API and database.
