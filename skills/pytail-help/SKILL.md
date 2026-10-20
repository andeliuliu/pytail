---
name: pytail-help
description: >
  Quick-reference card for all pytail modes, skills, and commands.
  One-shot display, not a persistent mode. Trigger: /pytail-help,
  "pytail help", "what pytail commands", "how do I use pytail".
---

# Pytail Help

Display this reference card when invoked. One-shot, do NOT change mode,
write flag files, or persist anything.

Pytail is a Python-tailored sibling of railtail, both forked from ponytail. It
does NOT auto-activate on session start — invoke `/pytail` to turn it on.

## Levels

| Level | Trigger | What change |
|-------|---------|-------------|
| **Lite** | `/pytail lite` | Build what's asked, name the lazier alternative in one line. |
| **Full** | `/pytail` | The ladder enforced: YAGNI → already in the repo → stdlib → installed dep → platform → one line → minimum. Default once invoked. |
| **Ultra** | `/pytail ultra` | YAGNI extremist. Deletion before addition. Challenges requirements before building. |

Level sticks until changed or session end. While on, every response opens with
`🐍 pytail · <level>` — no label means it's off.

## Skills

| Skill | Trigger | What it does |
|-------|---------|--------------|
| **pytail** | `/pytail` | Lazy mode itself. Simplest solution that works, a function before a class, stdlib before a package. |
| **pytail-review** | `/pytail-review` | Over-engineering review of this branch's diff vs parent (or a file/staged changes you point at): ranked list of what to delete/reuse/shrink. `L12: stdlib: cache class. @lru_cache.` |
| **pytail-audit** | `/pytail-audit [path]` | The same over-engineering review across the whole repo (or one package): also unused dependencies, modules nothing calls, logic written twice. Tests are left to pytail-tests. |
| **pytail-pr-commenter** | `/pytail-pr-commenter [PR] [--top-k K]` | Discover once, verify ranked candidates until K survive (default 10) or candidates run out, then post one comment. `--dry-run` previews. |
| **pytail-tests** | `/pytail-tests` | Over-testing review — which tests to cut: redundant, library-trivial, tautological-mock, dead, or near-duplicates `parametrize` would merge. |
| **pytail-help** | `/pytail-help` | This card. |

Code and test reviews discover candidates once, then try to disprove each cut
with targeted checks. Only verified suggestions are reported; test cuts must
preserve the failure signal. There is no convergence loop.

## Parallel work

For 12 or more files that form independent areas, `pytail`, `pytail-review`,
`pytail-audit` and `pytail-tests` may use up to three subagents while one coordinator owns
shared context, deduplication, and final verification. Tightly coupled or
smaller work stays with one agent. This help card never delegates.

## The ladder (full mode)

1. Does this need to exist at all? (YAGNI)
2. Already in this repo (a function, model, client, helper)?
3. Python standard library?
4. Already-installed dependency (pydantic, httpx, …)?
5. The platform already does it (DB constraint, framework built-in, existing endpoint)?
6. One line?
7. Only then: the minimum code that works.

Repo rules in `AGENTS.md` / `CLAUDE.md` rank above these defaults.

## Deactivate

Say "stop pytail" or "normal mode". Resume anytime with `/pytail`.

## More

Source & issues: https://github.com/andeliuliu/pytail
Rails sibling: https://github.com/andeliuliu/railtail
Upstream original (generic): https://github.com/DietrichGebert/ponytail
