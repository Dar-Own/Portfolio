# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy
Evidence: Source inspection and the documented assessment outcome; no original executable proof is included.
Remediation: Resolved in reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Task-owner visibility guard conflicts with routed recipients

## Description

Selecting a routed user who cannot open a linked record can reject an automation task and interrupt subsequent work in the affected run.

Prerequisite: Automation selects a configured recipient whose record visibility does not match the task’s attached lead or job.

Recorded result: The final review exercised migrated-database/API and worker assertions, including invoice and payment-related work. The current correction passes the linked native assertions within their documented bounds.

Evidence limits: The historical vulnerable revision 5447910e84e7 is unavailable. The excerpts show the database guard and corrected recipient selection in revision be12d54. The original failure is documented in the assessment; these corrected excerpts do not reproduce it.

### Mechanism step by step

#### Step 1 — Automation selects a recipient for a linked task

Automation selects a configured recipient whose record visibility does not match the task’s attached lead or job.

Source: `packages/db/migrations/0117_read_rules_on_derived_surfaces.sql:231-250` (be12d54).

```sql
-- The same rule as a wall, for every task the Automation member writes (the worker's jobs and the event
-- handlers): its owner must be someone who may open the record it is on, the job when it has one (the
-- schedule's rule, which admits the job's lead too), else the lead. A job that picked the wrong person
-- is refused here instead of naming the record to them; findTaskOwner, told the subject, picks the next
-- person on its list before it comes to that.
create function app.automation_task_owner_may_open() returns trigger
language plpgsql as $$
begin
  if not app.is_admin_session() and app.is_automation_member()
     and not (case when new.job_id is not null then app.automation_member_may_open(new.owner_id, 'job', new.job_id)
                   when new.lead_id is not null then app.automation_member_may_open(new.owner_id, 'lead', new.lead_id)
                   else true end) then
    raise exception 'This task''s owner may not open the record it is about: set the routing rules on the Automations page to someone who may'
      using errcode = '42501';
  end if;
  return new;
end $$;
revoke all on function app.automation_task_owner_may_open() from public;
create trigger tasks_automation_owner_may_open before insert or update of owner_id, lead_id, job_id on app.tasks
for each row execute function app.automation_task_owner_may_open();
```

#### Step 2 — The final visibility guard refuses an ineligible owner

New task-owner confidentiality checks and existing producer selection applied different eligibility rules.

Source: `apps/worker/src/jobs/invoice-reminders.ts:326-345` (be12d54).

```typescript
/**
 * One task for the office when there is nobody to email. Safe to run again: the dedupe key stops copies. It is
 * written on the invoice's job and lead and names the customer, so it goes to the office person when they may
 * open that job (else its lead), else the company's owner (0125, the round-2 review's N-02: 0117's wall refused
 * it for an office person blind to the job, and the refusal stopped the company's run). With no office rule at
 * all, the owner too (the round-2 attack: the task was never raised). Written only while the invoice is still owed
 * and still has nobody to email, read in the writing statement (rule 7): an address added since the scan stops it.
 */
async function raiseNoEmailTask(client: Client, tenantId: string, invoice: Candidate, settings: CompanySettings, now: Date): Promise<boolean> {
  const owner = await findTaskOwner(client, tenantId, [settings.officeUserId], { leadId: invoice.lead_id, jobId: invoice.job_id, orTheOwner: true });
  if (!owner) throw new Error('no one who may open the invoice\'s job to ask for an email address');
  return createTaskIf(client, {
    tenantId,
    title: `Invoice ${invoice.number}: add an email address for ${invoice.customer_name} so reminders can go out`,
    kind: 'manual',
    ownerId: owner.id,
    leadId: invoice.lead_id,
    jobId: invoice.job_id,
    dueAt: new Date(now.getTime() + 24 * 60 * 60 * 1000),
    dedupeKey: `invoice-reminder-no-email:${invoice.invoice_id}`,
```

#### Step 3 — Observe the effect and its correction

The final review exercised migrated-database/API and worker assertions, including invoice and payment-related work. The current correction passes the linked native assertions within their documented bounds.

Retain the confidentiality guard and select an eligible recipient or handle refusal without losing subsequent work. The final reviewed implementation passes the relevant routing controls.

The historical result is documented in the assessment. This corrected source is not an executable demonstration of the original failure:

Source: `packages/db/migrations/0117_read_rules_on_derived_surfaces.sql:231-250` (be12d54).

```sql
-- The same rule as a wall, for every task the Automation member writes (the worker's jobs and the event
-- handlers): its owner must be someone who may open the record it is on, the job when it has one (the
-- schedule's rule, which admits the job's lead too), else the lead. A job that picked the wrong person
-- is refused here instead of naming the record to them; findTaskOwner, told the subject, picks the next
-- person on its list before it comes to that.
create function app.automation_task_owner_may_open() returns trigger
language plpgsql as $$
begin
  if not app.is_admin_session() and app.is_automation_member()
     and not (case when new.job_id is not null then app.automation_member_may_open(new.owner_id, 'job', new.job_id)
                   when new.lead_id is not null then app.automation_member_may_open(new.owner_id, 'lead', new.lead_id)
                   else true end) then
    raise exception 'This task''s owner may not open the record it is about: set the routing rules on the Automations page to someone who may'
      using errcode = '42501';
  end if;
  return new;
end $$;
revoke all on function app.automation_task_owner_may_open() from public;
create trigger tasks_automation_owner_may_open before insert or update of owner_id, lead_id, job_id on app.tasks
for each row execute function app.automation_task_owner_may_open();
```

## Proof of concept

No original executable proof is available in this case study. The assessment documents the result, while the included source shows the corrected behavior.

Setup: Automation selects a configured recipient whose record visibility does not match the task’s attached lead or job.

Recorded result: The final review exercised migrated-database/API and worker assertions, including invoice and payment-related work. The current correction passes the linked native assertions within their documented bounds.

Scope: The historical vulnerable revision 5447910e84e7 is unavailable. The excerpts show the database guard and corrected recipient selection in revision be12d54. The original failure is documented in the assessment; these corrected excerpts do not reproduce it.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

No original executable test excerpt is available for this case. The historical failure is documented in the assessment; the source excerpts above show the guard and the corrected recipient-selection logic. They do not independently reproduce the original failure.
