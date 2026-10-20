# pytail

**Lazy senior-dev mode for Python.** pytail forces the simplest solution that
actually works: reuse what's already in the repo, the standard library before a
package, an installed dependency before a new one, a function before a class, one
line before fifty — and cut the code and tests that don't earn their place. It's
the Python sibling of [railtail](https://github.com/andeliuliu/railtail), both forked
from [ponytail](https://github.com/DietrichGebert/ponytail).

pytail is **opt-in** — it does not auto-activate. Invoke `/pytail` to turn it on.

## Skills

| Command | What it does |
|---------|--------------|
| `/pytail [lite\|full\|ultra]` | Lazy mode itself — the ladder: YAGNI → already in the repo → stdlib → installed dep → platform → one line → minimum. Default level: `full`. |
| `/pytail-review` | Over-engineering review of a diff — a ranked list of what to delete / reuse / shrink. |
| `/pytail-audit` | The same review across the whole repo, plus what only a repo-wide view shows: dependencies nothing imports, modules nothing calls, logic written twice. |
| `/pytail-pr-commenter [PR] [--top-k K]` | Discover once, verify candidates, then post up to K suggestions (default 10) in one PR comment. |
| `/pytail-tests` | The test counterpart — which tests to cut (redundant, library-trivial, tautological mocks, near-duplicates, "it no longer does X"). |
| `/pytail-help` | Quick-reference card. |

## Install

```
/plugin marketplace add andeliuliu/pytail
/plugin install pytail@pytail
```

Or add to `settings.json`:

```json
"extraKnownMarketplaces": { "pytail": { "source": { "source": "github", "repo": "andeliuliu/pytail" } } },
"enabledPlugins": { "pytail@pytail": true }
```

## Usage

```
/pytail            # turn on lazy mode (full)
/pytail ultra      # YAGNI extremist: deletion before addition
/pytail-review     # review this branch's diff for over-engineering
/pytail-audit      # audit the whole repo for over-engineering
/pytail-audit agent/  # audit one package
/pytail-pr-commenter               # verify this branch's PR suggestions; comment top 10
/pytail-pr-commenter 123 --top-k 5  # comment at most 5 suggestions on PR #123
/pytail-pr-commenter --dry-run     # preview the final comment without posting
/pytail-tests        # review this branch's tests for over-testing
stop pytail        # turn it off
```

While it's on, every response opens with `🐍 pytail · full` (or `lite`/`ultra`) —
no label means it's off.

`pytail-review`, `pytail-audit` and `pytail-tests` use one discovery pass followed by targeted
verification that tries to disprove each proposed cut. They report only verified
suggestions, checking actual callers/replacements for code and preserved failure
signals for tests. They do not repeat full reviews until the list stabilizes.

`pytail-pr-commenter` (also referred to as `pytail-PR-commenter`) uses the same
workflow on a PR. It works down the ranked candidates until K suggestions are
verified or candidates run out, then posts one comment. K defaults to 10; rejected
candidates do not consume slots. It posts nothing when no suggestions survive.
GitHub access via a connector or authenticated `gh` is required; invoking the
commenter authorizes its final comment unless `--dry-run` is supplied.

## Notes

- **Generic Python, local rules from your repo.** The ladder and rules fit any
  modern Python codebase. Repo-specific conventions belong in your `AGENTS.md` or
  `CLAUDE.md`; pytail reads them first and ranks them above its defaults.
- **No setup.** Tooling is detected from the repo (`uv.lock` → uv, `poetry.lock` →
  poetry, else pip + `python -m pytest`). The review skills resolve your base branch
  from `origin/HEAD` (falling back to `develop`/`main`/`master`).
- **Shortcut markers.** Deliberate simplifications are marked `# ponytail:` so
  `/ponytail-debt` can list them all.
- **Parallel when it earns its keep.** Tasks or reviews spanning 12 or more
  separable files may use up to three subagents. One coordinator keeps shared
  context, removes duplicate work/findings, and verifies the final result.

## Credits

Forked from [ponytail](https://github.com/DietrichGebert/ponytail) by Dietrich
Gebert (MIT). pytail keeps ponytail's laziness logic and re-tailors it to Python. See [LICENSE](LICENSE).
