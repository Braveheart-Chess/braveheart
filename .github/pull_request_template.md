<!--
Title: one Conventional Commit header with one of this repository's scopes,
for example `feat(search): …`. It becomes the squash commit
on main, so write the line you want to read in `git log`.

Fill in every section. Write "n/a" rather than deleting one.
-->

## Summary

<!-- What this pull request does, in one or two sentences. -->

## Why

<!-- The problem or the motivation. For a fix: the bug, its root cause, and how to reproduce it. -->

Closes #

## Changes

-

## Verification

<!-- The commands you ran and what they showed, screenshots, or "CI only". -->

## Bench and SPRT

<!-- A search or eval change (`feat`): the bench signature before and after, and both SPRT results from the comment the `sprt` label posts. A speed change (`perf`): the unchanged signature, and NPS before and after on a quiet machine. Anything else: "n/a". CI checks the LAST `Bench:` line (the after) against its own run; the placeholder is not a claim. -->

- Bench before: `Bench: <nodes>`
- Bench after: `Bench: <nodes>`
- SPRT (timed and fixed nodes), or NPS before and after:

## Checklist

- [ ] The title is a Conventional Commit with one of this repository's scopes
- [ ] Every commit is one line, with only `Co-authored-by` trailers after it
- [ ] A breaking change is marked with `!` and explained under Why
- [ ] Docs are updated where behavior changed
