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

# Cost-restricted sub-job manager loses the Notes editor

## Description

A manager authorized to update subcontractor notes cannot do so through the interface when price visibility is denied.

Prerequisite: An eligible manager opens a sub-job with Notes rights but without money permission.

Recorded result: Desktop and phone UI plus real API persistence reproduced the gap. Patched tests verify editing, price secrecy and write denial, lifecycle/access restrictions and preservation of a concurrent price change.

Evidence limits: The demonstrated issue is UI availability for an otherwise authorized notes operation. It does not grant price authority. Recorded desktop and phone browser checks do not establish native mobile-app behavior.

### Mechanism step by step

#### Step 1 — An authorized manager lacks price visibility

An eligible manager opens a sub-job with Notes rights but without money permission.

Source: `apps/web/src/features/subjobs/SubJobDrawer.tsx:62-87` (be12d54).

```tsx
  // Prices and notes, saved on Save.
  const [subPrice, setSubPrice] = useState('');
  const [customerPrice, setCustomerPrice] = useState('');
  const [notes, setNotes] = useState('');
  useEffect(() => {
    setSubPrice(subJob.subPriceCents !== null ? (subJob.subPriceCents / 100).toFixed(2) : '');
    setCustomerPrice(subJob.customerPriceCents !== null ? (subJob.customerPriceCents / 100).toFixed(2) : '');
    setNotes(subJob.notes ?? '');
  }, [subJob.id, subJob.subPriceCents, subJob.customerPriceCents, subJob.notes]);
  const save = useMutation({
    mutationFn: () => {
      const subPriceCents = parseDollarsToCents(subPrice);
      const customerPriceCents = parseDollarsToCents(customerPrice);
      if (subPriceCents === null || subPriceCents <= 0) throw new Error("Enter the sub's price, like 2,850.00.");
      if (customerPriceCents === null || customerPriceCents < 0) throw new Error('Enter the customer price, like 4,200.00.');
      return subJobsApi.update(subJob.id, { subPriceCents, customerPriceCents, notes: notes.trim() || null });
    },
    onSuccess: () => {
      refresh();
      toast('Saved');
    },
    onError: (error) => toast(errorMessage(error)),
  });

  const priceChanged =
    parseDollarsToCents(subPrice) !== subJob.subPriceCents || parseDollarsToCents(customerPrice) !== subJob.customerPriceCents || (notes.trim() || null) !== subJob.notes;
```

#### Step 2 — The UI hides Notes together with the price form

The Notes editor is coupled to the price form’s visibility gate, despite an authorized notes-only API operation.

Source: `apps/web/src/features/subjobs/SubJobDrawer.tsx:145-157` (be12d54).

```tsx
      {!subJob.seesMoney && <p className="muted small">The prices and the margin are hidden: your role doesn't see what we pay.</p>}
      {canManage && !finished && subJob.seesMoney ? (
        <div className="sj-price-form">
          <div className="fr">
            <Input label="Sub's price" inputMode="decimal" value={subPrice} onChange={(event) => setSubPrice(event.target.value)} />
            <Input label="Customer price" inputMode="decimal" value={customerPrice} onChange={(event) => setCustomerPrice(event.target.value)} />
          </div>
          <Textarea label="Notes" rows={2} value={notes} onChange={(event) => setNotes(event.target.value)} placeholder="What the sub does, and what we keep" />
          <div className="row">
            <Button variant="pri" busy={save.isPending} disabled={!priceChanged} onClick={() => save.mutate()}>
              Save prices
            </Button>
            <span className="muted small">
```

#### Step 3 — Observe the effect and its correction

Desktop and phone UI plus real API persistence reproduced the gap. Patched tests verify editing, price secrecy and write denial, lifecycle/access restrictions and preservation of a concurrent price change.

Separate Notes editing from prices and submit only notes for a notes-only save. Keep server authorization and price redaction.

Regression test for the corrected behavior:

Source: `e2e/tests/subjob-notes.spec.ts:25-39` (corrected implementation).

```typescript
    const drawer = page.getByRole('dialog');
    await expect(drawer.getByLabel('Notes', { exact: true })).toHaveValue('Initial notes');
    await expect(drawer.getByLabel("Sub's price", { exact: true })).toHaveCount(0);
    const view = await apiOk<{ subJob: { seesMoney: boolean; subPriceCents: null; customerPriceCents: null } }>(page, 'GET', `/api/sub-jobs/${id}`);
    expect(view.subJob).toMatchObject({ seesMoney: false, subPriceCents: null, customerPriceCents: null });
    await drawer.getByLabel('Notes', { exact: true }).fill('Changed notes');
    // A price changes after the manager read the record, before the notes request reaches the server.
    const update = await api(owner, 'PATCH', `/api/sub-jobs/${id}`, { subPriceCents: 123456 });
    expect(update.status).toBe(200);
    const outgoing = page.waitForRequest(r => r.method() === 'PATCH' && r.url().endsWith(`/api/sub-jobs/${id}`));
    const saved = page.waitForResponse(r => r.request().method() === 'PATCH' && r.url().endsWith(`/api/sub-jobs/${id}`));
    await drawer.getByRole('button', { name: 'Save notes', exact: true }).click();
    expect((await outgoing).postDataJSON()).toEqual({ notes: 'Changed notes' });
    expect((await saved).status()).toBe(200);
    expect(sql(`select notes || '|' || sub_price_cents from app.sub_jobs where id = '${id}'`)).toBe('Changed notes|123456');
```

## Proof of concept

The excerpts below test the corrected behavior. They support the remediation within the tested conditions, rather than independently demonstrating the original vulnerability.

Setup: An eligible manager opens a sub-job with Notes rights but without money permission.

Recorded result: Desktop and phone UI plus real API persistence reproduced the gap. Patched tests verify editing, price secrecy and write denial, lifecycle/access restrictions and preservation of a concurrent price change.

Scope: The demonstrated issue is UI availability for an otherwise authorized notes operation. It does not grant price authority. Recorded desktop and phone browser checks do not establish native mobile-app behavior.

Proof availability: Excerpts and source references; original harness not distributed.

## Relevant test excerpts

Regression test: `e2e/tests/subjob-notes.spec.ts`.

Source: `e2e/tests/subjob-notes.spec.ts:25-39` (corrected implementation).

```typescript
    const drawer = page.getByRole('dialog');
    await expect(drawer.getByLabel('Notes', { exact: true })).toHaveValue('Initial notes');
    await expect(drawer.getByLabel("Sub's price", { exact: true })).toHaveCount(0);
    const view = await apiOk<{ subJob: { seesMoney: boolean; subPriceCents: null; customerPriceCents: null } }>(page, 'GET', `/api/sub-jobs/${id}`);
    expect(view.subJob).toMatchObject({ seesMoney: false, subPriceCents: null, customerPriceCents: null });
    await drawer.getByLabel('Notes', { exact: true }).fill('Changed notes');
    // A price changes after the manager read the record, before the notes request reaches the server.
    const update = await api(owner, 'PATCH', `/api/sub-jobs/${id}`, { subPriceCents: 123456 });
    expect(update.status).toBe(200);
    const outgoing = page.waitForRequest(r => r.method() === 'PATCH' && r.url().endsWith(`/api/sub-jobs/${id}`));
    const saved = page.waitForResponse(r => r.request().method() === 'PATCH' && r.url().endsWith(`/api/sub-jobs/${id}`));
    await drawer.getByRole('button', { name: 'Save notes', exact: true }).click();
    expect((await outgoing).postDataJSON()).toEqual({ notes: 'Changed notes' });
    expect((await saved).status()).toBe(200);
    expect(sql(`select notes || '|' || sub_price_cents from app.sub_jobs where id = '${id}'`)).toBe('Changed notes|123456');
```
