Accepted review findings about `Orphans::Reaper`, `Server`'s background work and
`Fetch::Reliable`, rewritten as rules about the system.

### Every reaper predicate that can run out of information answers "the job is still there"
- **Holds because:** deleting a live lock lets a duplicate job run — the one failure the gem exists to prevent. Keeping a dead lock costs one key until the next pass. So `enqueued?` returns `true` the moment `queues_very_full?` counts more than `MAX_QUEUE_LENGTH` (1000) jobs, because it cannot afford to read them; `in_sorted_set?`, `active?` and `digest_in_queue?` each return `true` on `timeout?`, checked before the scan and again inside every loop, so a budget that runs out mid-scan preserves the lock rather than reaping on a partial read. `locked?` is the only predicate that may answer "orphan", and it does so on positive knowledge: `HLEN <digest>:LOCKED` is zero.
- **Where:** `lib/sidekiq_unique_jobs/orphans/reaper.rb#belongs_to_job?`, `#locked?`, `#in_sorted_set?`, `#enqueued?`, `#active?`, `#digest_in_queue?`, `#queues_very_full?`
- **Safe direction:** answer "keep" on any doubt.
- **Proven by:** `spec/sidekiq_unique_jobs/orphans/reaper_spec.rb:"preserves locks when queues are very full (fails closed)"`, `:"removes digests with no LOCKED hash"`; the mid-scan timeout returns have no test
- **Origin:** PR #946, fixed in ac0d26e9 and b2347b35

### `digest_in_queue?` clamps its `LRANGE` window to zero and re-reads `LLEN` every page
- **Holds because:** the queue is live: jobs pop while the reaper reads it. Without the clamp, `(page * PAGE_SIZE) - deleted_size` goes negative once enough jobs have left, and a negative `LRANGE` start counts from the tail — the scan silently re-reads the wrong end of the list and can miss the digest it is looking for, reaping a live lock.
- **Where:** `lib/sidekiq_unique_jobs/orphans/reaper.rb#digest_in_queue?` (`range_start = [(page * PAGE_SIZE) - deleted_size, 0].max`, `deleted_size = [initial_size - current_size, 0].max`)
- **Proven by:** no test drives a queue shrinking mid-scan
- **Origin:** PR #946, fixed in ac0d26e9

### Redis work inside the reaper uses the connection it was handed, not the `Redis::*` wrappers
- **Holds because:** `Orphans::Reaper#call` opens one connection and threads it through `execute` → `belongs_to_job?` → every predicate. `Redis::List`, `Redis::SortedSet` and friends each wrap their calls in `redis { |conn| … }`, which checks out a *second* connection from the pool. Inside a reaper pass that means interleaving two views of the same keyspace and holding two connections per worker process for no gain. The same reason is why `delete_orphans` calls `BatchDelete.call(orphans, conn)` with the connection rather than letting `BatchDelete` open its own.
- **Where:** `lib/sidekiq_unique_jobs/orphans/reaper.rb#call`, `#execute`, `#delete_orphans`; `lib/sidekiq_unique_jobs/batch_delete.rb#call`
- **Origin:** PR #946 review thread (wrapper-class suggestion rejected with reasons); `delete_orphans` fixed in ac0d26e9

### Deletion of many digests goes through `BatchDelete`, not a loop of individual deletes
- **Holds because:** a reaper pass can produce `reaper_count` (default 1000) digests, each needing an `UNLINK` and a `ZREM`. `BatchDelete#batch_delete` slices them into `BATCH_SIZE` (500) chunks and pipelines each chunk, so a full pass is two round trips rather than two thousand, and the two keys of a digest always travel together.
- **Where:** `lib/sidekiq_unique_jobs/batch_delete.rb#batch_delete`, called from `Orphans::Reaper#delete_orphans` and `Digests#delete_by_pattern`
- **Proven by:** `spec/sidekiq_unique_jobs/batch_delete_spec.rb`
- **Origin:** PR #946, fixed in ac0d26e9

### A `SCAN`/`ZSCAN` loop ends on the cursor, never on an empty page
- **Holds because:** Redis may return zero keys for a `SCAN` iteration that is not finished — `COUNT` is a hint about work done, not results returned. Breaking on an empty batch leaves working lists unrecovered and expiring digests unmerged, at random, under load. Every loop reads `cursor, values = conn.call("SCAN", …)` and breaks only on `cursor == "0"`.
- **Where:** `lib/sidekiq_unique_jobs/fetch/reliable.rb#recover_orphans`; `lib/sidekiq_unique_jobs/upgrade_locks.rb#batch_scan`, `#merge_expiring_digests`; `lib/sidekiq_unique_jobs/orphans/reaper.rb#in_sorted_set?`
- **Proven by:** `spec/sidekiq_unique_jobs/upgrade_locks_spec.rb:"merges expiring_digests into digests and removes the old ZSET"`
- **Origin:** PR #942, fixed in c2e87ded

### A fetcher's identity carries a random nonce, not just hostname and PID
- **Holds because:** `uniquejobs:working:<identity>` is claimed for the life of a process and recovered by whoever finds it with a dead heartbeat. A recycled PID on the same host would otherwise adopt a previous run's working list as its own and never recover it. `"#{Socket.gethostname}:#{Process.pid}:#{SecureRandom.hex(6)}"` — the same shape Sidekiq Pro's SuperFetch uses.
- **Where:** `lib/sidekiq_unique_jobs/fetch/reliable.rb#initialize`; `lib/sidekiq_unique_jobs/key.rb.working`, `.heartbeat`
- **Proven by:** `spec/sidekiq_unique_jobs/fetch/reliable_spec.rb:"Key.working returns a uniquejobs-namespaced key"` covers the key shape, not the nonce
- **Origin:** PR #942, fixed in c751b7b2

### `bulk_requeue` stops the heartbeat thread before it returns
- **Holds because:** shutdown must let the heartbeat key lapse so another process can recover anything left behind. A thread still refreshing `uniquejobs:heartbeat:<identity>` every 20 seconds keeps a dead process looking alive. `#bulk_requeue` sets `@done = true` — which the thread's loop checks — and `join(1)`s it before requeuing.
- **Where:** `lib/sidekiq_unique_jobs/fetch/reliable.rb#bulk_requeue`, `#start_heartbeat`
- **Proven by:** no test
- **Origin:** PR #942, fixed in c751b7b2

### Orphan recovery pushes jobs back with `RPUSH`
- **Holds because:** Sidekiq enqueues with `LPUSH` and fetches from the right, so the tail holds the oldest job. A recovered job was enqueued before everything now in the queue; `RPUSH` puts it at the tail, where it is taken next, instead of behind every job pushed while the process was down. (`lua/recover.lua`, which nothing calls, still `LPUSH`es — do not copy it.)
- **Where:** `lib/sidekiq_unique_jobs/fetch/reliable.rb#recover_orphans`
- **Proven by:** `spec/sidekiq_unique_jobs/fetch/reliable_spec.rb:"requeues jobs from dead process working lists"` asserts recovery, not ordering
- **Origin:** PR #942, fixed in c751b7b2

### `Server`'s background methods rescue `StandardError => ex` and log; none of them raise
- **Holds because:** they run inside Sidekiq's `:startup` hook and inside `TimerTask` threads. An exception there takes down boot or silently kills the timer. `Server.reap` logs the class and message and returns `0`; the mutex helpers rescue and continue, with `reaper_registered?` returning `true` on failure so a process that cannot read Redis never concludes the reaper is dead and steals the role. A bare `rescue` that swallows the exception object is not enough — the message is the only diagnostic a user gets.
- **Where:** `lib/sidekiq_unique_jobs/server.rb.reap`, `.register_reaper_process`, `.refresh_reaper_mutex`, `.reaper_registered?`, `.release_reaper_mutex`, `.resurrect_reaper`
- **Proven by:** `spec/sidekiq_unique_jobs/server_spec.rb:"returns the count of reaped digests"`
- **Origin:** PR #946, fixed in ac0d26e9

### Not a bug: `recover_orphans` runs synchronously in the constructor, so its specs need no waiting
- **Holds because:** only the heartbeat is a background thread. `#initialize` calls `start_heartbeat` and then `recover_orphans` inline, so by the time `Fetch::Reliable.new(capsule)` returns, recovery has finished. A spec that sleeps or polls for it would be hiding a race that does not exist.
- **Where:** `lib/sidekiq_unique_jobs/fetch/reliable.rb#initialize`
- **Proven by:** `spec/sidekiq_unique_jobs/fetch/reliable_spec.rb:"requeues jobs from dead process working lists"`
- **Origin:** PR #942 review thread (suggestion rejected with reasons)

### Not a bug: `Fetch::Reliable` gets its Redis access from the capsule, not from its own connection code
- **Holds because:** `#initialize` stores the Sidekiq 8 capsule as `@config`, and `include Sidekiq::Component` supplies `redis` and `logger` off that. There is nothing for the gem to wire up.
- **Where:** `lib/sidekiq_unique_jobs/fetch/reliable.rb` (`include Sidekiq::Component`, `@config = capsule`)
- **Origin:** PR #942 review thread (suggestion rejected with reasons)
