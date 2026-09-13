Accepted review findings about `Locksmith`, the lock classes and the Web UI's
`Lock` read model, rewritten as rules about the system.

### `Locksmith#execute` does not unlock in an `ensure`; each lock class owns its release
- **Holds because:** `#execute` is a primitive — lock, yield if locked. `UntilExecuted` deliberately leaves the lock held when the block raises, so the job's retry cannot run a duplicate; `WhileExecuting` unlocks in its own `ensure`; `UntilExpired` never unlocks at all. An `ensure locksmith.unlock` inside `#execute` would release the lock for every one of them and silently break the retry contract.
- **Where:** `lib/sidekiq_unique_jobs/locksmith.rb#execute`; `lock/until_executed.rb#execute`, `lock/while_executing.rb#execute`, `lock/until_expired.rb#execute`
- **Proven by:** `spec/support/shared_examples/an_executing_lock_implementation.rb:"keeps being locked when an error is raised"`, included by `spec/sidekiq_unique_jobs/lock/until_executed_spec.rb`
- **Origin:** PR #942 review thread (suggestion rejected with reasons)

### `Locksmith#delete!` passes the literal `"force"` in the lock-type slot
- **Holds because:** `unlock.lua`'s first line returns early for `until_expired`, so an ordinary unlock cannot remove such a lock. Deleting keys from Ruby instead would leave the digest in the ZSET or the hash behind. `"force"` is not a lock type; it is a sentinel the script tests before the `until_expired` check, so the delete goes through the same HDEL + ZREM + UNLINK path as any other release.
- **Where:** `lib/sidekiq_unique_jobs/locksmith.rb#delete!`; `lib/sidekiq_unique_jobs/lua/unlock.lua:7`
- **Proven by:** `spec/sidekiq_unique_jobs/locksmith_spec.rb:"disappears without a trace when calling delete!"`
- **Origin:** PR #942, fixed in c2e87ded

### Every connection-taking method on `Locksmith` goes through `redis(redis_pool)`
- **Holds because:** the middleware is handed a `redis_pool` by Sidekiq and the lock must be read on the same pool it was written on. A bare `redis` call checks out of the default pool, which in a multi-Redis or namespaced setup is a different server. `#unlock` and `#locked?` both branch on an explicitly passed `conn` first, then fall back to `redis(redis_pool)`.
- **Where:** `lib/sidekiq_unique_jobs/locksmith.rb#unlock`, `#locked?`
- **Proven by:** no test; the pool is `nil` throughout the suite
- **Origin:** PR #942, fixed in c751b7b2

### `Lock#unlock` removes the ZSET row and the hash once the last holder is gone
- **Holds because:** the Web UI's "Unlock" button `HDEL`s one JID. Without the follow-up the digest stays in `uniquejobs:digests` with an empty `:LOCKED` hash behind it, and the Locks page keeps listing a lock nobody holds until the reaper's next pass. Both keys go together everywhere else, and this path is no exception.
- **Where:** `lib/sidekiq_unique_jobs/lock.rb#unlock`
- **Proven by:** `spec/sidekiq_unique_jobs/lock_spec.rb:"removes the job_id from the LOCKED hash"` asserts only the HDEL; the ZREM + UNLINK half has no test (`"#del"` covers the same pairing on the other path)
- **Origin:** PR #942, fixed in c2e87ded

### `Lock#created_at` prefers the score the caller already read over a second Redis round trip
- **Holds because:** `Digests#page` gets every digest's score from the same `ZSCAN` that produced the row, and passes it as `time:`. Re-reading the metadata to recover a timestamp the caller already has costs one `HGETALL` per row on a page of a hundred. `initialize` stores a non-zero `time:` in `@created_at`, and `#created_at` returns it before touching Redis.
- **Where:** `lib/sidekiq_unique_jobs/lock.rb#initialize`, `#created_at`; `lib/sidekiq_unique_jobs/digests.rb#page`
- **Proven by:** `spec/sidekiq_unique_jobs/lock_spec.rb:"returns the timestamp from lock metadata"` covers only the fallback
- **Origin:** PR #942, fixed in c2e87ded

### `VersionCheck` wraps `Gem::Requirement`; it parses no version syntax of its own
- **Holds because:** a hand-rolled constraint regex is a parser with no grammar table behind it, and it was wrong on forms `Gem::Requirement` handles natively. `.satisfied?` splits the constraint string on `&&` and `,`, falls back to splitting on space-before-operator, and hands the parts to `Gem::Requirement`. The only string work left is deciding where one constraint ends and the next begins.
- **Where:** `lib/sidekiq_unique_jobs/version_check.rb.satisfied?`, `.split_space_separated`
- **Safe direction:** an unparseable constraint must raise from `Gem::Requirement`, not quietly match everything — a version gate that always passes is invisible.
- **Proven by:** `spec/sidekiq_unique_jobs/version_check_spec.rb` (`"when given dual constraints with space"`, `"…with &&"`)
- **Origin:** PR #942, fixed in c2e87ded (the `Gem::Requirement` wrapper) and 8b540ebe (the last regex removed from the splitter)

### Not a bug: `Lock#info` and the lock detail page describe only the first holder
- **Holds because:** with the default `lock_limit` of 1 the LOCKED hash has exactly one entry, and parsing every entry's JSON in ERB to show per-JID metadata buys nothing for the overwhelming majority of locks. `build_info` takes `entries.values.first`; `lock.erb:62` renders the single `@lock.created_at` on every holder row. Accepted as a known limitation for `lock_limit > 1`, to revisit if multi-lock users report it.
- **Where:** `lib/sidekiq_unique_jobs/lock.rb#build_info`, `#created_at`; `lib/sidekiq_unique_jobs/web/views/lock.erb:62`
- **Proven by:** no test asserts the multi-holder rendering
- **Origin:** PR #942 review threads (two, both answered with reasons)

### Not a bug: the gemspec's `sidekiq < 10.0.0` upper bound is deliberately wide
- **Holds because:** the CI matrix tests 8.0 and 8.1 because those are the released 8.x lines, not because the gem is known to break on 8.2. Narrowing the constraint to the tested versions would lock users out of every future 8.x patch on the day it ships. The bound exists to exclude a hypothetical Sidekiq 10, not to mirror the matrix.
- **Where:** `sidekiq-unique-jobs.gemspec` (`add_dependency "sidekiq", ">= 8.0.0", "< 10.0.0"`); `.github/workflows/rspec.yml`
- **Origin:** PR #942 review thread (suggestion rejected with reasons)
