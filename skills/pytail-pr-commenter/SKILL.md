---
name: pytail-pr-commenter
description: >
  Discover simplification candidates on a GitHub PR, verify them against the
  code, then post one comment with up to k verified suggestions (default 10).
  Use when the user invokes /pytail-pr-commenter or pytail-PR-commenter,
  or asks for a verified Pytail review followed by a PR comment.
---

Discover once, verify candidates, then comment once. Do not edit the reviewed code.

## Inputs and PR scope

Usage: `/pytail-pr-commenter [PR URL or number] [--top-k K] [--dry-run]`.

- `--top-k` defaults to 10 and must be a positive integer. It limits suggestions
  in the final comment, never initial discovery. Do not pad to K.
- `--dry-run` produces the final comment locally without posting it.

Resolve the explicit PR, or the current branch's PR, with the available GitHub
  connector or `gh`. Confirm the repository and PR identity from its metadata;
  if there is no unique target, ask for it. Fetch/read its actual base and head
  commits and pin the review to those SHAs. Review their merge-base diff rather
  than assuming the repository's default branch or including local edits.
  If the PR diff or needed context cannot be read completely, report the blocker
  instead of treating missing evidence as a clean review.

Read [pytail-review](../pytail-review/SKILL.md) and use its tags, hunt,
boundaries, finding format, and verification checks. This skill overrides its
scope resolution with the PR's pinned diff and publishes the verified output.
Keep correctness, security, and performance findings out of this review.

## Discover, verify, and comment

Run pytail-review's discovery and targeted verification workflow once on the
pinned diff, with K as its output limit. Discover and rank candidates across the
whole diff, then try to disprove candidates in rank order until K distinct
suggestions survive or the list is exhausted. Do not first run an unlimited
review and then repeat its verification here. Never rescan for convergence or
verify candidates below the cutoff just to fill a total count.

Draft one consolidated PR conversation comment with a `Pytail review` heading
and a numbered list in pytail-review's concise finding format. Link locations
to the reviewed head SHA. End with the net lines/dependencies possible for
**the selected suggestions only**, without double-counting shared cuts; label
estimates and say when a quantity cannot be estimated reliably. Do not describe
unexamined candidates as validated findings.
If no suggestions survive, report `No verified simplifications to suggest.`
locally and post no empty comment. Disclose missing evidence locally; do not
claim the review was clean when verification was blocked.

Before posting, re-read the PR's base/head SHAs. If either changed, stop without
posting and report that the review is stale and needs a fresh invocation.
For a dry run, return the drafted comment and a brief verification summary
locally. Report selected, rejected, and unexamined candidate counts separately.

An explicit invocation of this commenter authorizes its final PR comment;
honor any user instruction to preview only. Merely discovering this skill or
being asked to review code does not authorize posting. If posting was not
requested, finish the draft before asking whether to publish it.

Use the available GitHub connector or `gh pr comment` to publish exactly one
conversation comment, not an approval or change-request review. With `gh`, write
the exact Markdown to a temporary file and use `--body-file` to preserve it.
Include `<!-- pytail-pr-commenter head:<reviewed-head-sha> -->` in the body.
Before posting, check existing comments: if the same marker and selected
suggestions are already present, return that comment's URL rather than posting
a duplicate. After an uncertain submission result, inspect the PR comments for
the attempted body before retrying; if the outcome remains unknown, report it
without another write. Never edit or delete other comments.

Return the posted comment URL and the number of verified suggestions posted.
If GitHub access or posting fails, return the prepared draft and the actual
blocker; do not claim it was published.
