# Which GitHub Actions caches can a `pull_request` run restore?

A four-run probe, because the documentation says a run can restore caches from
"the current branch, the base branch, and the default branch" and that turns out
to hide the case that matters: **a cache written by a `pull_request` run cannot be
restored by anything, including a later run on the same pull request.**

`.github/workflows/probe.yml` runs on push to `main` and on `pull_request`. It
restores three caches, prints what each one found, then writes a marker naming
the run that wrote it. The marker is how each result identifies its source.

| Cache        | Key                        | `restore-keys`      | Isolates                     |
| ------------ | -------------------------- | ------------------- | ---------------------------- |
| `v6-prefix`  | `…-${{ github.run_id }}`   | `probe-v6-prefix-`  | prefix matching, `cache@v6`  |
| `v4-prefix`  | `…-${{ github.run_id }}`   | `probe-v4-prefix-`  | prefix matching, `cache@v4`  |
| `v6-exact`   | `probe-v6-exact-fixed`     | none                | can the scope see it at all  |

Keying on `run_id` guarantees the prefix caches never match exactly, so a restore
there can only have come through `restore-keys` — which is what a real build-info
or compiler cache depends on.

## Results

| Run | Event          | Restored from                     |
| --- | -------------- | --------------------------------- |
| 1   | push `main`    | nothing (first run)               |
| 2   | push `main`    | run 1 (`refs/heads/main`)         |
| 3   | `pull_request` | run 2 (`refs/heads/main`)         |
| 4   | `pull_request` | run 2 (`refs/heads/main`)         |

Run 4 is the finding. Its own pull request had already written a cache in run 3,
and that entry was **newer** than the one on `main`:

```
2026-…T07:36:23 ref=refs/heads/main    key=probe-v4-prefix-33728945180   ← restored by run 4
2026-…T07:37:00 ref=refs/pull/1/merge  key=probe-v4-prefix-33729001226   ← ignored
```

Run 4 reached past it to `main`. Identical for `cache@v4` and `cache@v6`, and for
both prefix and exact keys, so this is the cache service's scoping rather than an
action version or a key-matching quirk.

## What it means

A `pull_request` run writes its cache to `refs/pull/<N>/merge`. Nothing reads that
scope back — not a later run on the same pull request, and not the branch after
merge. From a pull request's point of view, cache writes are **write-only**, and
every restore comes from the base or default branch.

So a workflow that only runs on `pull_request` gets a cache that is never read.
Every run is cold, forever, and the cache list fills with entries nothing will
ever restore. Making it useful takes a run on the default branch:

```yaml
on:
  push:
    branches: [main]   # the only writer whose cache a pull request can read
  pull_request:
```

Two consequences worth planning around:

- **The first run on a new pull request is as warm as the default branch's cache,
  and no warmer.** Pushing again to the same pull request does not improve it.
- **The default branch's cache is the only one that ages.** A key pinned to
  something slow-moving (a lockfile hash, say) with `restore-keys` for the prefix
  keeps it useful; a key that changes every commit still restores through the
  prefix, but only ever from the default branch's latest.

`cache-hit` is `true` only for an exact key match. A `restore-keys` match reports
`cache-hit: false` while still restoring the files — checking `cache-hit` to
decide whether the cache was usable reads a prefix restore as a miss.

## Reproducing

Push to `main` twice, open a pull request, then push to it again, and read the
`RESULT` lines in each run's log.

run 6
