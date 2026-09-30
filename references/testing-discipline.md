# Testing discipline

Adapted from the `test-driven-development` skill and its `writing-good-tests` reference in [obra/superpowers](https://github.com/obra/superpowers) (MIT). Read this before writing or changing a test, and whenever you fix a bug.

## How this fits with "scale verification to blast radius"

`SKILL.md` says to match the *breadth* of verification (unit only, integration, full e2e, visual check) to what changed. That's about which suites you run. It does not relax the rule below, which is about how each piece of logic gets its test: **for logic you write or fix, the test comes first and you watch it fail.** A pure CSS/copy change has no logic to test-drive; a Server Action, RPC, permission check or money calculation always does.

## Red → green → refactor

1. **Red.** Write one minimal test for one behavior, with a name that says what should happen. Use real code; mock only what's slow or external.
2. **Verify red — mandatory.** Run it. It must *fail* (not error), for the expected reason (the behavior is missing, not a typo or import error). If it passes immediately, you're testing behavior that already exists — fix the test.
3. **Green.** Write the simplest code that passes. No extra options, no "while I'm here". YAGNI.
4. **Verify green — mandatory.** The new test passes, **the project's whole suite** passes, output is clean. A green run of your one file is not a green suite. Report any failure you see by name, including ones you didn't cause.
5. **Refactor** only while green: remove duplication, improve names, extract helpers. Don't add behavior.

If you wrote implementation code before its test, the honest options are to delete it and restart test-first, or to tell the user explicitly that this piece was tested after the fact. Don't quietly bolt tests onto it and call it TDD — a test you never saw fail hasn't proven it can catch anything.

**Bug fixes always start with a failing test that reproduces the bug.** For a regression test, prove it: revert the fix → test fails; restore → test passes.

## Writing tests that actually catch things

**Name the break.** Before writing the body, answer: *what production change would make this test fail — and is that change a bug or a decision?* Can't name one → redesign the test around observable behavior.

**Derive expectations independently.** Use literals and hand-checked fixtures. An expected value computed by the code under test (or its helpers) passes no matter what:

```ts
// ❌ Mirror assertion — always true
const expected = calcTotal(order);
expect(calcTotal(order)).toBe(expected);

// ✅ Hand-derived literal
expect(calcTotal({ items: [{ price: 1200, qty: 2 }], tip: 300 })).toBe(2700);
```

**No change detectors.** A test that only fails when you intentionally change a constant or a message wording fires on redesign and sleeps through bugs. Not `expect(MAX_RETRIES).toBe(5)` — test "a failing call is retried 5 times and the 6th never happens."

**Your code, not the framework.** Test the contract at your boundaries — the payload your Server Action returns (`{ok, error}`), the rows your RPC writes, what a user of role X can and can't see. Don't test that Next.js calls your handler or that Supabase executes SQL.

**Assert real behavior, never mock behavior.** If the assertion is about what the mock did, you've tested the mock. Understand a dependency's side effects before mocking it; keep test-only helpers in test utilities, never in production modules.

**Multi-tenant specifics worth a test every time:**
- Tenant A cannot read or write tenant B's rows (RLS), from each role.
- A lower role can't reach an action the UI merely hides from it.
- The error path of every Server Action returns a typed error instead of throwing (see gotchas #1).
- Every role that uses a screen sees non-empty data on it — RLS silently empties joins and embeds for the role that isn't owner/admin (gotchas #14). An owner-only test passes on all of those bugs.
- A sensitive column can't be read or written by a role that has the row but not the column, **through the API directly** with that role's own token, not only through your app's query (gotchas #15).

## Tests that looked fine and weren't (lessons from review rounds)

Each of these is a real gap that review found in a test suite that was otherwise green — the kind of test that keeps passing after someone reverts the fix.

**The test re-implemented the code it was testing.** A test for the installment calculation recomputed `precio / cuotas` inside itself and compared the result to itself; neither the form nor the server action was imported anywhere, so reverting the fix wouldn't have failed it. Fix: extract the calculation to a pure function in a module with no `"use server"` (`calcularMontoCuota()` in `lib/formato.ts`), imported by the component *and* the test, with hand-derived literals as expectations (60,000,000 with a 10,000,000 down payment in 10 installments → 5,000,000, not 6,000,000).

**When the unit can't be imported, the extraction must carry the assertion target with it.** A `"use server"` function that calls `cookies()` or React's `cache()` can't be imported under Vitest's node environment. The round-2 attempt extracted the report functions into plain "flat" functions so they could be called with a Supabase client — but the permission gate lived in the `"use server"` wrappers, not in the extracted bodies, so the new tests covered everything except the thing being fixed; the extraction also broke the build (a component still imported the types from the old file). It was introduced in round 2 and reverted in round 3. What held up, in order of preference: (1) extract *pure logic* and test that; (2) test the underlying RPC/policy with a real authenticated session; (3) as a last resort a **structural** test that reads the source and asserts the guard is still there (each of the three function bodies contains `if (!gate.ok)`; the form's `input` object uses `monto_mensual: montoCuota`). Name it as structural in the test title — it catches a revert, not behavior — and never present it as behavioral coverage.

**A concurrency test must let only the lock under test serialize the calls.** A test claimed to isolate the customer-row lock between two concurrent sales, but both sales used the same product row, which is locked *earlier* in the function — so they serialized there and the test stayed green even with the customer lock deleted. Use disjoint fixtures (two different products), then remove the lock under test and confirm the test fails. Assert the invariant rather than who wins (`money-and-ledger-integrity.md` #2), and run it several times in isolation to rule out flakiness before trusting it.

**Prove red against the old definition.** For SQL fixes, apply the old function body (or revert the migration) in the local database, run the new test, and watch it fail; only then restore. Several fixes here recorded "N of M cases fail without the migration"; the cases that still pass without the fix are controls (they pin the behavior that must *not* change), so report both numbers.

**Test every object a migration touches.** One migration fixed a function and two views; the first commit's test exercised only the function. List the objects in the migration header, then check each has a test.

**Test through the surface an attacker or another role sees.** For permission and column-privilege fixes, call PostgREST/RPC with the role's own token and assert the `42501` — an app-level test passes even while the direct API still leaks.

## When testing is hard

| Problem | What it's telling you |
|---|---|
| Don't know how to test it | Write the API you wish existed, then the assertion first. |
| Test is complicated | The design is complicated. Simplify the interface. |
| Must mock everything | Code is too coupled. Inject dependencies. |
| Setup is huge | Extract helpers; if it's still huge, simplify the design. |

## Rationalizations

| Excuse | Reality |
|---|---|
| "Too simple to test" | Simple code breaks; the test takes 30 seconds. |
| "I'll test after" | Tests written after pass immediately — which proves nothing. |
| "Already tested it manually" | Manual checks leave no record and can't be re-run on the next change. |
| "Deleting this work is wasteful" | Sunk cost. Keeping code you can't trust is the waste. |
| "Existing code has no tests" | You're touching it — add tests for the part you touch. |
