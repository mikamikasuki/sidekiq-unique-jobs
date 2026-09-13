# Lock model

How a job hash becomes a digest, how the digest becomes two Redis keys, and
which lock class decides when they go away.

## 1. From job hash to digest

`SidekiqUniqueJobs::Job.prepare(item)` (`lib/sidekiq_unique_jobs/job.rb:12-18`)
mutates the Sidekiq job hash in place, in this order: stringify a hash-valued
`on_conflict`, `add_lock_type`, `add_lock_timeout`, `add_lock_ttl`, then
`add_digest` — which is `add_lock_prefix`, `add_lock_args`, `add_lock_digest`
(`job.rb:22-28`). The order matters: `lock_args` must be on the hash before
`LockDigest` reads it.

| Key written | By | Source |
|---|---|---|
| `lock` | `LockType#call` (`lock_type.rb:33-35`) | `item["lock"]`, then the worker's `sidekiq_options`, then Sidekiq's default job options. `add_lock_type` uses `\|\|=`, so a value already on the hash is kept |
| `lock_timeout` | `LockTimeout#calculate` (`lock_timeout.rb:44-49`) | Sidekiq's default job options → `config.lock_timeout` → the worker's `lock_timeout` |
| `lock_ttl` | `LockTTL#calculate` (`lock_ttl.rb:70-78`) | `item["lock_ttl"]` → worker `lock_ttl` → `item["lock_expiration"]` → worker `lock_expiration` → `config.lock_ttl`; a `Proc` is called with `item["args"]`, a `Symbol` is sent to the worker class with `args`; `time_until_scheduled` is added for a job with `at` |
| `lock_prefix` | `Job.add_lock_prefix` | `item["lock_prefix"]` if present, else `config.lock_prefix` (`Config::PREFIX`, `"uniquejobs"`) |
| `lock_args` | `LockArgs.call` (`lock_args.rb:64-75`) | `args` verbatim unless a filter is configured |
| `lock_digest` | `LockDigest.call` (`lock_digest.rb:53-61`) | the hash below |

`LockDigest#digestable_hash` (`lock_digest.rb:65-70`) slices `class`, `queue`,
`lock_args`, `apartment`, then drops `queue` when `unique_across_queues` and
`class` when `unique_across_workers` — each read from the item *or* the worker's
options. The slice is sorted and JSON-dumped before hashing, so key order cannot
change a digest. `config.digest_algorithm` picks `OpenSSL::Digest::MD5`
(`:legacy`, the default) or `OpenSSL::Digest.new("SHA3-256", …)` (`:modern`);
the two produce different digests, so switching re-keys every lock.
`Config#digest_algorithm=` raises `ArgumentError` on anything else.

`LockArgs#lock_args_method` (`lock_args.rb:100-105`) resolves the filter in a
fixed order: `lock_args_method` or `unique_args_method` in the worker's options,
a `lock_args` class method, a `unique_args` class method, then the same two keys
in Sidekiq's default job options. A `Proc` is called with the JSON-normalised
args; a `Symbol` is sent to the worker class (unfiltered if the class does not
define it), and an `ArgumentError` from it becomes `InvalidUniqueArguments`.

## 2. The two keys

`Key` (`key.rb`) derives everything from the digest:

```text
<digest>:LOCKED       Hash   job_id => metadata JSON   — who holds the lock
uniquejobs:digests    ZSet   digest => score           — the global live index
```

`Key#to_a` (`key.rb:71-73`) returns `[locked, digests]`, which is the `KEYS`
array every lock script receives. `Key.working(identity)` (`39-41`) and
`Key.heartbeat(identity)` (`50-52`) build the `Fetch::Reliable` keys; those four
are every shape `Key` knows. Other keys in the gem are built elsewhere — the
reaper mutex (`UNIQUE_REAPER`), the metrics buckets (`LockMetrics`), the upgrade
marker (`LIVE_VERSION`).

## 3. `Locksmith` — the only thing that talks to the lock scripts

`Locksmith` (`locksmith.rb`) is constructed from the job hash and holds `key`,
`job_id`, `config` (a `LockConfig`) and `item`.

- `#lock(wait: nil)` (`locksmith.rb:54-67`) — one `call_script(:lock, key.to_a,
  lock_argv)`. On `nil` it reflects `:lock_failed`, records the metric and
  returns `nil`; on success it reflects `:locked` and returns `job_id`. The
  `wait:` keyword is declared and never read: **acquisition does not block**.
- `#lock_argv` (`161-163`) — `[job_id, config.pttl, config.type, config.limit,
  lock_metadata]`. `#lock_metadata` (`165-177`) dumps `worker, queue, limit,
  timeout, ttl, type, lock_args, time, at` as JSON; this is what the Web UI
  reads back.
- `#execute` (`75-82`) — raises `InvalidArgument` without a block, locks, and
  yields only if it got the lock. There is no `ensure unlock`: unlocking is each
  lock class's job (see [`../review/lock-model.md`](../review/lock-model.md)).
- `#unlock` / `#do_unlock` (`89-95`, `145-155`) — `call_script(:unlock,
  key.to_a, [job_id, config.type])`. Reflects `:unlocked` only when the script
  returns this job's id.
- `#delete!` (`100-103`) — the same script with the literal `"force"` in the
  lock-type slot, which is how a lock is removed regardless of `until_expired`.
  `#delete` (`108-112`) refuses when `config.pttl.positive?`.
- `#locked?` (`121-127`) — `HEXISTS <digest>:LOCKED <job_id>`, the one read that
  is not a script because it decides nothing.

## 4. The six lock classes

Every lock class descends from `Lock::BaseLock` (`lock/base_lock.rb`), two of
them through another class: `UntilExpired < UntilExecuted` and
`WhileExecutingReject < WhileExecuting`. `BaseLock` supplies `#locksmith`,
`#call_strategy`, `#unlock_and_callback`, `#callback_safely`, and
`NotImplementedError` stubs for `#lock` and `#execute`. `BaseLock#initialize`
calls `prepare_item`, which runs `Job.prepare` when `lock_digest` is missing — a
testing convenience; in production the middleware has already done it.

| `Config::LOCKS` aliases | Class | `#lock` (client) | `#execute` (server) |
|---|---|---|---|
| `until_executing`, `while_enqueued` | `UntilExecuting` | take the lock, else `call_strategy(origin: :client)` and return `nil` | unlock **first** (then `callback_safely`), run the job; on error reflect `:execution_failed` and re-take with `lock(wait: 0)` before re-raising |
| `until_executed`, `until_completed`, `until_performed`, `until_processed`, `until_successfully_completed` | `UntilExecuted` | same | `locksmith.execute { yield }`, then `unlock_and_callback`; reflects `:execution_failed` when the lock was not held, and again before re-raising any error |
| `until_expired` | `UntilExpired` | same | runs the job and never unlocks — `unlock.lua` returns early for this type, so the TTL is the only release |
| `while_executing`, `around_perform`, `while_busy`, `while_working` | `WhileExecuting` | returns the JID without locking | `:RUN` is appended to the digest in the constructor; locks, runs, unlocks in `ensure`; if the lock was not taken, reflects `:execution_failed` and `call_strategy(origin: :server)` |
| `while_executing_reject` | `WhileExecutingReject` | as above | as above, but `#server_strategy` is hard-wired to `:reject` |
| `until_and_while_executing` | `UntilAndWhileExecuting` | take the enqueue lock; `#lock` takes an `origin:` keyword the others do not | unlock the enqueue lock, then delegate to a `WhileExecuting` built from `item.dup`; if the unlock fails it reflects `:unlock_failed` and the job does not run |

`UntilExpired#lock` (`lock/until_expired.rb:20-30`) is a verbatim copy of the
`UntilExecuted#lock` it already inherits; `#execute` (`34-41`) is the same minus
the `unlock_and_callback`.

## 5. Conflict strategies

`BaseLock#call_strategy(origin:)` (`base_lock.rb:119-126`) increments `@attempt`,
picks the client or server strategy, and calls it with a block that re-locks
**once** when the strategy is `Replace` and `@attempt < 2` — the guard that
stops `replace` recursing. `strategy_for` raises `InvalidArgument` for any
origin other than `:client`/`:server`.

| Name | Class | Effect |
|---|---|---|
| `log` | `OnConflict::Log` | `log_info` naming the JID and digest |
| `raise` | `OnConflict::Raise` | raises `SidekiqUniqueJobs::Conflict` |
| `reject` | `OnConflict::Reject` | `Sidekiq::DeadSet#kill`, passing `notify_failure: false` when the installed Sidekiq's `kill` arity says it accepts options |
| `replace` | `OnConflict::Replace` | `delete_job_by_digest.lua` over the queue, schedule and retry sets; only if that deleted something does it run `Digests#delete_by_digest` and yield back for the re-lock |
| `reschedule` | `OnConflict::Reschedule` | re-pushes with `perform_in(schedule_in, *args)` (`schedule_in` from the worker's options, default 5) and `RESCHEDULED => true`, which the middleware uses to skip locking on the way back in |

`Lock::ClientValidator` rejects `:raise`, `:reject` and `:reschedule` on the
client side; `Lock::ServerValidator` rejects `:replace` on the server side.
`Lock::Validator#validate` (`lock/validator.rb:53-65`) sends `while_executing`
to the server check only, `until_executing` to the client check only, and every
other type to both.

## Related

- [`../middleware/summary.md`](../middleware/summary.md) — who calls `#lock` and `#execute`
- [`../lua-scripts/summary.md`](../lua-scripts/summary.md) — what `lock.lua` and `unlock.lua` actually do
- [`../review/lock-model.md`](../review/lock-model.md) — accepted review findings here
