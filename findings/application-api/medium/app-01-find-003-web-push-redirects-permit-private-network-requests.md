# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Medium
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and recorded local tests of the original behavior; case-specific execution and impact limits are stated below.
Remediation: Resolved, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# Web Push redirects permit private-network requests

## Description

A signed-in member can redirect the server's Web Push POST toward a private or link-local address, creating a blind internal request primitive.

Prerequisite and affected workflow: The member registers a public-looking push endpoint that redirects to an address reachable from the application network.

Established observation: A signed-in member registered a push endpoint whose public-looking address redirected the application's POST to a private-network destination. The test established the server-side request primitive, not access to a particular AWS service.

Limits: Reachability of a protected AWS or VPC service was not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

The member registers a public-looking push endpoint that redirects to an address reachable from the application network.

Source: `packages/integrations/src/web-push.ts:94-138` (1d5d98db9acf).

```typescript
export const MAX_ENDPOINT_LENGTH = 2048;

/**
 * A push service lives on the public internet. The server posts to whatever endpoint a browser hands
 * it, so an address on this machine, its own network or a link-local range is refused: nobody signed
 * in may use the server to knock on ports behind its firewall. (A name that resolves to such an
 * address is not caught here; that is for the proxy's outbound rules.)
 */
export function isPrivateHost(hostname: string): boolean {
  const host = hostname.replace(/^\[|\]$/g, '').toLowerCase();
  if (host === 'localhost' || host.endsWith('.localhost')) return true;
  const kind = isIP(host);
  if (kind === 6) {
    if (host === '::' || host === '::1' || /^f[cd]/.test(host) || /^fe[89ab]/.test(host)) return true;
    // An IPv4 address written as IPv6 (::ffff:7f00:1 is 127.0.0.1).
    const mapped = /^::ffff:([0-9a-f]{1,4}):[0-9a-f]{1,4}$/.exec(host);
    if (!mapped) return false;
    const high = parseInt(mapped[1]!, 16);
    return isPrivateHost(`${high >> 8}.${high & 255}.0.0`);
  }
  if (kind === 4) {
    const [a, b] = host.split('.').map(Number) as [number, number];
    return a === 0 || a === 10 || a === 127 || (a === 100 && b >= 64 && b <= 127) || (a === 169 && b === 254) || (a === 172 && b >= 16 && b <= 31) || (a === 192 && b === 168);
  }
  return false;
}

/** The subscription when it has the right shape (a public https endpoint, a P-256 point, a 16-byte secret), else null. */
export function parseSubscription(value: unknown): PushSubscription | null {
  if (typeof value !== 'object' || value === null) return null;
  const { endpoint, keys } = value as { endpoint?: unknown; keys?: unknown };
  if (typeof endpoint !== 'string' || endpoint.length > MAX_ENDPOINT_LENGTH) return null;
  try {
    const url = new URL(endpoint);
    if (url.protocol !== 'https:' || isPrivateHost(url.hostname)) return null;
  } catch {
    return null;
  }
  if (typeof keys !== 'object' || keys === null) return null;
  const { p256dh, auth } = keys as { p256dh?: unknown; auth?: unknown };
  if (typeof p256dh !== 'string' || typeof auth !== 'string') return null;
  const point = fromBase64url(p256dh);
  if (point.length !== 65 || point[0] !== 0x04 || fromBase64url(auth).length !== 16) return null;
  return { endpoint, keys: { p256dh, auth } };
}
```

#### Step 2 — Cross the missing state or authority check

The subscription parser validates only the initial URL and rejects literal private hosts: `packages/integrations/src/web-push.ts:94-138`. `sendWebPush` then uses native `fetch` with automatic redirects and performs no destination revalidation: `packages/integrations/src/web-push.ts:262-283`. The test endpoint is available to any signed-in member: `apps/api/src/features/push/routes.ts:45-107`.

Source: `packages/integrations/src/web-push.ts:262-283` (1d5d98db9acf).

```typescript
/** One alert to one device. Never throws for a push service's answer; a network failure, or no answer in time, is a failed send. */
export async function sendWebPush(subscription: PushSubscription, message: PushMessage, options: SendOptions): Promise<SendResult> {
  const body = encryptPayload(subscription, Buffer.from(JSON.stringify(message), 'utf8'));
  const doFetch = options.fetch ?? fetch;
  try {
    const response = await doFetch(subscription.endpoint, {
      method: 'POST',
      signal: AbortSignal.timeout(options.timeoutMs ?? SEND_TIMEOUT_MS),
      headers: {
        Authorization: vapidAuthorization(subscription.endpoint, options.subject, options.vapidPrivateKey),
        'Content-Encoding': 'aes128gcm',
        'Content-Type': 'application/octet-stream',
        'Content-Length': String(body.length),
        TTL: String(options.ttlSeconds ?? DEFAULT_TTL_SECONDS),
        Urgency: 'normal',
      },
      body,
    });
    return { ok: response.ok, status: response.status, gone: response.status === 404 || response.status === 410 };
  } catch {
    return { ok: false, status: 0, gone: false };
  }
```

Source: `apps/api/src/features/push/routes.ts:45-107` (1d5d98db9acf).

```typescript
const routes: FeatureRoutes = {
  prefix: '/api/push',
  register(app) {
    app.get('/settings', async () => ({
      publicKey: app.config.vapidPrivateKey ? vapidPublicKey(app.config.vapidPrivateKey) : null,
    }));

    // The person's own phones (row-level security), so the app can tell whether the subscription this
    // browser holds is theirs: { phones: [{ endpointHash (hex), label, createdAt }] }.
    app.get('/subscriptions', async (request) => {
      const { rows } = await app.db.withUser(request.session, ({ client, userId }) =>
        client.query<{ endpoint_hash: Buffer; label: string; created_at: string }>(
          'select endpoint_hash, label, created_at from app.push_subscriptions where user_id = $1 order by created_at',
          [userId],
        ),
      );
      return { phones: rows.map((row) => ({ endpointHash: row.endpoint_hash.toString('hex'), label: row.label, createdAt: row.created_at })) };
    });

    app.put('/subscriptions', { schema: { body: subscriptionBody } }, async (request, reply) => {
      const subscription = parseSubscription(request.body.subscription);
      if (!subscription) throw new HttpError(422, 'invalid', BAD_SUBSCRIPTION);
      if (!app.config.vapidPrivateKey) throw new HttpError(409, 'not_set_up', PUSH_NOT_SET_UP);
      const masterKey = app.config.secretsKey;
      if (!masterKey) throw new HttpError(409, 'no_master_key', MASTER_KEY_MISSING_MESSAGE);
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
    });

    app.delete('/subscriptions', { schema: { body: endpointBody } }, async (request, reply) => {
      // Row-level security keeps this to the person's own rows; a phone that isn't there is already off.
      await app.db.withUser(request.session, ({ client, userId }) =>
        client.query('delete from app.push_subscriptions where user_id = $1 and endpoint_hash = $2', [userId, endpointHash(request.body.endpoint)]),
      );
      await reply.status(204).send();
    });

    app.post('/test', { config: { rateLimit: testAlertLimit } }, async (request) => {
      const session = request.session;
      return sendPushToPerson(app.db, session, app.config, {
        title: '[redacted name]',
        body: `Test alert for ${session.user.fullName}. Alerts on this phone are working.`,
        url: '/my-day',
        tag: 'test',
      });
```


#### Step 3 — Observe the durable effect

A signed-in member registered a push endpoint whose public-looking address redirected the application's POST to a private-network destination. The test established the server-side request primitive, not access to a particular AWS service.

Reachability of a protected AWS or VPC service was not tested.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/pocs/run-leads-003-010.mjs:202-225` (test source).

```javascript
async function poc005WebPushRedirect() {
  const webPushUrl = pathToFileURL(path.join(repo, 'packages/integrations/src/web-push.ts')).href;
  const { newVapidPrivateKey, parseVapidPrivateKey, parseSubscription, sendWebPush } = await import(webPushUrl);
  const certDir = mkdtempSync(path.join(os.tmpdir(), '[redacted name]-'));
  const keyFile = path.join(certDir, 'key.pem');
  const certFile = path.join(certDir, 'cert.pem');
  command('/usr/bin/openssl', ['req', '-x509', '-newkey', 'rsa:2048', '-nodes', '-keyout', keyFile, '-out', certFile, '-subj', '/CN=push.attacker.invalid', '-days', '1']);

  let targetHit = null;
  const target = http.createServer((request, response) => {
    const chunks = [];
    request.on('data', (chunk) => chunks.push(chunk));
    request.on('end', () => {
      targetHit = {
        method: request.method,
        path: request.url,
        bodyBytes: Buffer.concat(chunks).length,
        authorizationForwarded: request.headers.authorization !== undefined,
      };
      response.writeHead(204);
      response.end();
    });
  });
  await new Promise((resolve) => target.listen(0, '127.0.0.1', resolve));
```

## Proof of concept

The excerpts below come from the original local proof. They show the asserted transition and controls within the stated scope.

Setup: The member registers a public-looking push endpoint that redirects to an address reachable from the application network.

Recorded result: A signed-in member registered a push endpoint whose public-looking address redirected the application's POST to a private-network destination. The test established the server-side request primitive, not access to a particular AWS service.

Scope: Reachability of a protected AWS or VPC service was not tested. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/pocs/run-leads-003-010.mjs:202-225` (test source).

```javascript
async function poc005WebPushRedirect() {
  const webPushUrl = pathToFileURL(path.join(repo, 'packages/integrations/src/web-push.ts')).href;
  const { newVapidPrivateKey, parseVapidPrivateKey, parseSubscription, sendWebPush } = await import(webPushUrl);
  const certDir = mkdtempSync(path.join(os.tmpdir(), '[redacted name]-'));
  const keyFile = path.join(certDir, 'key.pem');
  const certFile = path.join(certDir, 'cert.pem');
  command('/usr/bin/openssl', ['req', '-x509', '-newkey', 'rsa:2048', '-nodes', '-keyout', keyFile, '-out', certFile, '-subj', '/CN=push.attacker.invalid', '-days', '1']);

  let targetHit = null;
  const target = http.createServer((request, response) => {
    const chunks = [];
    request.on('data', (chunk) => chunks.push(chunk));
    request.on('end', () => {
      targetHit = {
        method: request.method,
        path: request.url,
        bodyBytes: Buffer.concat(chunks).length,
        authorizationForwarded: request.headers.authorization !== undefined,
      };
      response.writeHead(204);
      response.end();
    });
  });
  await new Promise((resolve) => target.listen(0, '127.0.0.1', resolve));
```

Source: `security-audit/pocs/run-leads-003-010.mjs:259-273` (test source).

```javascript
        fetch: (_url, init) => fetch(`[redacted URL]`, init),
      });
    } finally {
      if (previousTlsSetting === undefined) delete process.env.NODE_TLS_REJECT_UNAUTHORIZED;
      else process.env.NODE_TLS_REJECT_UNAUTHORIZED = previousTlsSetting;
    }
    assert.deepEqual({ ok: result.ok, status: result.status }, { ok: true, status: 204 });
    assert.ok(targetHit);
    assert.equal(targetHit.method, 'POST');
    assert.equal(targetHit.path, '/private-poc');
    assert.ok(targetHit.bodyBytes > 0);
    return {
      verdict: 'mechanism-proved-under-isolated-dns-stub',
      directPrivateEndpointRejected: true,
      publicEndpointAccepted: true,
```

## Remediation

Disable push redirects or validate and pin every redirected destination and resolved address before connecting.
