# Test harness

RSpec against a **real Redis**. There is no fake-Redis layer and no mocking of
the lock protocol: lock behaviour is Redis behaviour, so the suite talks to a
server and flushes the database around every example.

Two `before` hooks configure that connection, and the later registration wins.
`spec/support/sidekiq_meta.rb` — registered first, because `spec_helper.rb`
loads `spec/support/**` at line 30 — sets the Sidekiq client to
`redis://<REDIS_HOST>/<redis_db>`. `spec_helper.rb`'s own hook, registered in
the `RSpec.configure` block below it, then sets both client and server to
`{ port: 6379 }`. The net effect is **localhost:6379, database 0**, whatever
`REDIS_HOST` and the `redis_db:` metadata say. CI sets `REDIS_HOST=localhost`
and maps the Redis container's port, so the two agree there.

## Layout

114 `*_spec.rb` files under `spec/`:

| Directory | What lives there |
|---|---|
| `spec/sidekiq_unique_jobs/` | one file per class — `locksmith_spec.rb`, `lock_digest_spec.rb`, `digests_spec.rb`, `upgrade_locks_spec.rb`, … |
| `spec/sidekiq_unique_jobs/lock/` | one per lock class, end-to-end through `#lock` and `#execute` |
| `spec/sidekiq_unique_jobs/middleware/` | client and server chains, plus `server/until_and_while_executing_spec.rb` |
| `spec/sidekiq_unique_jobs/on_conflict/` | one per strategy |
| `spec/sidekiq_unique_jobs/orphans/`, `fetch/`, `script/`, `redis/`, `web/` | reaper, reliable fetch, script loading, the Redis wrappers, the Web helpers |
| `spec/workers/` | one per fixture worker in `spec/support/workers/` — the user-facing `sidekiq_options` contract |
| `spec/sidekiq/` | the gem's effect on `Sidekiq::Api`, `Sidekiq::Job`, `Sidekiq::RetrySet` |
| `spec/integration/` | one file, wrapped in `unless ENV.fetch("CI", false)` — it drives `until_and_while_executing` through a toxiproxy, so it runs locally and never in CI |
| `spec/performance/` | benchmarks tagged `:perf`, which `spec/support/rspec_benchmark.rb` excludes by default |
| `spec/support/` | helpers, fixture workers, shared contexts and examples, Lua fixtures |

CI runs `bin/rspec --require spec_helper --tag ~perf`. The tag is belt and
braces: `spec/support/rspec_benchmark.rb` already calls
`config.filter_run_excluding perf: true`, so a plain `rspec` skips the
benchmarks too, and `spec/performance/` runs only under `--tag perf`.

## `spec_helper.rb`

`spec/spec_helper.rb` requires every file under `spec/support/**` (twice — lines
30 and 88, harmlessly), installs both middlewares into Sidekiq's client and
server chains, and configures the gem with `lock_info = true`, `max_history =
10` and `debug_lua` from `DEBUG_LUA`. `LOGLEVEL` (default `ERROR`) sets the
logger level. `SCRIPTS_PATH` points at `spec/support/lua`, which holds
`lock.lua`, `test.lua` and two shared partials used by the `Script::*` specs so
they never load the gem's real scripts.

`config.include SidekiqUniqueJobs::Testing` pulls in `spec/support/sidekiq_unique_jobs/testing.rb`
(892 lines), a thin `conn.<command>` delegator for most of the Redis API plus
`push_item`, `queue_count`, `schedule_count`, `retry_count`, `dead_count`,
`unique_keys`, `locked_jids` and `flush_redis`.

## Per-example metadata

`spec/support/sidekiq_meta.rb` runs before every example:

- `redis_db:` (default `0`) goes into the client URL it builds — overridden a
  moment later, see above — and then `flush_redis` runs; an `after` hook flushes
  again.
- `sidekiq:` (default `:disable`) calls `Sidekiq::Testing.<value>!`; `true` is
  read as `:fake`. So **the default is `Sidekiq::Testing.disable!`** — real
  pushes to real Redis — and a spec opts into `:fake` or `:inline` explicitly.
- `sidekiq_ver:` skips the example unless `VersionCheck.satisfied?` against
  `Sidekiq::VERSION`; `spec/support/ruby_meta.rb` does the same for `ruby_ver:`
  against `RUBY_VERSION`.

Shared contexts (`spec/support/shared_contexts/`) cover the rest:
`:with_global_config` wraps the example in `SidekiqUniqueJobs.use_config`,
`:with_job_options` in `job_class.use_options`, `:with_sidekiq_options` in
`Sidekiq.use_options` — all three restore in an `ensure`, which is why a spec
must use them rather than assigning config directly.

## Shared examples

- `"a lock implementation"` — lockable, blocks a second process, records a
  `lock_failed` metric, calls `call_strategy(origin: :client)`.
- `"an executing lock implementation"` — stays locked while executing, stays
  locked when the block raises, blocks a second process, reflects
  `:execution_failed`.
- `"an executing lock with error handling"`, `"a performing worker"`,
  `"sidekiq with options"`.

## Matchers and config

`spec/support/matchers/` adds `resemble_date`, `have_enqueued` and
`be_enqueued_in`. `lib/sidekiq_unique_jobs/rspec/matchers.rb` ships
`have_valid_sidekiq_options` to applications, and it has its own spec.

Coverage is opt-in: `.simplecov` only loads when `COV` is set. Outside CI it
enforces `minimum_coverage line: 90, branch: 80` per file and refuses a drop;
in CI those thresholds are not applied.

## Dead helpers

`spec/support/simulate_lock.rb` is included into every example group and called
by nothing. Its `simulate_lock` writes `key.queued`, `key.primed` and
`key.changelog`, none of which `Key` still defines, and `lock_jid` passes four
ARGV entries where `lock.lua` now takes five. The same is true of
`locking_jids` / `queued_jids` / `primed_jids`, `changelogs` and `expired_keys`
in `spec/support/sidekiq_unique_jobs/testing.rb` — `SidekiqUniqueJobs::Changelog`
and `EXPIRING_DIGESTS` no longer exist. Treat all of it as v8 residue; do not
build a new spec on it.

## Related

- [`../workflow.md`](../workflow.md) — the commands, the CI matrix and what green means
- [`../../.claude/rules/testing.md`](../../.claude/rules/testing.md) — the TDD loop and coverage rules
