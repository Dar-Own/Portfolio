# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low (conditional subscription integrity/alert availability)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Cap-refused push claim deletes another tenant's subscription

## Description

A rejected push-subscription claim can delete another company member's endpoint. A test alert sent through the application then has no endpoint for that member until they register again.

Prerequisite and affected workflow: A member already at the five-phone cap claims a valid endpoint currently registered to the other member.

Established observation: A member at the five-device cap claimed a push endpoint already owned by another company's member. The API returned a refusal but deleted the prior owner's subscription, reducing test-alert handoff from one attempt to zero until that owner registered again.

Limits: Ordinary alert wiring, real push delivery, and persistent lockout remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

A member already at the five-phone cap claims a valid endpoint currently registered to the other member.

Source: `packages/db/migrations/0084_push_subscriptions.sql:94-110` (1d5d98db9acf).

```sql
create function app.claim_push_endpoint(p_endpoint_hash bytea) returns integer
language plpgsql security definer set search_path = app, pg_temp as $$
declare
  person uuid := app.current_user_id();
  gone integer;
begin
  if not app.is_app_login() or person is null or not app.is_member() or (select app.is_automation_member()) then
    raise exception 'Only the person signed in may claim a phone.' using errcode = 'insufficient_privilege';
  end if;
  if p_endpoint_hash is null or octet_length(p_endpoint_hash) <> 32 then
    raise exception 'An endpoint hash must be 32 bytes (SHA-256).';
  end if;
  delete from app.push_subscriptions where endpoint_hash = p_endpoint_hash and user_id <> person;
  get diagnostics gone = row_count;
  return gone;
end $$;
revoke all on function app.claim_push_endpoint(bytea) from public;
```

#### Step 2 — Cross the missing state or authority check

The endpoint claim deletes the old owner before the capped insert fails, and the API returns 409 after that deletion commits.

Source: `apps/api/src/features/push/routes.ts:70-89` (1d5d98db9acf).

```typescript
      const session = request.session;
      const ciphertext = encryptSubscription(masterKey, session.user.id, subscription);
      const hash = endpointHash(subscription.endpoint);
      const kept = await app.db.withUser(session, async ({ client, tenantId, userId }) => {
        // Whoever holds the phone gets its alerts: the same subscription under anyone else (a crew phone
        // that changed hands) is theirs no more, in any company.
        await client.query('select app.claim_push_endpoint($1)', [hash]);
        // A phone already on the list is renewed; a new one only while the person is under the cap.
        const result = await client.query(
          `insert into app.push_subscriptions (tenant_id, user_id, endpoint_hash, subscription_ciphertext, label)
           select $1, $2, $3, $4, $5
            where (select count(*) from app.push_subscriptions where user_id = $2 and endpoint_hash <> $3) < $6
           on conflict (tenant_id, user_id, endpoint_hash)
           do update set subscription_ciphertext = excluded.subscription_ciphertext, label = excluded.label`,
          [tenantId, userId, hash, ciphertext, request.body.label ?? 'This phone', MAX_PHONES],
        );
        return result.rowCount ?? 0;
      });
      if (kept === 0) throw new HttpError(409, 'too_many_phones', TOO_MANY_PHONES);
      await reply.status(204).send();
```


#### Step 3 — Observe the durable effect

A member at the five-device cap claimed a push endpoint already owned by another company's member. The API returned a refusal but deleted the prior owner's subscription, reducing test-alert handoff from one attempt to zero until that owner registered again.

Ordinary alert wiring, real push delivery, and persistent lockout remain unproved.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/fuzzing/complementary/cases/push-cap-http.mjs:77-100` (test source).

```javascript
      bAfterInvalid, aCount: (await list(a)).length });
    assert.equal(invalid.statusCode, 422);
    assert.deepEqual(bAfterInvalid, [shared.hash]);
    const removed = await app.inject({ method: 'DELETE', url: '/api/push/subscriptions', headers: a.headers,
      payload: { endpoint: aPhones[0].endpoint } });
    assert.equal(removed.statusCode, 204);
    const accepted = await put(a, shared, 'Accepted after free slot');
    const bAfterAccepted = await list(b), aAfterAccepted = await list(a);
    ctx.record({ label: 'below-cap-transfer-positive', status: accepted.status,
      bAfterAccepted, aAfterAccepted, fakePushAttempts: fakeAttempts.length });
    assert.equal(accepted.status, 204); assert.deepEqual(bAfterAccepted, []);
    assert.ok(aAfterAccepted.includes(shared.hash));
    assert.equal(ctx.providerAttempts.length, 0);
    const broken = !bAfterRefusal.includes(shared.hash);
    return { verdict: broken ? 'TEST-CONFIRMED' : 'FALSIFIED-IN-TESTED-SCOPE',
      summary: broken ? 'A real cap-refused HTTP 409 deleted another tenant member’s existing phone row; fake push to that member dropped from one accepted attempt to zero until they re-registered, which succeeded.' : 'The refused HTTP request left the prior subscription intact.',
      evidence: { refused, bBefore, bAfterRefusal, aBefore, aAfterRefusal, rowsBefore,
        rowsAfterRefusal, testBefore, testAfterRefusal, restore, bRestored, testRestored,
        invalidStatus: invalid.statusCode, accepted, bAfterAccepted, aAfterAccepted,
        fakePushAttempts: fakeAttempts,
        source: ['apps/api/src/features/push/routes.ts:70-89', 'packages/db/migrations/0084_push_subscriptions.sql:94-110'] },
      limitations: ['Shared endpoint possession is required; an arbitrary attacker cannot guess another phone endpoint.', 'The previous member can restore by re-registering; no durable lockout proved.', 'Ordinary production push events and real push-service delivery were not tested. The client separately wants alerts to stop when another user actually signs in on a shared device.'] };
  } finally { setPushFetchForTests(null); await app.close(); }
}
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: A member already at the five-phone cap claims a valid endpoint currently registered to the other member.

Recorded result: A member at the five-device cap claimed a push endpoint already owned by another company's member. The API returned a refusal but deleted the prior owner's subscription, reducing test-alert handoff from one attempt to zero until that owner registered again.

Scope: Ordinary alert wiring, real push delivery, and persistent lockout remain unproved. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/fuzzing/complementary/cases/push-cap-http.mjs:77-100` (test source).

```javascript
      bAfterInvalid, aCount: (await list(a)).length });
    assert.equal(invalid.statusCode, 422);
    assert.deepEqual(bAfterInvalid, [shared.hash]);
    const removed = await app.inject({ method: 'DELETE', url: '/api/push/subscriptions', headers: a.headers,
      payload: { endpoint: aPhones[0].endpoint } });
    assert.equal(removed.statusCode, 204);
    const accepted = await put(a, shared, 'Accepted after free slot');
    const bAfterAccepted = await list(b), aAfterAccepted = await list(a);
    ctx.record({ label: 'below-cap-transfer-positive', status: accepted.status,
      bAfterAccepted, aAfterAccepted, fakePushAttempts: fakeAttempts.length });
    assert.equal(accepted.status, 204); assert.deepEqual(bAfterAccepted, []);
    assert.ok(aAfterAccepted.includes(shared.hash));
    assert.equal(ctx.providerAttempts.length, 0);
    const broken = !bAfterRefusal.includes(shared.hash);
    return { verdict: broken ? 'TEST-CONFIRMED' : 'FALSIFIED-IN-TESTED-SCOPE',
      summary: broken ? 'A real cap-refused HTTP 409 deleted another tenant member’s existing phone row; fake push to that member dropped from one accepted attempt to zero until they re-registered, which succeeded.' : 'The refused HTTP request left the prior subscription intact.',
      evidence: { refused, bBefore, bAfterRefusal, aBefore, aAfterRefusal, rowsBefore,
        rowsAfterRefusal, testBefore, testAfterRefusal, restore, bRestored, testRestored,
        invalidStatus: invalid.statusCode, accepted, bAfterAccepted, aAfterAccepted,
        fakePushAttempts: fakeAttempts,
        source: ['apps/api/src/features/push/routes.ts:70-89', 'packages/db/migrations/0084_push_subscriptions.sql:94-110'] },
      limitations: ['Shared endpoint possession is required; an arbitrary attacker cannot guess another phone endpoint.', 'The previous member can restore by re-registering; no durable lockout proved.', 'Ordinary production push events and real push-service delivery were not tested. The client separately wants alerts to stop when another user actually signs in on a shared device.'] };
  } finally { setPushFetchForTests(null); await app.close(); }
}
```

## Remediation

Validate the cap before transferring endpoint ownership or roll back the transfer and insert together on cap failure.
