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

# Customer-link redaction removes unrelated reference codes

## Description

Legitimate third-party references can disappear from filed mail or prevent a manual message from being composed.

Prerequisite: A message contains a standalone 43-character base64url reference that is not an [redacted name] customer bearer.

Recorded result: The first-delivery scanner removed a 43-character sample while preserving 42- and 44-character controls. Differential and supporting-library checks in the final review confirm the second-delivery correction within the tested reference and bearer cases.

Evidence limits: The historical vulnerable revision 5447910e84e7 is unavailable. The excerpts show corrected revision be12d54, whose comments describe the old token-shape rule and EVERY_CODE primitive. The 42/43/44-character differential result is documented in the assessment, rather than independently reproduced by these excerpts.

### Mechanism step by step

#### Step 1 — A legitimate external reference resembles a customer bearer

A message contains a standalone 43-character base64url reference that is not an [redacted name] customer bearer.

Source: `packages/integrations/src/customer-links.ts:178-201` (be12d54).

```typescript
 * A customer link's token standing on its own (M-13, re-attacked): a run of exactly TOKEN_CHARS base64url
 * characters. Every link the app makes ends in one, and a browser reaches the customer's page by more names
 * than a scan of the install's host and path can list (its IP address in any form, "localhost." or [::1], a
 * backslash or a dot segment in the path, a zero-padded port, an ideographic full stop in the name), while a
 * person who knows the app's paths needs only the token. So the token is what is looked for, on any address or
 * none. Not one: a run followed by "=" or "@" (a base64 hash's padding, a Message-ID), a run next to a "+" or
 * followed by a "/", and a run that ends a line of a base64 block; the pieces of base64 between those are of
 * any length.
 * Such a run is a customer's token only when the app minted it (N-01: another service's ticket or reference
 * code is often 32 random bytes in base64url too, and stays): the scan asks `isMinted` of each run it finds,
 * and withMintedTokens answers from the database (app.minted_customer_link_tokens, migration 0128). A token
 * that opens a customer's page has its hash in one of the five link tables, so a run that is in none of them
 * opens nothing. With that answer the "not one" list above is dropped (ANY_TOKEN_RUN, N-01 re-attacked): it only
 * kept the M-13 rule (EVERY_CODE) from removing other codes, and a minted token followed by "/" still opens the
 * quote page, as does one alone on a line after a line of base64.
 */
// The character before is matched rather than looked behind for: twice as fast on a large file's text.
const TOKEN_RUN = new RegExp(`(?:^|[^A-Za-z0-9_+-])([A-Za-z0-9_-]{${TOKEN_CHARS}})(?![A-Za-z0-9_+/=@-])`, 'g');
/** Every token-long run of base64url characters, whatever is beside it: what is asked about when the database answers. */
const ANY_TOKEN_RUN = new RegExp(`(?:^|[^A-Za-z0-9_-])([A-Za-z0-9_-]{${TOKEN_CHARS}})(?![A-Za-z0-9_-])`, 'g');

/** A line of base64 (a mail header's signature may start it with "b="): the length of its base64. */
const BASE64_LINE = /^[\s>]*(?:[a-z]{1,2}=)?([A-Za-z0-9+/]+={0,2})\s*$/;

```

#### Step 2 — The scanner formerly treated shape as sufficient identity

The original token-wide scanner classified references by alphabet and length without establishing that the application minted the token.

Source: `packages/integrations/src/customer-links.ts:221-225` (be12d54).

```typescript
export type IsMinted = (token: string) => boolean;

/** Every code counts as a customer's (the M-13 rule): what a scan with no answer from the database does. */
export const EVERY_CODE: IsMinted = () => true;

```

#### Step 3 — Observe the effect and its correction

The first-delivery scanner removed a 43-character sample while preserving 42- and 44-character controls. Differential and supporting-library checks in the final review confirm the second-delivery correction within the tested reference and bearer cases.

Preserve unrelated references while retaining encoded customer-bearer protection. The reviewed second-delivery implementation passes the relevant scanner and integration assertions.

Regression test for the corrected behavior:

Source: `packages/integrations/src/customer-links.test.ts:516-548` (corrected implementation).

```typescript
// N-01 (the remediation review of 27 October 2026): another service's code of a token's shape is not a token

/** Another service's ticket references: 42, 43 and 44 base64url characters, none of them minted by the app. */
const TICKET_42 = 'ZD-7Hq2LmN9pR4sT6vW8xY0zA1bC3dE5fG7hJ9kL0m';
const TICKET_43 = 'ZD-7Hq2LmN9pR4sT6vW8xY0zA1bC3dE5fG7hJ9kL0mN';
const TICKET_44 = 'ZD-7Hq2LmN9pR4sT6vW8xY0zA1bC3dE5fG7hJ9kL0mNp';

/** The database's answer as a test gives it: only TOKEN is ours. Every lookup is recorded. */
function lookup(ours: readonly string[] = [TOKEN]): MintedTokens & { asked: string[][] } {
  const asked: string[][] = [];
  const minted = async (tokens: readonly string[]) => {
    asked.push([...tokens]);
    return new Set(tokens.filter((token) => ours.includes(token)));
  };
  return Object.assign(minted, { asked });
}

test('N-01: a third-party code of 42, 43 or 44 characters stays in a filed email; the minted token beside it goes', async () => {
  assert.equal(TICKET_42.length, 42);
  assert.equal(TICKET_43.length, 43);
  assert.equal(TICKET_44.length, 44);
  const text = `Your ticket ${TICKET_43} is open. Old ref ${TICKET_42}, new ref ${TICKET_44}.\nRef: ${TICKET_43}`;
  const minted = lookup();
  assert.equal(await customerLinksRemoved(text, ORIGINS, minted), text, "nothing of it is the app's");
  assert.deepEqual(minted.asked, [[TICKET_43]], 'one lookup, of the one token-shaped code');
  // The auditor's reproduction, as a pure call: the scan with the database's answer keeps the code.
  assert.equal(withoutCustomerLinks(`Ticket ${TICKET_43}`, ORIGINS, () => false), `Ticket ${TICKET_43}`);
  // The minted token on its own still goes, the code next to it stays.
  assert.equal(await customerLinksRemoved(`Ticket ${TICKET_43}, your code ${TOKEN}.`, ORIGINS, lookup()), `Ticket ${TICKET_43}, your code ${CUSTOMER_LINK_REMOVED}.`);
  // The ticket code in a link on another service's address stays too.
  const vendor = `[redacted URL]`;
  assert.equal(await customerLinksRemoved(`See ${vendor}`, ORIGINS, lookup()), `See ${vendor}`);
});
```

## Proof of concept

The excerpts below test the corrected behavior. They support the remediation within the tested conditions, rather than independently demonstrating the original vulnerability.

Setup: A message contains a standalone 43-character base64url reference that is not an [redacted name] customer bearer.

Recorded result: The first-delivery scanner removed a 43-character sample while preserving 42- and 44-character controls. Differential and supporting-library checks in the final review confirm the second-delivery correction within the tested reference and bearer cases.

Scope: The historical vulnerable revision 5447910e84e7 is unavailable. The excerpts show corrected revision be12d54, whose comments describe the old token-shape rule and EVERY_CODE primitive. The 42/43/44-character differential result is documented in the assessment, rather than independently reproduced by these excerpts.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Regression test: `packages/integrations/src/customer-links.test.ts`.

Source: `packages/integrations/src/customer-links.test.ts:516-548` (corrected implementation).

```typescript
// N-01 (the remediation review of 27 October 2026): another service's code of a token's shape is not a token

/** Another service's ticket references: 42, 43 and 44 base64url characters, none of them minted by the app. */
const TICKET_42 = 'ZD-7Hq2LmN9pR4sT6vW8xY0zA1bC3dE5fG7hJ9kL0m';
const TICKET_43 = 'ZD-7Hq2LmN9pR4sT6vW8xY0zA1bC3dE5fG7hJ9kL0mN';
const TICKET_44 = 'ZD-7Hq2LmN9pR4sT6vW8xY0zA1bC3dE5fG7hJ9kL0mNp';

/** The database's answer as a test gives it: only TOKEN is ours. Every lookup is recorded. */
function lookup(ours: readonly string[] = [TOKEN]): MintedTokens & { asked: string[][] } {
  const asked: string[][] = [];
  const minted = async (tokens: readonly string[]) => {
    asked.push([...tokens]);
    return new Set(tokens.filter((token) => ours.includes(token)));
  };
  return Object.assign(minted, { asked });
}

test('N-01: a third-party code of 42, 43 or 44 characters stays in a filed email; the minted token beside it goes', async () => {
  assert.equal(TICKET_42.length, 42);
  assert.equal(TICKET_43.length, 43);
  assert.equal(TICKET_44.length, 44);
  const text = `Your ticket ${TICKET_43} is open. Old ref ${TICKET_42}, new ref ${TICKET_44}.\nRef: ${TICKET_43}`;
  const minted = lookup();
  assert.equal(await customerLinksRemoved(text, ORIGINS, minted), text, "nothing of it is the app's");
  assert.deepEqual(minted.asked, [[TICKET_43]], 'one lookup, of the one token-shaped code');
  // The auditor's reproduction, as a pure call: the scan with the database's answer keeps the code.
  assert.equal(withoutCustomerLinks(`Ticket ${TICKET_43}`, ORIGINS, () => false), `Ticket ${TICKET_43}`);
  // The minted token on its own still goes, the code next to it stays.
  assert.equal(await customerLinksRemoved(`Ticket ${TICKET_43}, your code ${TOKEN}.`, ORIGINS, lookup()), `Ticket ${TICKET_43}, your code ${CUSTOMER_LINK_REMOVED}.`);
  // The ticket code in a link on another service's address stays too.
  const vendor = `[redacted URL]`;
  assert.equal(await customerLinksRemoved(`See ${vendor}`, ORIGINS, lookup()), `See ${vendor}`);
});
```
