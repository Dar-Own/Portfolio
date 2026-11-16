# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy
Evidence: Source inspection and recorded regression tests for corrected behavior; these excerpts are not a reproduction of the original failure.
Remediation: Resolved in reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Logical restore suppresses history-based security alerts

## Description

After restore into a cluster with a lower transaction counter, new invitation, two-factor-reset or connection-change warnings can be omitted even though audit rows exist.

Prerequisite: A logical backup imports a scan watermark greater than the destination cluster’s new transaction IDs.

Recorded result: A full native logical restore and actual worker reproduced the suppression. Patched restore controls verify consumed versus pending audit eligibility, null fallback, repeated restore, trigger protection and unchanged audit payloads.

Evidence limits: Logical restore and worker behavior were recorded locally. The accepted database/file snapshot consistency limitation is a separate historical finding. Rebase logic modifies derived metadata only; it is not proof of a deployed restore.

### Mechanism step by step

#### Step 1 — Restore imports a cursor from a different transaction-ID space

A logical backup imports a scan watermark greater than the destination cluster’s new transaction IDs.

Source: `apps/worker/src/jobs/security-alerts.ts:220-236` (be12d54).

```typescript
      // never read). `xmin` is the oldest transaction still in progress, taken BEFORE the history is read:
      // every line of an older one is there to read now, and every line this run can't see yet was written
      // by a transaction at or after it, which the next run reads (scanned_xmin).
      const point = await client.query<{ scanned_to: Date; scanned_xmin: string | null; xmin: string }>(
        `select coalesce((select s.scanned_to from app.security_alert_scans s where s.tenant_id = $1),
                         (select t.created_at from app.tenants t where t.id = $1)) as scanned_to,
                (select s.scanned_xmin::text from app.security_alert_scans s where s.tenant_id = $1) as scanned_xmin,
                pg_snapshot_xmin(pg_current_snapshot())::text as xmin`,
        [tenantId],
      );
      const scannedTo = point.rows[0]?.scanned_to ?? new Date(c.now.getTime() - 60 * 60_000);
      const scannedXmin = point.rows[0]?.scanned_xmin ?? null;
      await client.query(
        `insert into app.security_alert_scans (tenant_id, scanned_to, scanned_xmin) values ($1, $2, $3::xid8)
         on conflict (tenant_id) do update set scanned_to = greatest(app.security_alert_scans.scanned_to, excluded.scanned_to),
                                               scanned_xmin = greatest(app.security_alert_scans.scanned_xmin, excluded.scanned_xmin)`,
        [tenantId, c.now, point.rows[0]!.xmin],
```

#### Step 2 — The worker filters new audit events using the retained watermark

Derived audit transaction IDs and scan cursors retain source-cluster values that are not comparable with destination IDs.

Source: `apps/worker/src/jobs/security-alerts.ts:339-349` (be12d54).

```typescript
      // ponytail: a transaction left open for days anywhere on the server keeps the re-read growing; dedupe holds.
      const hourAgo = new Date(c.now.getTime() - 60 * 60_000);
      const since = new Date(Math.min(scannedTo.getTime() - 60 * 60_000, hourAgo.getTime()));
      const events = scannedXmin
        ? (await client.query<EventRow>('select * from app.security_alert_events_after($1::xid8, $2)', [scannedXmin, since])).rows
        : (await client.query<EventRow>('select * from app.security_alert_events($1)', [since])).rows;

      // 2. Three or more accounts locked within an hour: the strongest such hour ending at a lockout, one
      // alert per clock hour of its end (the key). Every hour a new lockout falls in is looked at again, so
      // its ends are every lockout from the first new one to an hour after the last (the attack on round 2:
      // a new lockout dated BEFORE two read by an earlier run made a full hour ending at the later ones,
```

#### Step 3 — Observe the effect and its correction

A full native logical restore and actual worker reproduced the suppression. Patched restore controls verify consumed versus pending audit eligibility, null fallback, repeated restore, trigger protection and unchanged audit payloads.

Before startup, rebase only derived XID metadata in one administrator transaction. Preserve consumed history, pending eligibility and null fallback; fail restoration on unexpected schema or trigger state.

Regression test for the corrected behavior:

Source: `packages/db/src/restore-alert-cursors.test.ts:16-43` (corrected implementation).

```typescript
test('N-03: rebasing preserves pending IDs, consumed history, null fallback, audit content and trigger protection',()=>{
 const tenant=sql("select id from app.tenants where slug='peach-state-group'");
 const other=sql(`select id from app.tenants where id<>'${tenant}' limit 1`);
 sql(`truncate app.audit_log;delete from app.security_alert_scans;
 insert into app.security_alert_scans(tenant_id,scanned_to,scanned_xmin) values('${tenant}',now(),'900000'::xid8),('${other}',now(),null);
 insert into app.audit_log(tenant_id,action,entity,entity_id,xid,occurred_at) values
 ('${tenant}','insert','synthetic','consumed','899999'::xid8,now()),
 ('${tenant}','insert','synthetic','pending','900001'::xid8,now()-interval '2 days'),
 ('${tenant}','insert','synthetic','legacy-pending',null,now()),
 ('${tenant}','insert','synthetic','legacy-consumed',null,now()-interval '2 days'),
 ('${other}','insert','synthetic','null-cursor','900002'::xid8,now())`);
 const payload=()=>sql("select jsonb_agg(to_jsonb(a)-'xid' order by id) from app.audit_log a");
 const scans=()=>sql("select jsonb_agg(jsonb_build_array(tenant_id,scanned_to,scanned_xmin is null) order by tenant_id) from app.security_alert_scans");
 const eligible=()=>sql(`select string_agg(a.id::text,',' order by a.id) from app.audit_log a join app.security_alert_scans s using(tenant_id)
 where case when s.scanned_xmin is null then a.occurred_at>=s.scanned_to-interval '1 hour'
 else a.xid>=s.scanned_xmin or (a.xid is null and a.occurred_at>=s.scanned_to-interval '1 hour') end`);
 const before={payload:payload(),scans:scans(),eligible:eligible()};
 assert.throws(()=>sql('set role app_worker;'+readFileSync(file,'utf8').replace(/^\\set.*$/gm,'')),/requires a database administrator/);
 for(let i=0;i<2;i++){
  const run=repair();assert.equal(run.status,0,run.stderr);
  assert.deepEqual({payload:payload(),scans:scans(),eligible:eligible()},before);
  assert.equal(sql("select xid::text from app.audit_log where entity_id='consumed'"),'0');
 }
 assert.throws(()=>sql("update app.audit_log set entity_id='changed'"),/append.only|immutable|cannot|may not/i);
 const metadata=()=>sql('select jsonb_agg(jsonb_build_array(id,xid::text) order by id) from app.audit_log');
 const xids=metadata();
 sql('alter table app.audit_log disable trigger audit_no_update');
 const invalid=repair();assert.notEqual(invalid.status,0);assert.match(invalid.stderr,/Unexpected restore trigger/);assert.equal(metadata(),xids);
```

## Proof of concept

The excerpts below test the corrected behavior. They support the remediation within the tested conditions, rather than independently demonstrating the original vulnerability.

Setup: A logical backup imports a scan watermark greater than the destination cluster’s new transaction IDs.

Recorded result: A full native logical restore and actual worker reproduced the suppression. Patched restore controls verify consumed versus pending audit eligibility, null fallback, repeated restore, trigger protection and unchanged audit payloads.

Scope: Logical restore and worker behavior were recorded locally. The accepted database/file snapshot consistency limitation is a separate historical finding. Rebase logic modifies derived metadata only; it is not proof of a deployed restore.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Regression test: `packages/db/src/restore-alert-cursors.test.ts`.

Source: `packages/db/src/restore-alert-cursors.test.ts:16-43` (corrected implementation).

```typescript
test('N-03: rebasing preserves pending IDs, consumed history, null fallback, audit content and trigger protection',()=>{
 const tenant=sql("select id from app.tenants where slug='peach-state-group'");
 const other=sql(`select id from app.tenants where id<>'${tenant}' limit 1`);
 sql(`truncate app.audit_log;delete from app.security_alert_scans;
 insert into app.security_alert_scans(tenant_id,scanned_to,scanned_xmin) values('${tenant}',now(),'900000'::xid8),('${other}',now(),null);
 insert into app.audit_log(tenant_id,action,entity,entity_id,xid,occurred_at) values
 ('${tenant}','insert','synthetic','consumed','899999'::xid8,now()),
 ('${tenant}','insert','synthetic','pending','900001'::xid8,now()-interval '2 days'),
 ('${tenant}','insert','synthetic','legacy-pending',null,now()),
 ('${tenant}','insert','synthetic','legacy-consumed',null,now()-interval '2 days'),
 ('${other}','insert','synthetic','null-cursor','900002'::xid8,now())`);
 const payload=()=>sql("select jsonb_agg(to_jsonb(a)-'xid' order by id) from app.audit_log a");
 const scans=()=>sql("select jsonb_agg(jsonb_build_array(tenant_id,scanned_to,scanned_xmin is null) order by tenant_id) from app.security_alert_scans");
 const eligible=()=>sql(`select string_agg(a.id::text,',' order by a.id) from app.audit_log a join app.security_alert_scans s using(tenant_id)
 where case when s.scanned_xmin is null then a.occurred_at>=s.scanned_to-interval '1 hour'
 else a.xid>=s.scanned_xmin or (a.xid is null and a.occurred_at>=s.scanned_to-interval '1 hour') end`);
 const before={payload:payload(),scans:scans(),eligible:eligible()};
 assert.throws(()=>sql('set role app_worker;'+readFileSync(file,'utf8').replace(/^\\set.*$/gm,'')),/requires a database administrator/);
 for(let i=0;i<2;i++){
  const run=repair();assert.equal(run.status,0,run.stderr);
  assert.deepEqual({payload:payload(),scans:scans(),eligible:eligible()},before);
  assert.equal(sql("select xid::text from app.audit_log where entity_id='consumed'"),'0');
 }
 assert.throws(()=>sql("update app.audit_log set entity_id='changed'"),/append.only|immutable|cannot|may not/i);
 const metadata=()=>sql('select jsonb_agg(jsonb_build_array(id,xid::text) order by id) from app.audit_log');
 const xids=metadata();
 sql('alter table app.audit_log disable trigger audit_no_update');
 const invalid=repair();assert.notEqual(invalid.status,0);assert.match(invalid.stderr,/Unexpected restore trigger/);assert.equal(metadata(),xids);
```
