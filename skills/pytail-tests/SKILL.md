---
name: pytail-tests
description: >
  The over-TESTING counterpart to pytail-review: review this branch's test
  changes (or a test file you point at) and list which tests to CUT — tests for
  deleted code, coverage already proven cheaper elsewhere, tests of the
  standard library's / pydantic's / a framework's own behavior, tautological
  mock-asserts-mock, near-identical cases parametrize would merge, and "it no
  longer does X" tests. One line per finding: location, why it earns nothing,
  what (if anything) replaces it. Use when the user says "do I need these
  tests", "cut unnecessary tests", "are these over-tested", "which tests can I
  delete", "pytail tests", or invokes /pytail-tests. Lists findings, applies
  nothing.
---

Review tests for over-testing. A test earns its place ONLY if it would fail when
a real behavior breaks AND that failure isn't already caught elsewhere. If
deleting it loses no failure signal, cut it. The suite's best outcome: fewer,
sharper tests that still go red when the code is wrong.

## Scope

Default to this branch's test changes vs its parent. Resolve the base branch,
never assume it:

`git symbolic-ref --short refs/remotes/origin/HEAD`  # → e.g. origin/main

If that fails (no remote HEAD set), take the first of `develop`, `main`,
`master` that `git rev-parse --verify` finds. Then:

`git diff <base>...HEAD -- tests/ test/`                          # test changes in this branch
`git diff --name-only <base>...HEAD | grep -E '(^|/)test_.*\.py$|_test\.py$'`  # just the test files touched

If the user names a base, use theirs. If the user points at a
test file (or the tests covering a file they just changed), review those instead.

## Discover once, then verify

Scan the scoped test changes once, collect candidates, merge overlapping cuts,
and rank biggest worthwhile cut first. Treat tags and examples below as leads,
not sufficient evidence to delete a test.

Work down the candidate list and try to **disprove each proposed cut** using
targeted reads of the test, its fixtures and conftest.py, the implementation, and
any claimed surviving coverage:

- What concrete regression could this test catch that the survivor would miss?
  Compare assertions, inputs, edge cases, and exercised code paths. Name the
  exact surviving test when claiming duplicate coverage.
- Does a lower-layer test actually prove the integration behavior? A unit test
  of a permission check does not prove the endpoint or handler enforces it.
- Does a framework-looking assertion protect application configuration, or a
  negative assertion prevent a real regression? Does a mock check a required
  interaction? Labels alone do not justify cutting these tests.
- Would cheaper setup still exercise the behavior, including persistence,
  queries, I/O and constraints? Do not suggest an in-memory fake for a behavior
  that requires the real database or network contract.

Reject cuts that lose a distinct failure signal or whose safety is uncertain.
For accepted cuts, record why the failure signal survives and where. Run the
relevant surviving tests with the repository's runner when available; a green
run alone does not prove redundancy. If execution is unavailable, disclose that
and rely only on evidence actually inspected. Do not modify tests during review.

By default, verify every discovered candidate and report only survivors. If the
user supplies a limit K, stop after K distinct suggestions survive or the list
is exhausted, moving past rejected candidates without padding. Before reporting,
check the retained cuts together: they must not remove each other's surviving
coverage. Drop conflicting cuts and backfill from remaining candidates if needed.
Score only reported, verified cuts. Do not rescan the suite or loop for agreement;
state missing evidence rather than treating an incomplete review as clean.

## Large reviews

Review directly unless at least 12 test files split into two or more coherent
feature areas. Then, if subagents are available:

1. Keep one coordinator for cross-suite coverage and give each of at most three
   subagents a disjoint feature group, the resolved base, and this rubric. Keep
   shared integration and end-to-end tests with the coordinator.
2. Have subagents return candidate findings only for their assigned diffs.
3. Apply the targeted verification workflow, merge repeated coverage findings,
   and calculate one net score for reported cuts. Never cut the last test that proves behavior across all groups.

If the tests are tightly coupled, do not delegate. Parallel review reduces
elapsed time; it never weakens the failure signal or multiplies findings.

## Tags (what to cut, and why it earns nothing)

- `delete:` a test for code that no longer exists, a `@pytest.mark.skip`/`xfail` that never runs, or an "it no longer does X" test asserting *removed* behavior. Replacement: nothing.
- `dup:` the same behavior already proven by a cheaper or more meaningful test at another layer (a validation tested in the model AND the handler AND an end-to-end test). Keep one — name it — cut the rest.
- `framework:` tests the standard library's, pydantic's or a framework's own behavior with no custom logic (a pydantic field rejects a str for an int, a dataclass compares equal, `json.dumps` round-trips). Cut — don't test the library.
- `mock:` tautological — patches a collaborator, then asserts `assert_called_once_with` on the mock it just set up; it tests the double, not behavior. Assert the real result or state instead, or cut.
- `shrink:` same coverage, cheaper: near-identical test functions → one `@pytest.mark.parametrize`; a heavy fixture where a literal suffices; `unittest.mock` where a tiny fake (`httpx.MockTransport`, a stub object) is clearer. Merge/downgrade.

## Format

`<file>:L<line>: <tag> <what>. <keep instead / nothing>.` Rank biggest cut first.

## Examples

✅ `tests/test_models.py:L12: framework: asserts pydantic rejects "abc" for an int field — library behavior, no custom validator. Cut.`

✅ `tests/test_api.py:L40: dup: re-asserts the 403 already covered by test_permissions.py::test_denies_analyst. Keep that, cut here.`

✅ `tests/test_orders.py:L22: mock: patches price_order then asserts it was called. Asserts the double — assert the saved order total instead, or cut.`

✅ `tests/test_email.py:L8: delete: "no longer sends the legacy digest" — asserts removed behavior. Cut.`

✅ `tests/test_parse.py:L10-60: shrink: five copies of one test with different inputs. One @pytest.mark.parametrize.`

## Scoring

End with `net: -<N> tests, -<M> test files possible.`

Nothing to cut: `Coverage is earning its keep. Ship.`

## Boundaries

Cut over-testing ONLY. NEVER cut the coverage of real behavior, an edge case, a
money/auth/security path, or a regression tied to a ticket — and never delete the LAST
test that would fail if the logic breaks (the pytail minimum is sacred, not
bloat). When unsure whether a test catches a real failure, KEEP it and say so —
a wrongly-deleted test is a silent regression waiting to happen. Verify survivors as described above. Lists findings, applies nothing.
"stop pytail-tests" or "normal mode" to revert.
