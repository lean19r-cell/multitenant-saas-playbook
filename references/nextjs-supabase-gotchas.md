# Next.js + Supabase/Postgres gotchas

Concrete bug patterns from this stack, each confirmed in a real production app — not theoretical. Each one cost real debugging time before the mechanism was understood. Read the mechanism, not just the fix — the mechanism is what tells you where else to look.

## 1. A thrown Server Action error is invisible in production

**The mechanism:** Next.js redacts any error that propagates out of a `"use server"` function in production — it replaces the real message with a generic one plus an opaque digest, on purpose (this is documented, intentional behavior, not a bug in Next.js). Locally, in dev mode, you see the real message, so this is invisible until someone hits it against a real deploy.

**What it looks like from the outside:** a feature that clearly has real, specific validation ("no podés reintegrar una venta que nunca se cobró") instead shows a generic, unhelpful error in production, and a user reasonably concludes the whole feature is broken rather than that they hit one specific, well-defined business rule.

**Broken:**
```ts
"use server";
export async function doTheThing(id: string) {
  const { error } = await supabase.rpc("the_thing", { p_id: id });
  if (error) throw error;          // real message dies in prod
}
```

**Fixed — catch it, return it as data:**
```ts
"use server";
export type Result = { ok: true } | { ok: false; error: string };
export async function doTheThing(id: string): Promise<Result> {
  const { error } = await supabase.rpc("the_thing", { p_id: id });
  if (error) return { ok: false, error: error.message };
  return { ok: true };
}
```

The caller then renders `result.error` directly — it's just a string in a normal render path, nothing about it triggers Next's Server Action error handling.

**Where to look for this:** grep every `"use server"` file for `throw` following an `if (error)` check, or for an un-checked `error` that gets returned/ignored silently (the opposite failure — swallowing it so nothing shows at all). If a codebase has fixed this once, audit *every* Server Action file for the same shape — this is exactly the kind of thing that gets copy-pasted as a starting template and then multiplies. In one real audit pass across a ~25-file project, this turned up in **21 of 23 Server Action files, roughly 99 functions** — a single fixed instance (a "no se puede anular" bug report) was the tip of something present almost everywhere else in the codebase, simply never yet reported because nobody had hit those specific validation paths in production yet.

**Two traps found only by auditing exhaustively, not by spot-checking:**
- A function *assumed* to already follow the good pattern (because a sibling function in the same file did) turned out not to — always verify each function individually, don't infer from one neighbor.
- A function with a code comment explicitly discussing this exact redaction problem, sitting right next to a `throw new Error(error.message)` — someone had diagnosed the mechanism correctly and then not actually finished applying the fix (wrapping the real message in `Error` doesn't help; it's still a `throw`, still redacted). A comment describing the problem is not evidence the problem is fixed — check the control flow, not the prose next to it.

## 2. Changing a Postgres function's parameter count creates a silent overload, not a replacement

**The mechanism:** Postgres identifies a function by name *and* parameter type signature. `create or replace function foo(a text, b text)` does not touch an existing `foo(a text)` — it creates a second, distinct function. PostgREST then can't disambiguate a call with the old arity and throws "function is not unique" (`PGRST203`), which surfaces as a confusing RPC failure that has nothing obviously to do with the migration you just wrote.

**Fixed:**
```sql
-- Explicit drop of the OLD signature before creating the new one
drop function if exists public.confirmar_pedido(text, text, boolean);

create function public.confirmar_pedido(
  p_canal text default 'Mostrador',
  p_forma_pago text default 'Efectivo',
  p_pago_pendiente boolean default false,
  p_referencia_pago text default null   -- the new parameter
)
returns uuid ...
```

Applies every time a parameter is added or removed from a function already exposed via RPC — not just when the signature "looks" different, since a single added parameter with a default is exactly the case that's easy to assume `create or replace` handles fine.

## 3. `empresa_id` + RLS is the tenant boundary — not a WHERE clause in application code

Every table that holds tenant data gets an `empresa_id` column and a `for select/insert/update/delete using (empresa_id = current_empresa_id())`-shaped policy (`current_empresa_id()` reading the value off the authenticated session). The application code never filters by tenant manually — it can't, by construction, leak cross-tenant, because the database refuses the query regardless of what the client asked for.

The recurring mistake is *not* forgetting the column — it's writing a query that fetches something correctly-scoped but then treating "all rows this policy returned" as "all rows relevant to what I'm about to render," which is a different, UI-level bug (see next item) rather than a security one. Verify which one you're looking at before assuming a security hole: check whether the policy itself is missing the `empresa_id` predicate (real leak) versus whether the query is tenant-safe but returning a broader set than the UI should display (UX gap, item 4 below).

## 4. A filter populated from the full lookup table, not from values actually present

**The mechanism:** a filter/pill/dropdown built as `theWholeCatalog.map(item => <option>...)` looks correct and reads correct in review — every valid value is technically an option. But a "filter" is supposed to narrow an already-rendered list, and any catalog entry with zero matching rows in that list is a dead end: clicking it always produces "no results," and the user has no way to tell which pills are real without clicking each one.

**Broken:**
```tsx
<FiltrosCategoria categorias={categorias} .../>  {/* every category the tenant ever created */}
```

**Fixed — derive the option list from what's actually being filtered:**
```tsx
const categoriasEnUso = categorias.filter(c => productos.some(p => p.categoria_id === c.id));
<FiltrosCategoria categorias={categoriasEnUso} .../>
```

**The distinction that matters:** this fix applies to *filter/narrow* contexts only. An *assignment* dropdown (choosing a category while creating/editing a record) should absolutely show the full catalog, including entries with nothing assigned yet — that's not a bug, it's the only way to ever assign the first item to an empty category. Don't "fix" those; check which kind of control you're looking at before touching it. When one instance of this bug gets fixed, the highest-value follow-up is grepping every other filter/pill-selector in the app for the same "maps the raw catalog with no `.filter()` against the actual data" shape — this tends to repeat across every screen that was built from the same original template, since a filter and an assignment dropdown often look identical in the code until you check what they're feeding. One real audit pass found the exact same shape twice more in a second, structurally unrelated part of the app (a "filter contracts by tenant" and a "filter expenses by property" control, both built from the same original filter-bar template as the one already fixed) — confirming this really does propagate across a codebase via copy-paste, not just in theory. A hardcoded, small, fixed-enum filter (three or four literal string options, not a growable catalog table) carries the same theoretical risk but is much lower priority — the failure mode only bites once real data creates a channel/status that's never actually used, which is rarer and lower-stakes than a user-editable catalog.

## 5. Hand-rolled bar/progress visuals: percentage height needs a *direct* parent with resolved height

**The mechanism:** CSS percentage-height resolution is checked against the immediate parent's own computed height, not any ancestor further up. A grandparent with `h-40` does nothing for a percentage set on a grandchild if the child in between is sized by content (`height: auto`, e.g. plain `flex flex-col` with no explicit height and no `flex-1`/stretch pulling a real size down to it). The element silently renders at `height: 0` — no console error, no layout warning, it just doesn't show up.

**Broken:**
```tsx
<div className="flex h-40 items-end gap-2">
  <div className="flex flex-1 flex-col items-center justify-end">  {/* height: auto */}
    <div style={{ height: `${pct}%` }} className="bg-blue-500" />   {/* resolves against auto → 0 */}
  </div>
</div>
```

**Fixed — an intermediate wrapper that actually inherits a resolved size:**
```tsx
<div className="flex h-40 items-end gap-2">
  <div className="flex h-full flex-1 flex-col items-center gap-1">      {/* h-full: resolves against h-40 */}
    <span>{label}</span>
    <div className="flex w-full flex-1 items-end">                     {/* flex-1 in a sized flex column: resolved */}
      <div style={{ height: `${Math.max(pct, 2)}%` }} className="bg-blue-500" />
    </div>
  </div>
</div>
```

Note the `Math.max(pct, 2)` — a real zero-or-near-zero data point should still render a sliver, not vanish, or it reads as the same bug even once the mechanism is fixed.

**Width percentages on block-level elements mostly don't have this problem** — a block box's `width: auto` resolves to fill its containing block by default (that's normal-flow behavior), where `height: auto` shrink-wraps to content. A `<div style={{width: pct+'%'}}>` inside a plain (non-flex, non-inline-block) block-level track div is very likely fine without an intermediate wrapper; verify by checking whether anything in the ancestor chain uses `inline-block`/flex with a size determined by content, but don't assume every percentage-styled element needs the same treatment as the height case.

## 6. `npm install` warnings are not build failures

`npm warn deprecated ...` and `npm warn allow-scripts ...` (unapproved postinstall scripts) show up on every install of a project with a handful of common transitive dependencies, succeed or fail identically regardless of these warnings, and tell you nothing about whether the actual build/deploy worked. When troubleshooting "did my deploy work," skip past these lines to the part of the log that says `Compiled successfully`, lists the actual routes, and ends in `Build Completed` / `Deployment completed` (or their equivalents on the target platform) — and check the deployment's commit hash against the merge commit you expect, since that's the one piece of log output that actually answers the question.

## 7. Infra gotchas specific to a local Supabase + Docker dev loop

- **Stale Kong routing after `supabase db reset`:** the local API gateway container can end up pointing at a stale upstream IP after a reset, producing opaque `502`/connection errors that look like the reset itself failed. Fix: `docker restart <kong-container-name>`, wait a few seconds, verify `curl http://127.0.0.1:54321/auth/v1/health` returns 200 before retrying whatever failed.
- **Never run a production build and a dev server in the same directory at the same time.** They share a `.next/` build-cache directory and will corrupt each other's output — the failure mode is bizarre (a page renders as raw unstyled HTML, or a stale dev bundle serves after a supposedly-clean build) and looks exactly like a real application bug. If you need both, use separate worktrees/checkouts, or stop one before running the other.
- **Docker Desktop can silently stop during a long idle period**, and every test in the suite fails at once. A sudden 100%-failure run after a long gap is a Docker-down signal before it's a real-regression signal — check `docker ps` first. On Windows the symptom is a named-pipe error rather than "connection refused", and it can mean the Docker Desktop *app* is off, not just the containers.
- **The local Supabase database is shared by every git worktree.** A `supabase db reset` run from any one worktree wipes it under all the others working in parallel. Apply new migrations with `supabase migration up`; if you think a reset is needed, stop and ask. Data that "disappears" or results that contradict each other mid-verification: suspect another session before suspecting your own code.
- **Migration numbers collide between parallel branches.** Before merging, check that the branch's number range isn't already taken on the base branch (this happened repeatedly, once forcing a renumber of 24 files). Before creating a migration, list the real highest number (`ls supabase/migrations | sort | tail -3`) — earlier tasks in the same plan shift it.

## 8. `position: sticky` inside a sidebar-layout content pane needs a *height-bounded* scroll container, not just `overflow-y-auto`

**The mechanism:** a common layout for "fixed sidebar + independently-scrolling content" is `flex` row with the content pane getting `flex-1 overflow-y-auto`. That alone is not enough. If the row's own height is only `min-h-screen` (a floor, not a ceiling) instead of `h-screen` (fixed), the content pane's height stretches to fit whatever it contains instead of being capped at the viewport — so it never actually overflows *itself*, and the real scrolling happens on the document/`html` element instead. The pane still has `overflow-y: auto` declared, and per spec that alone is enough to make it the "nearest scrolling ancestor" that any descendant `position: sticky` element positions against — **even though this ancestor never scrolls in practice.** The browser is not lying: `getComputedStyle(el).position` genuinely returns `"sticky"`. But since the scroll container it's pinned against never moves independently of the document, the element just travels 1:1 with the real page scroll — visually indistinguishable from `position: static`.

**What it looks like from the outside:** a feature that was implemented correctly, code-reviewed, and "verified" (CSS class present, computed style checked) still gets reported by a real user as "doesn't float, stays at the top" — and every earlier verification pass looked fine because it only ever asked the DOM "what does this compute to," never "does this visually track the scroll the way it should."

**Broken:**
```tsx
<div className="flex min-h-screen ...">      {/* no ceiling — grows to fit content */}
  <Sidebar />
  <div className="flex-1 overflow-y-auto ...">
    <SomeCard className="sticky top-6 ..." />   {/* getComputedStyle says sticky; never actually sticks */}
  </div>
</div>
```

**Fixed — bound the row to the viewport so the inner pane actually has to scroll:**
```tsx
<div className="flex h-screen overflow-hidden ...">   {/* fixed ceiling */}
  <Sidebar />
  <div className="flex-1 overflow-y-auto ...">          {/* now genuinely scrolls when content overflows */}
    <SomeCard className="sticky top-6 ..." />           {/* now really sticks */}
  </div>
</div>
```

**The verification lesson this exposes:** "verify by seeing it, not by inferring it" (see SKILL.md) has a sharp edge here — checking `getComputedStyle(el).position === 'sticky'` feels like seeing it, but it only confirms the CSS declaration parsed and applied, not that the *effect* the declaration is supposed to produce actually happens. The only real check is behavioral: change the scroll position (real user scroll, or `scrollContainer.scrollTop = N` in a script) and read the element's `getBoundingClientRect()` before and after. If the top coordinate keeps changing 1:1 with the scroll delta instead of clamping at the offset you set, it isn't actually sticking, no matter what the computed style says. This also means: when a user reports "the thing you fixed still doesn't work" on something you already "verified," don't reflexively assume the user is looking at a stale deploy or the wrong viewport — check whether your own verification actually exercised the effect, or just its precondition.

## 9. A "runs globally across all tenants" batch function needs row locks once tests run concurrently against a shared DB

**The mechanism:** a function meant to run as a cron job over every tenant's rows (e.g. generating this month's installments for every active contract, across every company) is correct for its real use case — nothing else touches those rows at 3am. But an integration suite that creates and deletes its own tenant's rows *in parallel*, against that same shared database, breaks the assumption: the function's `SELECT` can see a row that another test's transaction deletes before the function's `INSERT` commits. Because the batch runs as one atomic statement, the `INSERT`'s foreign-key check (which reads *current* state, not the `SELECT`'s snapshot) rejects that one row — and the whole statement aborts, including rows that had nothing to do with the deleted one. From the outside this looks like a cascading, intermittent failure across unrelated tests in the same file, reproducible only under full-suite load, never in isolation.

**Fixed:**
```sql
-- before: plain SELECT, no protection against a concurrent delete
select c.id, c.empresa_id, ... from contratos c where c.estado = 'Vigente' ...

-- after: lock each candidate row before inserting its dependent row
select c.id, c.empresa_id, ... from contratos c
where c.estado = 'Vigente' ...
for update of c
```
A concurrent `DELETE` on a locked row waits for this statement to finish; if the delete already committed before the lock was taken, the row simply doesn't appear in the `SELECT` at all — either way, no partial/rejected insert. No behavior change in the real, non-concurrent cron case.

**Where to look for this:** any function that scans "all rows matching X" across tenants (not scoped to one caller) and then writes something derived from each row it finds, in a codebase whose test suite creates/deletes real rows in parallel against a shared local DB. Confirm with a targeted concurrency reproduction (N rows + their deletes fired in parallel against the old function, 0 failures against the fixed one under the same load) before trusting the fix — don't assume `for update` is the right lock target without proving the specific race actually goes away.

## 10. Comparing a Postgres timestamp against a Node one, with zero tolerance, is flaky under load — not a real bug

**The mechanism:** a test asserts `updated_at >= timestampCapturedBeforeTheRequest`. In practice these come from two different clocks in two different processes (Postgres's transaction-start `now()`, Node's `Date.now()`), and the real gap between them is normally a few milliseconds — comfortably positive in isolation. Running the full suite in parallel (many files competing for CPU against the same Docker-hosted database) shrinks that margin and occasionally pushes it negative for a few milliseconds, failing the assert on a genuinely correct update.

**Fixed:** give the comparison a bounded tolerance sized to the real observed margin — not zero, and not so generous it stops catching an actual bug (a stale/null timestamp fails by seconds, or fails an earlier `toBeDefined()`, not by single-digit milliseconds):
```ts
expect(updatedAt.getTime()).toBeGreaterThan(before.getTime() - 1000); // 1s, not 0
```
Confirm it's really clock skew before widening any tolerance: reproduce under the same concurrent load that produced the original failure, and confirm the same test never fails run in isolation.

## 11. A fixed-UTC-offset country's local time depends on each Postgres's own tzdata build — don't resolve it by zone name

**The mechanism:** a named IANA zone (`'America/Asuncion'`, say) is resolved against the tzdata version compiled into that specific Postgres instance. When a country changes its rule (Paraguay fixed itself at UTC-3 year-round by law in Oct 2024, dropping DST), every Postgres still running an older tzdata build keeps computing the old rule — silently, no error, for exactly the months the old DST would have applied. A local dev container and a managed production Postgres can each run a different tzdata vintage and therefore disagree with each other, and neither necessarily matches what the app's Node process computes from its own, independently-updated tz database. This isn't a test flake — it's a real, silent wrong answer for real users during the affected months, and patching each environment's tzdata one at a time doesn't reach a managed production Postgres you don't control.

**Fixed — stop resolving by zone name for a country with a fixed, known offset; convert explicitly instead:**
```sql
-- before: depends on this Postgres's tzdata build
select (now() at time zone 'America/Asuncion')::date;

-- after: fixed offset, independent of any tzdata table
select ((now() at time zone 'UTC') - interval '3 hours')::date;
```
Applies to every function/view that derived a "local date" or "local hour" this way, not just the obvious ones — grep for the zone name across functions, views, and any report that buckets by day/hour.

**Where to look for this:** a country whose current UTC offset is fixed and known (no DST) is safe to hardcode as an interval instead of resolving by name; a country that still observes DST cannot use this fix and needs its tzdata kept current on every Postgres instance instead — check which situation actually applies before hardcoding an offset.

## 12. `overflow-x: auto` on a container silently sets `overflow-y` to `auto` too — enough to clip an absolutely-positioned child

**The mechanism:** per the CSS Overflow spec, setting only one axis of `overflow` to something other than `visible` forces the *other* axis to resolve to `auto` as well, even though the stylesheet reads as if it were untouched. A container given `overflow-x-auto` purely for horizontal scroll (a wide table on mobile, say) quietly becomes a scroll/clip container on the vertical axis too — and any `position: absolute` descendant (a filter dropdown, a popover) that would normally escape a short parent by rendering past its bottom edge gets clipped at that parent's boundary instead. The effect only becomes visible once the parent is shorter than the popover needs — e.g. a table collapsed to a single "no results" row — so it sits unnoticed through review against normal-sized data.

**Fixed — stop relying on the DOM ancestor for positioning at all:** render the popover in a portal to `document.body` with `position: fixed`, computing its position from the trigger's `getBoundingClientRect()` at open time, and close it on scroll/resize so it doesn't drift from its trigger. Click-outside detection then has to check both the trigger and the portaled panel explicitly, since they no longer share a DOM ancestor.

**Where to look for this:** any absolutely-positioned popover/dropdown/tooltip nested inside a container that sets `overflow-x` (or `-y`) to anything but `visible` for an unrelated reason (horizontal scroll on a table, say) — the clip risk exists regardless of whether the popover has actually been clipped yet, since it only shows up once content happens to be shorter than usual.

## 13. Turning a throwing gate into a returned `{ok:false}` makes it possible to ignore it

**The mechanism:** after the Server Action error audit (#1), the permission helper became `requiereModuloReportes(): Promise<{ok:true} | {ok:false; error:string}>` instead of throwing. Five report functions were written against the new shape (`const gate = await …; if (!gate.ok) return gate;`). Three older ones kept the bare line `await requiereModuloReportes();` — it still compiles, still runs, and discards the result. **The permission check was a no-op**: a user without the reports module (the collections role, say) could pull the property-owner settlement, sales-commission and zone-performance reports by invoking the action directly. The UI already blocked the screen, so nothing looked wrong, and there was no error and no type error to point at it.

**Broken:**
```ts
export async function obtenerReporteComisionesVendedor(desde: string, hasta: string): Promise<ComisionVendedor[]> {
  await requiereModuloReportes();          // result dropped — gate does nothing
  ...
}
```

**Fixed** — these three deliberately kept the throwing convention of read functions (they don't return `Resultado<T>`, so `return gate` doesn't type-check), so they throw the gate's message:
```ts
  const gate = await requiereModuloReportes();
  if (!gate.ok) throw new Error(gate.error);
```

**Where to look:** after converting *any* throwing helper into a result-returning one, audit every caller — `grep -rnE "^\s*await (requiere|assert|ensure|check)\w*\(" lib app` finds the bare-statement calls. Prefer helpers whose failure can't be ignored: throw, or return a value the caller must destructure to proceed. This is the mirror image of #1: converting throws into returned errors fixes the redaction problem and creates a new failure mode where the error is never looked at.

**Testing caveat:** a `"use server"` function that calls `cookies()` can't be imported under Vitest, so the gate can't be exercised directly — see `testing-discipline.md` ("When the unit can't be imported") for what held up.

## 14. RLS silently empties a join or an embed for the role that uses the screen most

**The mechanism:** RLS on the *joined* table filters rows without raising anything, so a `left join` or a PostgREST embed (`select("*, productos(nombre)")`) comes back NULL or empty for a role that can read the parent table but not the joined one. `security_invoker` views behave the same way — they evaluate the base tables' policies as the caller. It shows up for the least-privileged role while the owner and admin (who hold every module) never see anything wrong. Three real instances in one codebase:
- The cashier's ticket printed **without the invoice number**: `v_ventas_turno_actual` left-joined `timbrados`, whose SELECT policy was `nivel = 'owner'` → every joined column NULL for a cashier.
- The salesperson saw an **empty product selector** on the sales screen: the list read `productos_comercio` directly, whose policy requires the `productos` module, which the seller role doesn't have.
- The warranty list showed **every product name blank** for the same role: an embedded `productos_comercio(nombre)`.

**Two fixes, chosen by how sensitive the joined table is:**
```sql
-- Not sensitive (invoice number): widen the policy to the module that needs it
drop policy "timbrados_select_owner" on public.timbrados;
create policy "timbrados_select_modulo_comanda" on public.timbrados
  for select using (
    empresa_id = public.current_empresa_id()
    and (public.current_nivel() = 'owner' or public.current_tiene_modulo('comanda'))
  );

-- Sensitive columns on the same table (cost): do NOT widen the policy. Add a narrow
-- security-definer RPC gated on either module, returning only the safe columns.
create or replace function public.listar_productos_venta()
returns table (id uuid, nombre text, precio_venta numeric)
language plpgsql security definer set search_path = public as $$
begin
  if not (public.current_tiene_modulo('ventas') or public.current_tiene_modulo('productos')) then
    raise exception 'No tenés permiso para ver el catálogo de venta';
  end if;

  return query
  select p.id, p.nombre, p.precio_venta      -- never costo_promedio
  from public.productos_comercio p
  where p.empresa_id = public.current_empresa_id()
  order by p.nombre;
end;
$$;
```
Widening `productos_comercio`'s policy would have handed the seller `costo_promedio`; a `security_invoker` view doesn't help because it still evaluates the base policy. Resolve names by matching on `producto_id` in a plain module (no `"use server"`, client passed in) so it's also testable.

**Where to look:** for each screen, take its *least*-privileged role and list every table the screen joins or embeds; check that role's SELECT policy on each. Test by logging in as that role with data present — an owner-only test passes on all three of these. Put "each role that uses this screen sees non-empty data" in the plan's review-focus list.

## 15. RLS filters rows, not columns — protect a sensitive column with column-level grants

**The mechanism:** Postgres gives `authenticated` table-level privileges on every column, and RLS only decides *which rows*. A role that legitimately has access to a row can read and write every column of it through PostgREST with its own token, bypassing whatever your app's queries choose to select. Two real cases:
- A seller could read `ventas_detalle.costo_unitario_snapshot` (the frozen unit cost). Round 2 removed the embed from the app query — that closed only the *app* side; a direct API call still returned it. Round 3 closed the database side.
- The warehouse role could run `update productos_comercio set costo_promedio = 1` directly. That column is meant to be written only by the weighted-average purchase RPC; a hand-written value corrupts the cost frozen into every future sale and the profitability reports.

**Fixed — write side:**
```sql
revoke insert, update on public.productos_comercio from anon, authenticated;
grant insert (empresa_id, marca_id, categoria_id, nombre, precio_venta, stock_minimo),
      update (marca_id, categoria_id, nombre, precio_venta, stock_minimo)
  on public.productos_comercio to authenticated;
-- costo_promedio is now writable only by security-definer RPCs (registrar_compra) and service_role
```
**Fixed — read side:** `revoke select on public.ventas_detalle from anon, authenticated; grant select (id, venta_id, producto_id, cantidad, precio_unitario) on public.ventas_detalle to authenticated;` — `costo_unitario_snapshot` is left out on purpose.

**Consequences to expect** (they're the intended behavior, not regressions):
- `security definer` RPCs and the service role are unaffected — they don't run as `authenticated`.
- A direct `UPDATE` on a table with no update policy used to affect 0 rows silently; now it fails with `42501`. Adjust tests that asserted "0 rows".
- After a read-side revoke, `select("*")` from the client fails with `42501` because it expands to columns the role can't read — list the columns explicitly.
- The grant is a whitelist: a column added later has no grant until a migration adds it. An unexpected "permission denied" on a brand-new column usually means this.

**Verify against the API surface, not the app:** call PostgREST with that role's own token and expect `42501`; then revert the migration and confirm those tests fail. **Where to look:** any column an RPC is supposed to own — cost averages, balances, credit limits, stock levels, totals, audit stamps. If a role can `update` the table, it can write that column.

## 16. Adding a prop to a shared component: make it required, or callers you didn't know about keep the old behavior

`AnularVentaModal` gained `formaPago` so the refund option could say "no afecta la caja" for card and transfer sales. It was added as *optional*, to avoid breaking a second caller (`HistorialVentasTab`, in Reports) that the brief hadn't mentioned. That caller silently kept showing "genera un egreso de caja por el total de la venta" for every sale — exactly the misleading text the change existed to remove. A reviewer found it.

**Fix:** pass the value in the second caller (the data was already in the row object), and change the prop from optional to **required** — `formaPago: string | null`, required but nullable — so the compiler lists every caller and a future one has to decide. Rule: before changing a shared component's contract, `grep` all importers; use a required prop rather than an optional one with a legacy default whenever that default *is* the behavior you're removing.
