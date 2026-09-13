# sidekiq-unique-jobs

Sidekiq middleware that stops the same job running twice. A job hash is reduced
to a **digest** (worker class + queue + filtered arguments, hashed), and that
digest is a lock held in Redis for as long as the chosen lock type says it
should be. The gem installs on both Sidekiq middleware chains: the client chain
takes enqueue-time locks, the server chain takes and releases execution-time
locks. It ships six lock classes under fourteen `sidekiq_options lock:` aliases
(`Config::LOCKS`), five registered conflict strategies plus a `NullStrategy`
fallback (`Config::STRATEGIES`), an orphan reaper, an optional lock-aware fetch
strategy, and a Locks tab for Sidekiq Web.

The version in `lib/sidekiq_unique_jobs/version.rb` is `9.0.0.alpha1`, and v9 is
a deliberate reduction: **a held lock is two Redis keys** — the `<digest>:LOCKED`
hash mapping `job_id` to metadata, and the one global `uniquejobs:digests`
sorted set indexing every live digest. `UpgradeLocks#upgrade_v8_to_v9`
(`upgrade_locks.rb:135-176`) deletes up to nine obsolete v8 keys per digest on
first server start — the digest STRING, `:QUEUED`, `:PRIMED`, `:INFO`, and the
same set again under the `:RUN` suffix — and `#merge_expiring_digests`
(`179-202`) folds `uniquejobs:expiring_digests` into the one ZSET. Three
invariants govern every change:

1. **A lock state transition the middleware makes is a Lua script.** Acquire,
   release, acknowledge and fetch each run start-to-finish inside Redis, so two
   racing pushes with the same digest cannot both win. `Locksmith` never does
   read-then-write in Ruby. (The Web UI's own `Lock#unlock` and `Lock#del` are
   the exception: administrative deletes, not the lock protocol.)
2. **Acquisition never blocks.** `Locksmith#lock` makes one attempt and returns
   `nil` on failure; the conflict strategy runs immediately. `lock_timeout`
   survives as metadata only.
3. **Uncertainty preserves the lock.** The reaper deletes a digest only when it
   has positively established the job is gone; every scan that runs out of
   budget, or hits queues too large to read, answers "still in use".

See [`lode-map.md`](lode-map.md) for the index.
