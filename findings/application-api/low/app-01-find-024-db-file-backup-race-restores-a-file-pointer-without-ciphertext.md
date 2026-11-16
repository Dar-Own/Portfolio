# Source-code-based

Project: Anonymous construction SaaS (APP-01)
Category: Application/API
Severity: Low (conditional)
Engagement: Private engagement
Chain: Not applicable
Tech stack: TypeScript/Node.js, Fastify/Zod, PostgreSQL/SQL/RLS, React/Vite, background workers, Docker/Caddy; provider adapters as applicable below
Evidence: Source inspection and a partial or modeled proof of the original behavior; the complete application or provider flow is not established by these excerpts.
Remediation: Acknowledged, reviewed code as of 03 November 2026; deployment unverified
Contribution: Team security engagement; no claim of sole discovery.
Affected revision: v1.0.0-rc.1 / 1d5d98db9acf

Name-redacted material. Original relative filenames and code remain searchable and can identify the project. Bracketed substitutions visibly mark redactions inside excerpts.

# DB/file backup race restores a file pointer without ciphertext

## Description

A backup can restore a document row and recoverable key without its encrypted file bytes, leaving that document unreadable.

Prerequisite and affected workflow: An authorized file deletion falls between the database dump and file archive, followed by a restore.

Established observation: A controlled backup sequence dumped the database, deleted a document, archived the files, and restored both parts. The restored database referenced a document key and row whose encrypted file bytes were absent, leaving the document unreadable.

Limits: The full encrypted off-host backup and restore path was not exercised. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

### Mechanism step by step

#### Step 1 — Reach the affected workflow

An authorized file deletion falls between the database dump and file archive, followed by a restore.

Source: `deploy/backup/backup.sh:78-107` (1d5d98db9acf).

```bash
echo "Backup $stamp: database $DB_NAME from $PGHOST..."
pg_dump --format=custom --no-password --dbname="$DB_NAME" --file="$work/$name.dump"
# A dump pg_restore can't list is no backup at all.
pg_restore --list "$work/$name.dump" > /dev/null
# The full proof, while the dump is still readable here: restored into a scratch database (row counts,
# the master key it records), which is dropped again. A dump that does not restore is no backup either.
if ! /bin/bash "$here/restore.sh" --check "$work/$name.dump" > "$work/check.log"; then
  cat "$work/check.log"
  echo "The dump did not restore into a scratch database (see above): no backup was written." >&2
  exit 1
fi
cat "$work/check.log"
counts="$(sed -nE 's/^    ([a-z]+ [0-9]+)\r?$/\1/p' "$work/check.log" | paste -sd ',' - | sed 's/,/, /g')"
echo "  database: $name.dump ($(du -h "$work/$name.dump" | cut -f1)), listed by pg_restore and restored into a scratch database"
plain=("$name.dump")

if [ -d "$FILES_DIR" ]; then
  tar -czf "$work/[redacted name]-files-$stamp.tar.gz" -C "$(dirname "$FILES_DIR")" "$(basename "$FILES_DIR")"
  echo "  files:    [redacted name]-files-$stamp.tar.gz ($(du -h "$work/[redacted name]-files-$stamp.tar.gz" | cut -f1))"
  plain+=("[redacted name]-files-$stamp.tar.gz")
else
  echo "  files:    none ($FILES_DIR does not exist)"
fi

# The settings file: the master key (AOS_SECRETS_KEY) the dump's two-factor keys and saved API keys open
# with, and every password, so that a whole new NAS can be brought back from an off-site copy of this set.
if [ -f "$BACKUP_ENV_FILE" ]; then
  cp "$BACKUP_ENV_FILE" "$work/$name.env"
  echo "  settings: $name.env (.env, with the master key this dump's data is bound to)"
  plain+=("$name.env")
```

#### Step 2 — Cross the missing state or authority check

The backup takes the database dump and file archive at different times without a coherent snapshot.

Source: `apps/api/src/core/files.ts:305-326` (1d5d98db9acf).

```typescript
    async forget(session, fileIds) {
      // A caller may hand us nulls (a record that had no file) and the same id twice.
      const wanted = [...new Set(fileIds.filter((id): id is string => typeof id === 'string' && UUID.test(id)))];
      if (wanted.length === 0) return 0;
      try {
        let count = 0;
        // In batches, because the database takes at most 50 at a time on purpose (an appointment with a
        // hundred site photos on it is one request).
        for (let from = 0; from < wanted.length; from += FORGET_BATCH) {
          const batch = wanted.slice(from, from + FORGET_BATCH);
          // The record first, then the bytes: bytes with no record are what the nightly sweep finds, a
          // record with no bytes is a file that cannot be opened.
          const gone = await db.withUser(session, async (ctx) => {
            const { rows } = await ctx.client.query<{ storage_key: string }>('select k as storage_key from app.forget_unused_files($1::uuid[]) k', [batch]);
            return rows.map((row) => row.storage_key);
          });
          for (const storageKey of gone) await discardKey(storageKey);
          count += gone.length;
        }
        return count;
      } catch (error) {
        logger.error({ err: error, company: session.company.id }, 'A deleted file could not be cleared away; the nightly sweep will take it');
```


#### Step 3 — Observe the durable effect

A controlled backup sequence dumped the database, deleted a document, archived the files, and restored both parts. The restored database referenced a document key and row whose encrypted file bytes were absent, leaving the document unreadable.

The full encrypted off-host backup and restore path was not exercised.

Test assertion for this transition (test code, not application implementation):

Source: `security-audit/pocs/run-backup-snapshot-gap.mjs:117-140` (test source).

```javascript
    const unwrapped = unwrapOwnFileKey(restoredKey, tenantId, id, Buffer.from(wrappedHex, 'hex'));
    assert.ok(unwrapped, 'restored key must unwrap with the backed-up master key');
    let readable = false, failure = null;
    if (exists) {
      try { readable = sha(await readEncryptedFile(file, unwrapped, { tenantId, fileId: id }, { expectedSha256: Buffer.from(plainHash, 'hex') })) === plainHash; }
      catch (error) { failure = String(error); }
    } else failure = 'ciphertext missing';
    output.controls[row.label] = { rowRestored: true, ciphertextPresent: exists, ciphertextSha256: exists ? sha(readFileSync(file)) : null,
      expectedCiphertextSha256: row.ciphertextSha256, keyUnwrapped: true, readable, failure };
  }
  assert.equal(output.controls.stable.readable, true);
  assert.equal(output.controls.changed.ciphertextPresent, false);
  assert.equal(output.controls.changed.readable, false);
  output.fault.archiveEntries = run('tar', ['-tzf', archive]).split('\n');
  output.fault.restoredFileRows = restoredRows.length;
  output.fault.restoredMasterKeyMatches = restoredKey.equals(key);
  output.limits.push('The dump and tar follow backup.sh order; age wrapping and the backup.sh outer scheduler were not exercised.');
  output.limits.push('The local macOS tar lacks GNU --skip-old-files; a test-only wrapper maps it to BSD tar -k for restore.sh --files.');
  output.limits.push('The file removal and SQL deletion model a committed authorized deletion; the actual API delete route was not invoked.');
  output.limits.push('An inert spatial shim was required locally; no spatial security claim follows.');
  writeFileSync(path.join(repo, 'security-audit/pocs/results/2026-10-08-backup-snapshot-gap.json'), JSON.stringify(output, null, 2) + '\n');
  console.log(JSON.stringify({ result: 'PROVED_LOCAL_SNAPSHOT_GAP', rows: output.controls, boundary: output.boundary, migrations: migrations.length }));
} finally {
  if (started) spawnSync(path.join(pgBin, 'pg_ctl'), ['-D', data, '-m', 'immediate', '-w', 'stop'], { stdio: 'ignore' });
```

## Proof of concept

The original proof isolates a narrower component or modeled transition. It does not demonstrate every step of the complete external workflow.

Setup: An authorized file deletion falls between the database dump and file archive, followed by a restore.

Recorded result: A controlled backup sequence dumped the database, deleted a document, archived the files, and restored both parts. The restored database referenced a document key and row whose encrypted file bytes were absent, leaving the document unreadable.

Scope: The full encrypted off-host backup and restore path was not exercised. The original assessment used a pre-production source release. Local fixtures and captured provider handoffs do not establish public reachability, actual external delivery, real charges or production build equivalence. Private lab evidence, where specifically recorded above, applies only to that narrow mechanism.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Source: `security-audit/pocs/run-backup-snapshot-gap.mjs:117-140` (test source).

```javascript
    const unwrapped = unwrapOwnFileKey(restoredKey, tenantId, id, Buffer.from(wrappedHex, 'hex'));
    assert.ok(unwrapped, 'restored key must unwrap with the backed-up master key');
    let readable = false, failure = null;
    if (exists) {
      try { readable = sha(await readEncryptedFile(file, unwrapped, { tenantId, fileId: id }, { expectedSha256: Buffer.from(plainHash, 'hex') })) === plainHash; }
      catch (error) { failure = String(error); }
    } else failure = 'ciphertext missing';
    output.controls[row.label] = { rowRestored: true, ciphertextPresent: exists, ciphertextSha256: exists ? sha(readFileSync(file)) : null,
      expectedCiphertextSha256: row.ciphertextSha256, keyUnwrapped: true, readable, failure };
  }
  assert.equal(output.controls.stable.readable, true);
  assert.equal(output.controls.changed.ciphertextPresent, false);
  assert.equal(output.controls.changed.readable, false);
  output.fault.archiveEntries = run('tar', ['-tzf', archive]).split('\n');
  output.fault.restoredFileRows = restoredRows.length;
  output.fault.restoredMasterKeyMatches = restoredKey.equals(key);
  output.limits.push('The dump and tar follow backup.sh order; age wrapping and the backup.sh outer scheduler were not exercised.');
  output.limits.push('The local macOS tar lacks GNU --skip-old-files; a test-only wrapper maps it to BSD tar -k for restore.sh --files.');
  output.limits.push('The file removal and SQL deletion model a committed authorized deletion; the actual API delete route was not invoked.');
  output.limits.push('An inert spatial shim was required locally; no spatial security claim follows.');
  writeFileSync(path.join(repo, 'security-audit/pocs/results/2026-10-08-backup-snapshot-gap.json'), JSON.stringify(output, null, 2) + '\n');
  console.log(JSON.stringify({ result: 'PROVED_LOCAL_SNAPSHOT_GAP', rows: output.controls, boundary: output.boundary, migrations: migrations.length }));
} finally {
  if (started) spawnSync(path.join(pgBin, 'pg_ctl'), ['-D', data, '-m', 'immediate', '-w', 'stop'], { stdio: 'ignore' });
```

## Remediation

Take a coherent database-and-files snapshot and verify every restored file pointer and key in an isolated recovery drill.
