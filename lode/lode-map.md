# Lode map

The index of this repository's durable memory. Read this first; it beats a
directory listing. Every file describes the system as it is now, with rationale;
`../CHANGELOG.md` records what changed.

- `summary.md` — what the gem is, the two keys a lock is made of, the three invariants
- `terminology.md` — digest, LOCKED hash, digests ZSET, RUN digest, lock type, conflict strategy, orphan, reaper mutex, working list, reflection, script injection
- `practices.md` — Redis correctness (one script per transition, ARGV indexing, scan to cursor), failing safe, adding a config key, and the dead weight to leave alone
- `workflow.md` — the profile the shared `/lode:*` workflow skills read: commands, branches, layers, shapes, constraints, docs, CI, flake sources, conflicts, verification
- `plans/README.md` — where plans live

## Subsystems

- `lock-model/summary.md` — job hash → digest → two Redis keys; `Locksmith`; the six lock classes and fourteen aliases; the five conflict strategies and the validators
- `middleware/summary.md` — the two Sidekiq chains, `Server`'s startup/shutdown and reaper election, `Orphans::Reaper`, the opt-in `Fetch::Reliable`
- `lua-scripts/summary.md` — how scripts are loaded, cached and called; what `lock.lua`, `unlock.lua`, `ack.lua` and `fetch.lua` do line by line; which scripts and partials nothing calls
- `web-ui/summary.md` — the Locks tab: registration, the five routes, ZSCAN paging, the `Lock` read model, the metrics panel
- `test-harness/summary.md` — the spec layout, `spec_helper`, per-example metadata, shared examples and matchers, and the v8 helpers that no longer work

## Review rules (`review/`)

Accepted review findings rewritten as rules about the system, verified against
the code, each with the test that proves it or an honest "no test".
`/lode:gate` reads every file here before reviewing a diff; `/lode:learn` adds
to them.

- `review/lock-model.md` — no `ensure unlock` in `Locksmith#execute`, the `"force"` sentinel, pool-aware connections, `Lock#unlock` cleaning both keys, `Lock#created_at`, `VersionCheck` as a `Gem::Requirement` wrapper, two *Not a bug* entries
- `review/orphans-and-fetch.md` — the reaper's fail-closed rule, the `LRANGE` clamp, using the caller's connection, `BatchDelete`, scan-to-cursor, the fetcher's identity nonce, heartbeat shutdown, `RPUSH` on recovery, `Server`'s rescues, two *Not a bug* entries
- `review/web-and-metrics.md` — the tab's trailing slash, `flush` re-merging on error, the summed Failures column, three *Not a bug* entries about reflections, `by_type` and time in specs
- `review/tooling-and-release.md` — what the style gate really is, what the packaged gem may contain, tag/version matching, one *Not a bug* about the benchmark scripts
- `review/myapp.md` — the localhost-only stance, the `constantize` allowlist, literal Tailwind class names, three *Not a bug* entries

## Not memory

- `tmp/` — git-ignored: gate diffs and reports, handovers, scratch
