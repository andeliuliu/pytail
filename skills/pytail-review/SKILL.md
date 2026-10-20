---
name: pytail-review
description: >
  Code review focused exclusively on over-engineering in Python code. Scans this
  branch's diff vs its parent (or a file/staged changes you point at) and
  returns a ranked list of what to delete, simplify, or replace with existing
  repo code / an installed dependency / the standard library. One line per
  finding: location, what to cut, what replaces it. Use when the user says
  "review for over-engineering", "audit this branch", "what can we/I delete",
  "find bloat", "is this over-engineered", "simplify review", or invokes
  /pytail-review. Complements correctness-focused review — this one only hunts
  complexity. Lists findings, applies nothing.
---

Review this branch's diff for unnecessary complexity, ranked biggest cut first.
One line per finding: location, what to cut, what replaces it. The diff's best
outcome is getting shorter.

## Scope

Default to this branch's diff vs its parent — not the whole repo, not just
uncommitted work. Resolve the base branch, never assume it:

`git symbolic-ref --short refs/remotes/origin/HEAD`  # → e.g. origin/main

If that fails (no remote HEAD set), take the first of `develop`, `main`,
`master` that `git rev-parse --verify` finds. Then:

`git diff <base>...HEAD`              # the branch's full diff
`git diff --name-only <base>...HEAD`  # just the files it touched

If the user names a base, use theirs. If the user points
at specific staged/working changes or a single file, review that instead. Read a
changed file in full when a finding needs context, but every finding must live
inside the reviewed diff — don't flag pre-existing code it didn't touch.

## Discover once, then verify

Scan the scoped diff once to collect candidates. Deduplicate by root cause and
rank biggest worthwhile cut first; prefer better-supported, lower-risk cuts for
ties. Keep the location, proposed replacement, and expected benefit with each
candidate. This list is provisional, not the final findings.

Then work down that list, trying to **disprove each suggestion** with targeted
reads of the relevant code, callers, and tests:

- What behavior would break or disappear if this were removed or shortened?
  Check actual usage, contracts, edge cases, and framework/dynamic entry points;
  a missing text match alone does not prove code is dead.
- Does the named replacement exist in this repository or installed version,
  and does it support this use case, including inputs, outputs, and side effects?
- Does the change actually reduce complexity without moving it elsewhere or
  overlapping another suggestion? Keep one actionable cut per root cause.

Reject unsupported or behavior-changing cuts. If a candidate needs correction,
check the corrected proposal against the same evidence before retaining it.
Use a focused existing test only when it can resolve a concrete uncertainty;
do not edit code to prove a review suggestion. Reuse evidence already read.

By default, verify all discovered candidates and report only survivors. If the
user or calling skill supplies an output limit K, verify in rank order until K
suggestions survive or candidates run out; rejected candidates do not consume
slots. Do not pad the result or verify the remainder just to count it. Review
retained suggestions together for conflicting cuts before finalizing; backfill
from remaining candidates if needed.

Do not repeat full-diff discovery or loop until the list stabilizes. Score only
reported, verified suggestions. If missing context prevents verification, state
that limitation instead of presenting an unverified candidate or a clean bill.

## Large diffs

Review directly unless the diff touches at least 12 files and splits into two or
more coherent areas. Then, if subagents are available:

1. Keep one coordinator. Group changed files by feature or layer, not arbitrary
   file counts, and give each of at most three subagents a disjoint group, the
   resolved base, and this skill's rubric.
2. Have subagents review only their assigned diff and return candidate findings
   in the normal format. Keep shared entry points and cross-cutting changes with
   the coordinator.
3. Apply the targeted verification workflow before including any candidate. Merge overlapping findings, prefer one root-cause cut over repeated
   per-file cuts, rank once, and calculate one net score.

If the files are tightly coupled or cannot form two meaningful groups, do not
delegate. Subagents reduce elapsed time; they never expand scope or multiply
findings.

## Tags (mirror pytail's ladder, highest rung first)

- `delete:` dead code, unused flexibility, speculative feature. Replacement: nothing.
- `reuse:` hand-rolled logic an existing function, model, client or helper in this repo already does. Name it. (Rung 2)
- `stdlib:` hand-rolled thing the standard library ships. Name the module and function. (Rung 3)
- `dep:` custom code doing what an installed dependency ships (pydantic validation, httpx/requests retries and timeouts, a framework feature), or a new dependency for what's already covered. Name the feature. (Rung 4)
- `yagni:` abstraction with one implementation (ABC/Protocol with one subclass, factory for one product), a class that should be a function, config nobody sets, a layer with one caller.
- `shrink:` same logic, fewer lines. Show the shorter form.

## Hunt

Packages the stdlib already ships (`pytz`/`python-dateutil` for `zoneinfo`,
`requests` added beside an installed `httpx`, `simplejson`); classes with one
method plus `__init__`; `Manager`/`Helper`/`Service` wrappers around one call;
getters and setters where a `dataclass` or pydantic model fits; `ABC`/`Protocol`
with one implementation; hand-rolled retry loops when the client has retries;
manual `dict`/`list` building a comprehension does; `os.path` string juggling
where `pathlib` is shorter; `try/except: pass` swallowing errors; `async` with
nothing awaited concurrently; a `utils.py` grab bag; custom exception
hierarchies nobody catches; re-implemented `functools.lru_cache`,
`cached_property`, `itertools.groupby` or `collections.Counter`.

## Format

`L<line>: <tag> <what>. <replacement>.`, or `<file>:L<line>: ...` for
multi-file diffs. Rank findings biggest cut first.

## Examples

❌ "This helper class might be more complex than necessary, have you
considered whether a simpler approach would be cleaner here?"

✅ `cache.py:L1-40: stdlib: DealCache class with dict + manual eviction. @functools.lru_cache(maxsize=256) on the fetch.`

✅ `client.py:L22-48: dep: hand-rolled retry/backoff loop. httpx transport retries / the SDK's max_retries.`

✅ `notifier.py:L5: yagni: BaseNotifier(ABC) with one subclass. Drop the ABC, keep SlackNotifier as a function.`

✅ `settings.py:L10-30: dep: manual env parsing and type casting. pydantic-settings / a pydantic model, already installed.`

✅ `report.py:L30-44: shrink: loop + append builds a dict. {r.id: r for r in rows}, 1 line.`

✅ `pyproject.toml:L14: stdlib: pytz added. zoneinfo.ZoneInfo, 0 new deps.`

## Scoring

End with the only metric that matters: `net: -<N> lines, -<M> deps possible.`

If there is nothing to cut, say `Lean already. Ship.` and stop.

## Boundaries

Scope: over-engineering and complexity only. Correctness bugs, security holes,
and performance are explicitly out of scope. Route them to a normal review
pass, not this one. A single small test run via the repo's own runner is
the pytail minimum, not bloat, never flag it for deletion. Lists findings,
applies nothing.
"stop pytail-review" or "normal mode": revert to verbose review style.
