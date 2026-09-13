# Terminology

The words this repository uses, and what each one means in the code.

- **digest** — the lock's identity: `"<lock_prefix>:<hash>"`, built by
  `LockDigest#create_digest` (`lib/sidekiq_unique_jobs/lock_digest.rb:53-61`)
  from the job hash's `class`, `queue`, `lock_args` and `apartment` keys. Two
  jobs with the same digest are "the same job".
- **lock_args** — the arguments that actually feed the digest, after
  `LockArgs#filtered_args` (`lock_args.rb:64-75`) applies the worker's
  `lock_args_method`, `lock_args` class method, or legacy `unique_args`. With no
  filter configured this is the whole `args` array; with one configured that is
  neither a `Proc` nor a `Symbol` (a stray String, say) the `case` falls through
  and `lock_args` is `nil`.
- **LOCKED hash** — `"<digest>:LOCKED"`, a Redis hash of `job_id => metadata
  JSON`. Its existence for a given `job_id` *is* the lock. Built by
  `Key#initialize` (`key.rb:26-30`).
- **digests ZSET** — `uniquejobs:digests` (constant `DIGESTS`), the single
  sorted set indexing every live digest. Score is the lock time, except in
  `lock.lua` where an `until_expired` lock with a positive `pttl` is scored
  `current_time + pttl`.
- **RUN digest** — `"<digest>:RUN"`. `Lock::WhileExecuting#append_unique_key_suffix`
  (`while_executing.rb:63-67`) appends `RUN_SUFFIX` so the execution-time lock of
  an `until_and_while_executing` job never collides with its enqueue-time lock.
  `Orphans::Reaper` declares its own `RUN_SUFFIX` (`orphans/reaper.rb:19`) to
  strip it back off before matching.
- **lock type** — the `sidekiq_options lock:` value. Fourteen aliases map onto
  six classes in `Config::LOCKS` (`config.rb:77-82`, merged from the four
  `LOCKS_*` hashes at `config.rb:43-73`).
- **conflict strategy** — what happens to the loser. Five entries in
  `Config::STRATEGIES` (`config.rb:86-92`); `OnConflict.find_strategy`
  (`on_conflict.rb:31-42`) returns `NullStrategy` for `nil` and, for an unknown
  name, logs a warning and returns `NullStrategy` — it never raises.
- **client / server** — which Sidekiq middleware chain. `on_conflict:` may be a
  hash `{ client:, server: }`, split by `LockConfig#on_client_conflict`
  (`lock_config.rb:115-121`) and `#on_server_conflict` (`124-130`); an explicit
  `on_client_conflict`/`on_server_conflict` key on the job hash wins over both.
- **lock_ttl** — seconds the lock lives; `LockConfig` turns it into `pttl`
  (milliseconds) and `lock.lua` `PEXPIRE`s the LOCKED hash when `pttl > 0`. For
  a scheduled job `LockTTL#calculate` adds `time_until_scheduled`.
- **lock_timeout** — historically the acquisition wait. Inert in v9:
  `Locksmith#lock(wait:)` (`locksmith.rb:54`) declares the argument and never
  reads it. Still parsed by `LockTimeout` and written into the lock metadata.
- **lock_limit** — how many holders the LOCKED hash may have. `lock.lua`
  refuses when `HLEN >= limit`; the default from `LockConfig#initialize`
  (`lock_config.rb:62`) is 1.
- **orphan** — a digest in the ZSET whose job is nowhere: not enqueued, not
  scheduled, not retrying, not running. `Orphans::Reaper#belongs_to_job?`
  (`orphans/reaper.rb:78-85`) is the definition.
- **reaper mutex** — `uniquejobs:reaper` (constant `UNIQUE_REAPER`), a
  `SET NX EX` key that elects one process to run the reaper
  (`server.rb:126-132`).
- **resurrector** — the timer every non-reaper process runs to take over when
  the reaper mutex lapses (`server.rb:99-121`).
- **working list** — `uniquejobs:working:<identity>` (`Key.working`), the
  per-process list `Fetch::Reliable` moves jobs into; `identity` is
  `"#{hostname}:#{pid}:#{SecureRandom.hex(6)}"` (`fetch/reliable.rb:41`).
- **heartbeat** — `uniquejobs:heartbeat:<identity>` (`Key.heartbeat`), a
  60-second key refreshed every 20 seconds (`HEARTBEAT_TTL` /
  `HEARTBEAT_INTERVAL`); its absence is how another process knows a working list
  is abandoned.
- **reflection** — an observability hook. `Reflections::REFLECTIONS` declares
  fourteen names; `SidekiqUniqueJobs.reflect { |on| … }` registers one block per
  name and `Reflectable#reflect` dispatches. Registration is per key, so a
  second `reflect` block merges rather than replaces.
- **script injection** — `Script::Caller#do_call` (`script/caller.rb:56-60`)
  appends `now_f, debug_lua, max_history, script_name, redis_version` to every
  ARGV, so a script's own arguments come first and the injected ones follow.
- **myapp/** — a localhost-only Rails 8 app used to drive locks by hand during
  development. Never deployed; excluded from the packaged gem by the gemspec's
  file glob and by `release.yml`'s "Verify gem contents" step.
