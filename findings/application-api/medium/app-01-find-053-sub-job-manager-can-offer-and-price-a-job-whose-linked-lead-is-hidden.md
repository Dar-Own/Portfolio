# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Medium (same-tenant process/pricing integrity)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Acknowledged, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Sub-job manager can offer and price a job whose linked lead is hidden

## Description

A sub-job manager can create and change priced offers on a job whose linked lead they cannot open.

Prerequisite and affected workflow: A member with sub-job management grants learns the job ID from the allowed picker and submits offer or price changes despite linked-lead denial.

Established observation: A sub-job manager could not open the linked lead, but learned its job ID from an allowed picker and created a priced subcontract offer for it. The offer and activity record were saved; unrelated-vendor controls were denied.

Limits: Supplier delivery, payment reliance, split-grant prevalence, and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A member with sub-job management grants learns the job ID from the allowed picker and submits offer or price changes despite linked-lead denial.

Source: `apps/api/src/features/subjobs/routes.ts:132-149` (1d5d98db9acf).

```typescript
    // Offer a job to a subcontractor. The answer carries a warning when the sub's COI is not in order.
    app.post('/', { schema: { body: createBody } }, async (request, reply) => {
      requirePermission(request.session, 'subjobs.manage');
      const body = request.body;
      const result = await app.db.withUser(request.session, async (ctx) => {
        const id = await createSubJob(ctx, {
          jobId: body.jobId,
          vendorId: body.vendorId,
          subPriceCents: body.subPriceCents,
          customerPriceCents: body.customerPriceCents ?? null,
          notes: body.notes ?? null,
        });
        const subJob = await requireSubJob(ctx, id, await viewFor(ctx));
        await noteActivity(ctx, `${subJob.job.number} offered to ${subJob.vendor.companyName} (sub job)`, subJob.job.leadId);
        return { subJob, warning: subJob.vendor.coi.warning };
      });
      return reply.status(201).send(result);
    });
```

#### Step 2 — Cross the missing state or authority check

Sub-job writes check general feature permission and job status but do not enforce access to the linked lead.

Source: `packages/db/migrations/0038_sub_jobs.sql:170-229` (1d5d98db9acf).

```sql
  -- Plain refusals first. Row-level security checks a new row only after this trigger has run, so
  -- without these a person with no rights would hear about the vendor or the estimate they can't see.
  if tg_op = 'INSERT' and not admin then
    if new.tenant_id is distinct from app.current_tenant() then
      raise exception 'That job belongs to another company';
    end if;
    if not app.can('subjobs.manage') then
      raise exception 'You don''t have permission to subcontract jobs and manage sub jobs';
    end if;
  end if;

  -- The subcontractor: an active vendor of kind subcontractor, seen under this person's own rights.
  if tg_op = 'INSERT' or new.vendor_id is distinct from old.vendor_id then
    select v.kind, v.is_active, v.company_name into vendor
      from app.vendors v where v.tenant_id = new.tenant_id and v.id = new.vendor_id;
    if not found then
      raise exception 'Pick a subcontractor from Vendors & subs';
    end if;
    if vendor.kind <> 'subcontractor' then
      raise exception '% is a vendor, not a subcontractor. Change that on Vendors & subs first', vendor.company_name;
    end if;
    if not vendor.is_active then
      raise exception '% is not active any more. Make it active again on Vendors & subs first', vendor.company_name;
    end if;
    if tg_op = 'UPDATE' and not admin and old.stage <> 'offered' then
      raise exception 'The subcontractor can only change while the sub job is Offered';
    end if;
  end if;

  if tg_op = 'INSERT' then
    if not admin then
      new.created_by := app.current_user_id();
      new.stage := 'offered';
      new.offered_at := now();
      new.stage_entered_at := now();
      new.accepted_at := null;
      new.started_at := null;
      new.completed_at := null;
      new.paid_at := null;
      select j.status::text into job_status from app.jobs j where j.tenant_id = new.tenant_id and j.id = new.job_id;
      if job_status is null then
        raise exception 'We couldn''t find that job';
      end if;
      if job_status in ('completed', 'cancelled') then
        raise exception 'Only a job that is still open can be subcontracted';
      end if;
    else
      new.offered_at := coalesce(new.offered_at, new.created_at, now());
      new.stage_entered_at := coalesce(new.stage_entered_at, new.offered_at);
    end if;
    -- The customer price starts from the job's estimate total when nobody typed one.
    if new.customer_price_cents is null then
      select e.total_cents into new.customer_price_cents
        from app.jobs j
        join app.estimates e on e.tenant_id = j.tenant_id and e.id = j.estimate_id
       where j.tenant_id = new.tenant_id and j.id = new.job_id;
      if new.customer_price_cents is null then
        raise exception 'Enter the customer price: this job has no estimate total to start from';
      end if;
    end if;
```


#### Step 3 — Observe the durable effect

A sub-job manager could not open the linked lead, but learned its job ID from an allowed picker and created a priced subcontract offer for it. The offer and activity record were saved; unrelated-vendor controls were denied.

Supplier delivery, payment reliance, split-grant prevalence, and deployed effect remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/complementary/cases/subjob-boundaries.mjs:84-104` (test source).

```javascript
      const linkedLead=ctx.sql(`select lead_id from app.jobs where id=${q(ids.j203)}`).trim();
      const lead=await app.inject({method:'GET',url:`/api/leads/${linkedLead}`,headers:actor.headers});
      const picker=await app.inject({method:'GET',url:'/api/sub-jobs/jobs',headers:actor.headers});
      const pickerJob=read(picker)?.items?.find((item)=>item.id===ids.j203);
      ctx.record({label:'job-lead-and-picker',leadStatus:lead.statusCode,pickerStatus:picker.statusCode,
        pickerHasJ203:Boolean(pickerJob),perms});
      if(!pickerJob)return{verdict:'BLOCKED',summary:'Manager could not learn the hidden job ID from the real picker.'};
      const base={jobId:pickerJob.id,vendorId:ids.rapid,subPriceCents:240000,customerPriceCents:310000};
      const supplier=await call('supplier-not-subcontractor',actor,'POST','/api/sub-jobs',{...base,vendorId:ids.supplier});
      const foreignVendor=ids.foreignVendor?await call('foreign-vendor-id',actor,'POST','/api/sub-jobs',{...base,vendorId:ids.foreignVendor}):null;
      const valid=await call('valid-known-job-offer',actor,'POST','/api/sub-jobs',base);
      if(!(supplier.r.statusCode>=400&&JSON.stringify(supplier.before)===JSON.stringify(supplier.after)
        &&(!foreignVendor||foreignVendor.r.statusCode>=400&&JSON.stringify(foreignVendor.before)===JSON.stringify(foreignVendor.after))))
        return{verdict:'TECHNICAL-BEHAVIOR-PROVED',summary:'Supplier/foreign vendor denial or no-effect control failed.'};
      const hidden=[403,404].includes(lead.statusCode);
      const offered=valid.r.statusCode===201&&valid.after.offers203===valid.before.offers203+1;
      return{verdict:hidden&&offered?'TECHNICAL-BEHAVIOR-PROVED-POLICY-CONDITIONAL':offered?'FALSIFIED-IN-SCOPE':'BLOCKED',
        summary:hidden&&offered?'Subjobs.manage offered a job whose linked lead this member cannot open; companywide manage intent is pending.'
          :offered?'Selected offer used a lead this actor could open; foreign/supplier vendor substitutions were denied.':'Valid offer positive failed.',
        evidence:{leadStatus:lead.statusCode,pickerStatus:picker.statusCode,offered,status:valid.r.statusCode},
        limitations:['Subjobs.manage may intentionally apply companywide; no cross-tenant offer was shown.']};
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A member with sub-job management grants learns the job ID from the allowed picker and submits offer or price changes despite linked-lead denial.

Recorded result: A sub-job manager could not open the linked lead, but learned its job ID from an allowed picker and created a priced subcontract offer for it. The offer and activity record were saved; unrelated-vendor controls were denied.

Scope: Supplier delivery, payment reliance, split-grant prevalence, and deployed effect remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/complementary/cases/subjob-boundaries.mjs:84-104` (test source).

```javascript
      const linkedLead=ctx.sql(`select lead_id from app.jobs where id=${q(ids.j203)}`).trim();
      const lead=await app.inject({method:'GET',url:`/api/leads/${linkedLead}`,headers:actor.headers});
      const picker=await app.inject({method:'GET',url:'/api/sub-jobs/jobs',headers:actor.headers});
      const pickerJob=read(picker)?.items?.find((item)=>item.id===ids.j203);
      ctx.record({label:'job-lead-and-picker',leadStatus:lead.statusCode,pickerStatus:picker.statusCode,
        pickerHasJ203:Boolean(pickerJob),perms});
      if(!pickerJob)return{verdict:'BLOCKED',summary:'Manager could not learn the hidden job ID from the real picker.'};
      const base={jobId:pickerJob.id,vendorId:ids.rapid,subPriceCents:240000,customerPriceCents:310000};
      const supplier=await call('supplier-not-subcontractor',actor,'POST','/api/sub-jobs',{...base,vendorId:ids.supplier});
      const foreignVendor=ids.foreignVendor?await call('foreign-vendor-id',actor,'POST','/api/sub-jobs',{...base,vendorId:ids.foreignVendor}):null;
      const valid=await call('valid-known-job-offer',actor,'POST','/api/sub-jobs',base);
      if(!(supplier.r.statusCode>=400&&JSON.stringify(supplier.before)===JSON.stringify(supplier.after)
        &&(!foreignVendor||foreignVendor.r.statusCode>=400&&JSON.stringify(foreignVendor.before)===JSON.stringify(foreignVendor.after))))
        return{verdict:'TECHNICAL-BEHAVIOR-PROVED',summary:'Supplier/foreign vendor denial or no-effect control failed.'};
      const hidden=[403,404].includes(lead.statusCode);
      const offered=valid.r.statusCode===201&&valid.after.offers203===valid.before.offers203+1;
      return{verdict:hidden&&offered?'TECHNICAL-BEHAVIOR-PROVED-POLICY-CONDITIONAL':offered?'FALSIFIED-IN-SCOPE':'BLOCKED',
        summary:hidden&&offered?'Subjobs.manage offered a job whose linked lead this member cannot open; companywide manage intent is pending.'
          :offered?'Selected offer used a lead this actor could open; foreign/supplier vendor substitutions were denied.':'Valid offer positive failed.',
        evidence:{leadStatus:lead.statusCode,pickerStatus:picker.statusCode,offered,status:valid.r.statusCode},
        limitations:['Subjobs.manage may intentionally apply companywide; no cross-tenant offer was shown.']};
```

## Remediation

Require access to the linked job and lead when offering, editing, or changing a sub-job, including in the database write checks.
