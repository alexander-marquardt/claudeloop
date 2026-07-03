---
id: suppressed-failures
label: "Suppressed Failures & Silenced Errors"
---

Find places where a failure signal is silenced instead of surfaced — a test that is skipped, a type error that is suppressed, a lint rule that is disabled, or an exception that is swallowed — and either restore the signal or attach an explicit, specific justification. A suppression hides a bug the moment the underlying condition changes; the whole point of the signal is to fail loudly when it should.

Scope this to suppressions **introduced or present in this run's changes** (the diff against the scratch-branch base). A suppression that predates the run and is untouched is out of scope unless a change in this run made it newly wrong. Do not sweep the entire pre-existing codebase.

Look for these patterns:

1. **Skipped / disabled tests** — `@pytest.mark.skip`, `@pytest.mark.xfail`, `pytest.skip(...)`, `@unittest.skip`, `it.skip` / `describe.skip` / `xit` / `xdescribe`, `test.skip`, `it.only` / `describe.only` / `fit` / `fdescribe` (which silently *stop the rest of the file's tests from running*), Go's `t.Skip()`, JUnit `@Disabled`/`@Ignore`. A skipped test is coverage that looks present in the count but exercises nothing.

2. **Suppressed type errors** — `# type: ignore` (especially bare, without a specific error code), `# mypy: ignore-errors`, `cast(...)` used to launder an incompatible type, `@ts-ignore`, `@ts-expect-error`, `@ts-nocheck`, `as any` / `as unknown as T`, `!` non-null assertions added to make the checker stop complaining. These turn a compile-time guarantee off exactly where the type is uncertain.

3. **Disabled lint / analysis rules** — `# noqa` (especially bare), `# pylint: disable=...`, `// eslint-disable` / `eslint-disable-next-line`, `# flake8: noqa`, `//nolint`, `# noinspection`, Ruff `# ruff: noqa`, an inline rule turned off for a whole file. A disabled rule is a check the team decided mattered, turned off for this one spot.

4. **Swallowed exceptions** — `except Exception: pass`, `except: pass`, a bare `except` that logs nothing and re-raises nothing, `catch (e) {}` / `.catch(() => {})`, Go `_ = err` or `if err != nil { }` with an empty body, a `try` whose `except`/`catch` discards the error and returns a default. The failure happened; the code pretended it didn't.

5. **Weakened assertions to force green** — an assertion changed from a specific expected value to a permissive one (`assert result` instead of `assert result == expected`), a tolerance widened, a golden/snapshot regenerated to match new (possibly wrong) output, a `mock`/stub/`freeze`/lenient comparison inserted so a test no longer exercises the real path. Editing a *passing* test to keep it passing after a behavior change often pins the bug rather than catching it.

For each suppression found:

- **First try to remove it and fix the underlying cause.** If a test was skipped because it failed, make the source correct so the test passes — do not delete the test. If a `type: ignore` hides a real type mismatch, fix the types. If an exception was swallowed, handle it or let it propagate. Removing the suppression and fixing the root cause is always the preferred outcome.

- **If the suppression is legitimately necessary, make it specific and justified.** A suppression is acceptable only when it is (a) narrow (a specific error code, a single line, not a whole file), and (b) carries an inline comment stating *why* — the concrete reason and, where relevant, the condition under which it can be removed. Turn `# type: ignore` into `# type: ignore[attr-defined]  # <lib> ships no stubs; tracked in <ref>`. Turn `except Exception: pass` into a narrow `except SpecificError:` that logs. Turn a bare `pytest.skip()` into `pytest.skip("requires GPU; unavailable in CI")` — and prefer a conditional `skipif` that runs whenever the dependency *is* present, so the test isn't dead everywhere. An unexplained suppression is treated as a finding even if it turns out to be harmless.

Do NOT flag these legitimate uses:

- A `skipif`/`sk_unless` that runs the test whenever its declared dependency (a service, a platform, an optional package) is available and skips only when it genuinely cannot run — this is graceful degradation, not silencing, provided the condition is specific (not a blanket always-skip).
- A `@ts-expect-error` / `# type: ignore` inside a **test** that deliberately passes an invalid value to verify the code rejects it — the suppression there is the mechanism of the test. (But a `type: ignore` used to pass an invalid value into a test of a path that *cannot actually receive that value in production* is coverage-chasing — flag it, consistent with `cleanup-ai-slop`.)
- A narrow, already-justified suppression with a specific code and a real reason comment that this run did not introduce or invalidate.
- A broad `except Exception` at a top-level daemon/worker/request boundary that logs the error with context and keeps the process alive by design — that is a deliberate resilience seam, not a swallowed failure. The swallow is only a defect when the error is discarded *silently* (no log, no metric, no re-raise) or caught far from where it can be handled meaningfully.

If a suppression genuinely must stay and you cannot fix the root cause within this check's scope (e.g. it hides a real bug that needs a design change), leave it in place, add the specific justification comment, and surface it in your final report as `unresolved suppression: <file:line> — <why it can't be removed yet>` rather than quietly leaving it bare. Run the test suite after any changes; a suppression you removed must leave the suite green (by a real fix, not a new suppression elsewhere).
