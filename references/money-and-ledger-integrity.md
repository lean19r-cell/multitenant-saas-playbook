# Money, stock and ledger integrity in Postgres RPCs

Read this when you touch anything that moves money or inventory (cash register and shift close, sales, voids and refunds, payments, customer balances, stock, installments), when you write a migration that redefines a function or view, or right before you call a fix to a "the numbers don't add up" report done.

**Provenance, so you can calibrate trust.** These came from a full-app audit ("Tanda 1: que no se pierda ni se invente plata", Sept 2026) and from the whole-branch review of the third vertical (a retail/wholesale POS with customer credit accounts). They were found by audit and review, then reproduced against the real schema in a local database — not reported by users. What makes them worth a file of their own: most of these fixes were **incomplete the first time**. The permission-gate fix took three rounds; the cash-close fix introduced a new bug in round two. Section 8 is the checklist that would have caught those in round one.

## 1. A void flag must be honored by every reader — and by the compensating entry it created

**The mechanism.** Voiding a sale does two things: it sets `ventas.anulada = true`, and for a *refund* (`es_reintegro`) it also inserts a real cash-out row in `movimientos_caja`, with `venta_id` pointing back. Every place that aggregates the sales table is a separate reader that has to know about `anulada`: the shift-close function, the live-balance view, and — found later by the audit — three stock/consumption objects that read `venta_items` directly and never joined the parent sale. Original symptom: the close counted voided sales as cash, so every void produced a false shortage.

**The trap in the fix.** The obvious patch — `where not anulada` on the sales branch — is right for a *correction*-type void (no cash moved) and wrong for a *refund*-type void: the sale (+X) leaves the total but the refund egreso (−X) is still subtracted, so the close is off by −X in the other direction — a **false surplus**, which is worse than the original bug because it can hide a real shortage of the same amount. The rule that survived review: exclude a voided sale only if it has no compensating egreso; keep counting it (the egreso cancels it) if it does.

```sql
-- before (round 1): correct for corrections, double-subtracts refunds
select total, forma_pago, turno_caja_id from public.ventas where not anulada

-- after (round 2)
select vr.total, vr.forma_pago, vr.turno_caja_id
from public.ventas vr
where not vr.anulada
   or exists (select 1 from public.movimientos_caja m
              where m.venta_id = vr.id and m.tipo = 'egreso')
```

Detect the link through a table **both readers can see the same way**. The close function is `security definer`; the live view is `security_invoker`. The alternative — checking `anulaciones_venta.es_reintegro` — was rejected on purpose: that table is readable only with the reports module, so a cashier's live number would silently differ from what the close computes. `movimientos_caja` is the row the egreso branch already sums, so the two objects can't disagree.

**A refund follows the original payment method.** A card or transfer sale never put physical cash in the drawer. A refund that inserts a cash egreso for it removes money that never entered (false shortage) — and the same RPC demanded an *open shift* just to void a card sale. Fix: the egreso, and the open-shift requirement, apply only when the original `forma_pago` was cash; the void is still recorded with `es_reintegro = true` as evidence that a refund is owed outside the system. The UI copy has to say which case it is — the modal claimed "genera un egreso de caja" for card sales too.

**Where to look.** For every table that carries a void/status flag, list *all* its readers — `grep -rn "from public.ventas\|venta_items" supabase/migrations`, taking the latest definition of each — and decide per reader. Test the matrix (void type correction/refund × payment method cash/card), not one happy path. And test **every object the migration touches**: one commit fixed a function and two views and its test covered only the function; the views got tests in a follow-up.

## 2. Two concurrency bugs in one RPC family: a lookup without `for update`, and an inconsistent lock order

**Missing lock.** `confirmar_pedido()` found the open shift with a plain `select`, while its siblings `cobrar_pedido()` and `cerrar_caja()` used `for update`. A sale racing a close could be inserted with `turno_caja_id` pointing at a shift that had *just* been closed — after the close computed `monto_esperado` — so real cash sold vanished from that close, with no error anywhere. The fix is one line. Rule: an RPC that reads a "container" row (shift, period, batch, open order) and then writes children that reference it must lock the container whenever another RPC can close it.

**Lock order.** `anular_venta()` locked sale → stock rows → shift; `confirmar_pedido()` locked shift → invoice sequence → stock rows. A cash refund and a new order sharing one ingredient could deadlock (`40P01`): no money lost, but one transaction aborts with an error the user can't interpret. Fix: a pure reorder, sale → shift → stock, matching every other RPC. This one was invisible to per-task review — the two functions belonged to different tasks — and surfaced only in the whole-branch review (see `review-and-agents.md`).

Keep the lock order written down for the schema (a table in docs, or a header comment in each migration), and re-check it whenever an RPC gains a `for update`. To prove a reorder-only migration is reorder-only, compare the multiset of statements old vs new and confirm the live body (`pg_proc.prosrc`) equals the file after applying.

**Testing it.** Two real authenticated sessions, `Promise.all` (never two sequential calls dressed up as concurrent), N iterations each with fresh fixtures. Assert the **invariant**, not who wins — here, "the close's `monto_esperado` equals fondo + the sales that actually committed against that shift"; both orderings are valid, only "sale committed but missing from the close" is not. Then run the same test against the *old* function body and watch it fail (the RES-07 test failed on iteration one without the fix and 0 of 20 with it). See `testing-discipline.md` for making sure only the lock under test can serialize the calls.

## 3. "Eliminar" plus `on delete cascade` on a history FK erases history silently

**The mechanism.** `venta_items.producto_id` and `ventas.usuario_id` were `on delete cascade`, and the Productos/Insumos screens had an "Eliminar" button with no confirmation. Deleting a product deleted every past sale line that used it: sale totals no longer matched their lines, any report built from the lines silently lost those sales, and nothing errored.

**Fix, in three parts.**
1. FKs from history/ledger tables to catalogs and users become `on delete restrict`.
2. `activo boolean not null default true` plus "Desactivar / Reactivar" with a confirm dialog that says what is kept ("se conserva su historial de ventas").
3. Filter `activo = true` **only where the user is choosing something for a new operation** — the sale grid, the "add ingredient to a recipe" picker, the "load stock" picker. Never in the admin list (it must keep showing inactive items, dimmed, or nobody can reactivate them), never in reports, never in the detail of a sale that already happened.

**The miss.** The task brief named two pickers; review found a third ("Cargar Stock" let you record a stock entry against a deactivated ingredient). Inventory every `select` from the table and classify each as "choosing for something new" vs "looking at something that happened". When one query feeds both a table that must show inactive rows *and* a picker (`listarInsumosConStock()` did), filter at the picker, not in the query.

**Where to look.** `grep -rn "on delete cascade" supabase/migrations` for FKs whose parent is a catalog/user and whose child is history; `.delete()` calls against catalog tables in `lib/`.

## 4. Numeric parameters that slip past `> 0`

- **`NaN`.** Postgres accepts `'NaN'::numeric` even inside `numeric(12,2)` and orders it *above* every number: `NaN >= 0` is true and `x > NaN` is false. So `check (col >= 0)` doesn't stop it, and once a customer balance is NaN the "can't overpay" test (`monto > saldo`) is false forever — the balance never becomes a number again. It got in through direct REST writes and through RPC parameters. One `check (col <> 'NaN')` per numeric column closes every write path at once — including ones you haven't written yet (take the column list from `information_schema.columns`, not from memory — it was 22 columns across 11 tables, including two shared with another vertical). `Infinity` is already rejected by the typmod; `NULL` passes the check, so nullable columns behave as before. `add constraint` validates existing rows, so run `select count(*) … where col = 'NaN'` against production *before* the push and leave that query in the migration comment.
- **Round, then validate.** `if p_monto <= 0 then raise …; v_monto := round(p_monto, 2);` lets `0.001` through; it rounds to `0.00` and dies later on a raw `check (monto > 0)` violation instead of the business message. Round first, validate the rounded value, and use only the rounded value from then on.
- **Don't silently coerce a mismatched parameter.** For non-credit sales the RPC overwrote `p_monto_pagado` with the total. A caller sending 500 for a 1000 cash sale got a fully paid sale, and the close counted cash that never entered the drawer. Reject inconsistent input with a business error; `NULL` may still mean "paid in full".
- **`array_agg(distinct x)` keeps a single `NULL`.** Two items without a `producto_id` were reported as "repeated product" instead of "missing product". Reject NULLs before the duplicate check.

## 5. A business rule must hold on every path that writes the field — and the default must be the safe value

One feature (a customer's credit limit) produced a chain of review findings, each closing a path the previous fix left open:

1. A salesperson could set or remove any customer's limit → only the administrator may, enforced inside the RPC.
2. RLS restricts rows, not columns, so a direct REST `insert`/`update` still wrote the column → column-level grants (`nextjs-supabase-gotchas.md` #15).
3. A *new* customer created without a limit got `NULL`, which means "no cap". A salesperson could create a duplicate of a limited customer (same document) and sell on credit without control — 13.5M in the reviewer's demo. The default was the unsafe value. Fix: the RPC forces `0` for non-administrators **and** the column default becomes `0`, so the direct-REST path also gets the safe value; `NULL` now exists only when an administrator writes it explicitly.
4. The UI then lied: the optimistic row showed "Sin límite" for a customer stored as `0`, and editing it immediately sent `null`, which the RPC rejected as "removing the limit". Optimistic state must mirror the server's normalization, not the raw input.

**The same shape for cash.** Any payment that lowers a debt must land inside an audited container. A cash collection with `turno_caja_id = NULL` (no branch selected, or a branch whose shift is closed) reduced the customer's debt without ever appearing in a shift close — money could leave the drawer with no difference showing. Down payments on credit sales had the same hole (no payment method, in no close) and were removed outright: the sale is 100% on credit, and payment is registered afterwards as its own record. A `check` constraint can't know whether a shift was open, so enforce it in the RPC ("cash needs a branch with an open shift") *after* the `for update` lookup of the shift — then a race with the close either sees it closed (rejected) or gets counted by the close.

**Where to look.** For each rule ("only X may set Y", "must fall inside a shift"), list every path that can write the field: RPCs, direct REST insert/update, seed scripts, batch functions, a sibling RPC that does nearly the same thing. Then ask: *what does a row created by a path that never mentions this column get?* That is the default, and it must be the restrictive value.

## 6. Fixing a wrong-unit field changes what its readers mean — and a default that reproduces the bug is not a fix

The reservation form sent the property's *total price* as `monto_mensual`, so a 60,000,000 sale in 60 installments billed 60,000,000 every month. The fix separated "Precio total" from "Cantidad de cuotas" and sent `total / cuotas`. Four things the first pass missed:

- **The default reproduced the bug.** `cantidadCuotas` started at `"1"`, so a salesperson who didn't touch the field still sent price ÷ 1. It must start empty and be validated. (The monthly generator counts months from the date range and never reads this field, which is why a wrong number kept being billed every month.)
- **Readers of the field changed meaning.** After the fix `monto_mensual` is per installment, so the generated *boleto de compraventa* — which printed it under "Precio:" — now put a false price on a legal document (before, it had been right by accident). Grep every consumer of a field whose meaning you change: generated documents, reports, exports, labels. Where the true value doesn't exist in the model yet, use a qualified label rather than inventing the number.
- **Business rules hide in the arithmetic.** The down payment is *part of* the total, not on top of it (a rule the user had to confirm): 60M with 10M down in 10 installments is 5M each, not 6M. Ask before assuming.
- **Put the calculation in one pure function** (no `"use server"`, e.g. `lib/formato.ts`) imported by the component *and* the test — otherwise the test re-implements the formula and compares it to itself (`testing-discipline.md`).

## 7. `create or replace` from a stale base silently reverts later changes

A migration that redefines a function or view is a full copy of its body. Copy it from the *first* migration that created it — or from an audit written days earlier — and you drop every change made since. Here the cash-close function had been redefined by the new vertical between the audit and its fix: it now also summed `ventas_comercio` and customer payments. Starting from the older body would have removed those branches and broken the register of the other vertical, with no error at migration time.

- Take the base from the live definition (`select pg_get_functiondef('public.f(args)'::regprocedure)`, or `pg_proc.prosrc`, in a local DB with every migration applied), diff your body against it, and check that the *only* differences are the intended ones.
- After applying, confirm the live body equals the migration file.
- A plan or audit that cites line numbers or definitions ages in days when parallel branches are merging — re-verify against current code before executing it.
- Same family, different trap: never `create or replace` with a different parameter list (`nextjs-supabase-gotchas.md` #2).

## 8. Before calling a money / stock / ledger fix done

Most of the nine Tanda 1 fixes needed at least a second review round; the permission-gate one needed three. Every second-round finding was one of these:

1. **Other readers of the same table?** Grep every view, function and report that reads it — voided rows, inactive rows, the same join, the same timezone handling.
2. **Compensating entries?** If the flagged row has a paired row (refund egreso, restock, reversal), does your filter drop both or neither?
3. **Other callers?** Every caller of the changed component, function or RPC — including the second screen the brief didn't mention.
4. **Other write paths?** Direct REST, sibling RPCs, seeds, batch functions — and what default do they get?
5. **Changed meaning?** Readers of any field whose meaning your fix changed (documents, reports, labels).
6. **Locks?** Does the new or changed RPC lock the same rows, in the same order, as its siblings?
7. **Base?** Did you start from the live definition, not from the first migration or an old report?
8. **Does the test fail without the fix, and does it exercise the real function** — not a re-implementation, not a mock of the thing under test? (`testing-discipline.md`)
