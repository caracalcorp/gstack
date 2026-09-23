# caracalcorp/gstack — what this fork changes, and why

This is a fork of [garrytan/gstack](https://github.com/garrytan/gstack) that
carries org-local commits. It is **not** a mirror, and `gh repo sync` does not
work on it.

This file is the **patch register**. It exists so that a conflict resolution has
something to check itself against: when `git merge upstream/main` touches one of
these files, the question "was this ours, and what was it for?" has a written
answer instead of a `git blame` archaeology session.

Authoritative check, which is what actually enforces this file:

```sh
bin/gstack-fork-verify --tree .        # from the config repo (~/rs/config)
```

13 assertions. It is the gate; this file is the explanation.

---

## Why these patches exist

Everything that decides **which repository gstack talks to** must name ours.
Upstream hardcodes its own org in a handful of places, and a fork that leaves
them pointing at upstream will quietly upgrade itself from a repo it does not
control.

The failure this guards against is not a loud one. `git merge` resolving a hunk
in upstream's favour raises no error, and a reverted CI runner label does not
fail a job, it **queues** it — which reads as slow CI, not absent CI.

---

## The patch set

| File | What changed | Why |
|---|---|---|
| `bin/gstack-update-check` | one `GSTACK_FORK_ORG` constant, used three times | Upstream hardcodes its org in three places, and the third is a trap: lines 53-54 honour `GSTACK_REMOTE_URL`/`GSTACK_REMOTE_REPO`, the SHA-pinned URL does not. Override only the repo and it `ls-remote`s our fork, requests that SHA from **upstream's** raw host, 404s, falls through, and silently reads upstream's VERSION. One constant makes a partial revert impossible. |
| `gstack-upgrade/SKILL.md.tmpl` | vendored-install clone URL | The upgrade path's fallback arm would clone upstream. |
| `gstack-upgrade/SKILL.md` | regenerated | Generated from the `.tmpl`. A patched template with a stale render is a half-applied patch, and the render is what runs. |
| `bin/gstack-team-init` | 5 teammate clone instructions | Would hand a teammate upstream's URL. |
| `scripts/gen-llms-txt.ts` | repository line | Emits into `gstack/llms.txt`. |
| `scripts/gen-agents-digest.ts` | repository lines | Emits into `agents-digest/gstack-AGENTS.md`. |
| `gstack/llms.txt` | regenerated | Committed alongside its generator, same reason as the SKILL.md above. |
| `agents-digest/gstack-AGENTS.md` | regenerated | Same. |
| `.github/workflows/free-tests.yml` | `free-suite` runner → `ubuntu-24.04` | Upstream runs it on `ubicloud-standard-8`, a third-party provider. On this fork that label matches no runner, so the job queues until timeout instead of failing. `ubuntu-24.04` is what every other job in the file already uses. |
| `.github/workflows/quality-gate.yml` | runner → `ubuntu-24.04` | Same. |

**Zero test changes** in the pointer set above. That is deliberate and worth
preserving — it is the cleanest possible patch set to carry through a merge.

### Race fixes (not pointers)

A second, separate category, added 2026-09-23. Both are upstream defects that
made `gstack-sync` step 4 fail intermittently under the 6-shard load, and a
flaky test cannot go in `known-failing-tests.txt`: the gate compares exactly, so
it would fail as NOW PASSING on every clean run. **Drop each row the moment
upstream fixes the same race** — resolve the conflict in upstream's favour.

| File | What changed | Why |
|---|---|---|
| `bin/gstack-codex-probe` | bash-native watchdog in `_gstack_codex_timeout_wrapper` | After its kill, the watchdog subshell still had to exit; `kill -0` read that window as "finished early" and returned 143 instead of 124. Measured 16/40 wrong with 40 concurrent runs. Now the watchdog ignores TERM once it fires and the wrapper reads its exit status. 160/160 correct idle and loaded. Affects real hosts without `timeout(1)` (stock macOS), not only the test. |
| `test/office-hours-attempt.test.ts` | `exitsSoon()` polls up to 1s where two tests sampled `ps` once | The runner returns right after *sending* the group SIGKILL; the target stays visible 9-51ms under load. 6/48 failed loaded, 0/72 after. Still red on a real leak (verified with `setsid` escaping the group). The one test change this fork carries. |

---

## What is deliberately NOT patched

The full classification lives in the config repo at
`roles/gstack/files/fork-pointer-allowlist.txt`, and `gstack-fork-verify` fails
on any upstream pointer that appears in neither that file nor the table above.
The categories:

- **Provenance pins** — `lib/cso/*.ts`, `scripts/cso-*.ts`. The `/cso` skill
  accepts container images only from upstream's registry, pinned by sha256, and
  a qualification attestation only from a run of upstream's own workflow. That
  is a supply-chain control and upstream's name is load-bearing in it.
  Repointing would not move the trust anchor, it would **delete** it and name
  artifacts we do not build.
- **Attribution** — `browse/src/welcome.html`, `LICENSE`. MIT requires the
  notice. We do not strip credit.
- **Fixture data** — `test/**`. `garrytan` there is a sample owner name in slug
  and URL-parsing tests, not a pointer. Patching would churn the suite and prove
  nothing.
- **Upstream prose** — `docs/**`, `README.md`, `CHANGELOG.md`, `CONTRIBUTING.md`.
  We read it; we do not fork it.
- **`setup`** — three doc-link comments, zero function, and it was touched by 6
  of the 12 commits in one upstream window. The worst merge surface in the tree
  for no benefit.
- **`VERSION`** — touched by every single upstream release. Any fork version
  scheme conflicts on every sync, and `gstack-update-check` validates
  `^[0-9]+\.[0-9.]+$`, so a `+caracal` suffix silently breaks update checks.
- **`bin/gstack-gbrain-install`** — deferred to the gbrain lane, which owns the
  `caracalcorp/gbrain` decision. It is also the only pointer with a coupled
  test, and deferring it is what keeps this patch set test-free.

---

## Syncing

Never `gh repo sync` — it fast-forwards only, and answers `HTTP 404` once the
fork is ahead. From the config repo:

```sh
just gstack-sync            # merge, regenerate, verify, test, publish, deploy
just gstack-sync --check    # stop before publishing
```

Conflicts stop the script and hand you the tree. Resolve in favour of the table
above, commit, re-run.

**Expect upstream to redesign under a patch, not merely collide with it.** The
first real sync conflicted in `free-tests.yml` because upstream had restructured
`free-suite` into a sharded matrix — and the right resolution was to **drop**
half our patch (a timeout bump that only existed because one job ran every
shard). File-touch counts do not predict that. Budget a session.

## CI

Two workflows are enabled: **Free Tests** and **Quality gate**. Every other
workflow is `disabled_manually`, including `E2E Evals` and `Periodic Evals`,
which cost money.

They are not off by default. Enabling Actions to turn on two lanes turns on all
of them. Check after any sync that adds a workflow — a new file arrives active:

```sh
gh api /repos/caracalcorp/gstack/actions/workflows -q '.workflows[] | "\(.state)\t\(.name)"' | sort
```
