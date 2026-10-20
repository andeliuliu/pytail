---
name: pytail-audit
description: >
  Whole-repo audit for over-engineering in Python code. Like pytail-review, but
  scans every tracked Python file and the dependency manifest instead of a
  diff: a ranked list of what to delete, simplify, or replace with existing repo
  code / an installed dependency / the standard library, including what only a
  whole-repo view shows (dependencies nothing imports, modules nothing calls,
  the same logic written twice). One line per finding. Use when the user says
  "audit this codebase", "audit the repo", "audit for over-engineering", "what
  can I delete from this repo", "find bloat in the whole project", or invokes
  /pytail-audit. Lists findings, applies nothing.
---

Audit the whole repository for unnecessary complexity, ranked biggest cut first.
One line per finding: location, what to cut, what replaces it. The repo's best
outcome is getting shorter.

## Scope

Every tracked Python source file plus the dependency manifest:

`git ls-files '*.py' pyproject.toml 'requirements*.txt' setup.py setup.cfg`

Leave out:

- **Tests** (`tests/`, `test_*.py`, `*_test.py`, `conftest.py`). Over-testing is
  `pytail-tests`'s job. Read tests only as evidence: a function whose only caller
  is a test is a candidate.
- **Vendored, generated and migration code** (`vendor/`, `third_party/`,
  `*_pb2.py`, `migrations/`, `alembic/versions/`). Nobody hand-shrinks those.

If the user names a directory or package, audit only that. If the repo has no
git history, use the Python files under the current directory with the same
exclusions.

Repo rules in `AGENTS.md` / `CLAUDE.md` rank above these defaults. A pattern
those files require (a registry, a decorator, a folder convention) is never a
finding, however much it looks like indirection.

## Discover once, then verify

Scan the scoped files once to collect candidates. Deduplicate by root cause and
rank biggest worthwhile cut first; prefer better-supported, lower-risk cuts for
ties. Keep the location, proposed replacement, and expected benefit with each
candidate. This list is provisional, not the final findings.

Then work down that list, trying to **disprove each suggestion** with targeted
reads of the relevant code, callers, and tests:

- What behavior would break or disappear if this were removed or shortened?
  Check actual usage, contracts, edge cases, and dynamic entry points:
  `python -m` modules, `__main__` blocks, console scripts in `pyproject.toml`,
  decorator registries, framework hooks, `importlib`/string imports, and anything
  a Dockerfile, Procfile, CI file or README tells someone to run. A missing text
  match alone does not prove code is dead.
- Does the named replacement exist in this repository or installed version,
  and does it support this use case, including inputs, outputs, and side effects?
- Does the change actually reduce complexity without moving it elsewhere or
  overlapping another suggestion? Keep one actionable cut per root cause.

Reject unsupported or behavior-changing cuts. If a candidate needs correction,
check the corrected proposal against the same evidence before retaining it.
Use a focused existing test only when it can resolve a concrete uncertainty;
do not edit code to prove an audit suggestion. Reuse evidence already read.

By default, verify all discovered candidates and report only survivors. If the
user supplies an output limit K ("top 10"), verify in rank order until K
suggestions survive or candidates run out; rejected candidates do not consume
slots. Do not pad the result or verify the remainder just to count it. Review
retained suggestions together for conflicting cuts before finalizing; backfill
from remaining candidates if needed.

Do not repeat full-repo discovery or loop until the list stabilizes. Score only
reported, verified suggestions. If missing context prevents verification, state
that limitation instead of presenting an unverified candidate or a clean bill.

## Large repos

Audit directly unless the scope holds at least 12 files and splits into two or
more coherent areas. Then, if subagents are available:

1. Keep one coordinator. Group files by package or layer, not arbitrary file
   counts, and give each of at most three subagents a disjoint group and this
   skill's rubric.
2. Have subagents audit only their group and return candidate findings in the
   normal format. The coordinator keeps the cross-cutting hunts that need the
   whole repo: unused dependencies, modules nothing imports, and logic
   duplicated across groups.
3. Apply the targeted verification workflow before including any candidate.
   Merge overlapping findings, prefer one root-cause cut over repeated per-file
   cuts, rank once, and calculate one net score.

If the files are tightly coupled or cannot form two meaningful groups, do not
delegate. Subagents reduce elapsed time; they never expand scope or multiply
findings.

## Tags (mirror pytail's ladder, highest rung first)

- `delete:` dead code, unused flexibility, speculative feature, a dependency nothing imports. Replacement: nothing.
- `reuse:` hand-rolled logic an existing function, model, client or helper in this repo already does, including the same logic written twice in two modules. Name the one to keep. (Rung 2)
- `stdlib:` hand-rolled thing the standard library ships. Name the module and function. (Rung 3)
- `dep:` custom code doing what an installed dependency ships (pydantic validation, httpx/requests retries and timeouts, a framework feature), or a dependency for what's already covered. Name the feature. (Rung 4)
- `yagni:` abstraction with one implementation (ABC/Protocol with one subclass, factory for one product), a class that should be a function, config nobody sets, a layer with one caller.
- `shrink:` same logic, fewer lines. Show the shorter form.

## Hunt

Everything `pytail-review` hunts: packages the stdlib already ships
(`pytz`/`python-dateutil` for `zoneinfo`, `requests` beside an installed `httpx`,
`simplejson`); classes with one method plus `__init__`; `Manager`/`Helper`/
`Service` wrappers around one call; getters and setters where a `dataclass` or
pydantic model fits; `ABC`/`Protocol` with one implementation; hand-rolled retry
loops when the client has retries; manual `dict`/`list` building a comprehension
does; `os.path` string juggling where `pathlib` is shorter; `try/except: pass`
swallowing errors; `async` with nothing awaited concurrently; a `utils.py` grab
bag; custom exception hierarchies nobody catches; re-implemented
`functools.lru_cache`, `cached_property`, `itertools.groupby` or
`collections.Counter`.

Plus what only the whole repo shows:

- **Dependencies nothing imports.** Each runtime dependency in the manifest
  against the imports in scope. Mind import names that differ from package
  names (`pyyaml` → `yaml`, `python-dateutil` → `dateutil`, `slack-bolt` →
  `slack_bolt`), plugins loaded by entry point, and dev tools run from the
  command line rather than imported.
- **Modules and public functions nothing calls**, after checking the dynamic
  entry points listed above.
- **The same logic in two places**: two helpers that build the same URL, parse
  the same header, retry the same way, or wrap the same client.
- **Config and flags nobody sets**: environment variables read with a default
  that no `.env.example`, deploy file or doc ever overrides, parameters every
  caller passes the same value for.

## Format

`<file>:L<line>: <tag> <what>. <replacement>.` Rank findings biggest cut first.

## Examples

✅ `app/cache.py:L1-40: stdlib: DealCache class with dict + manual eviction. @functools.lru_cache(maxsize=256) on the fetch.`

✅ `pyproject.toml:L14: delete: requests, nothing imports it (httpx does every call). Remove it, -1 dep.`

✅ `app/slack.py:L30-41: reuse: second Gmail-link builder. inbox.gmail_link(), already used by find_emails.`

✅ `app/notifier.py:L5: yagni: BaseNotifier(ABC) with one subclass. Drop the ABC, keep SlackNotifier as a function.`

✅ `app/settings.py:L10-30: dep: manual env parsing and type casting. A pydantic model, already installed.`

## Scoring

End with the only metric that matters: `net: -<N> lines, -<M> deps possible.`

If there is nothing to cut, say `Lean already. Ship.` and stop.

## Boundaries

Scope: over-engineering and complexity only. Correctness bugs, security holes,
and performance are explicitly out of scope. Route them to a normal review
pass, not this one. A single small test run via the repo's own runner is the
pytail minimum, not bloat, never flag it for deletion. Lists findings, applies
nothing. One-shot. "stop pytail-audit" or "normal mode": revert to verbose
review style.
