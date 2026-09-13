Accepted review findings about the Sidekiq Web extension, `LockMetrics` and the
reflection plumbing behind them.

### The Locks tab registers `index: %w[locks]` with no trailing slash
- **Holds because:** Sidekiq 8 builds the tab's `href` straight from the `index:` entry, so `%w[locks/]` rendered `href="/locks/"`, which does not match the `app.get "/locks"` route the extension defines and left the tab link broken.
- **Where:** `lib/sidekiq_unique_jobs/web.rb` (the `Sidekiq::Web.configure` block); the routes in `Web.registered`
- **Proven by:** `spec/sidekiq_unique_jobs/web_spec.rb:"registers the locks tab without a trailing slash"`, `:"renders the locks navigation without a trailing slash"`
- **Origin:** PR #961

### `LockMetrics#flush` puts its batch back when the pipeline fails
- **Holds because:** `#reset` empties `@counters` under the mutex *before* the pipeline runs, so a Redis blip during the 60-second flush would otherwise drop a whole minute of counts with no trace. The `rescue StandardError` merges the unflushed batch back into `@counters` under the mutex, and the next flush carries it.
- **Where:** `lib/sidekiq_unique_jobs/lock_metrics.rb#flush`, `#reset`
- **Safe direction:** double-counting on a retry is better than losing counts; these are observability numbers, not lock state.
- **Proven by:** no test — `spec/sidekiq_unique_jobs/lock_metrics_spec.rb` covers the happy path and the empty case only
- **Origin:** PR #942, fixed in c751b7b2

### The metrics table's Failures column sums `execution_failed` and `unlock_failed`
- **Holds because:** the panel has one column for failures and two kinds of failure are recorded. Showing only one makes the other invisible on the page that exists to surface them, and they are recorded by different writers — `unlock_failed` from `BaseLock#unlock_and_callback`, `execution_failed` from the lock classes.
- **Where:** `lib/sidekiq_unique_jobs/web/views/_metrics.erb:23`; `lib/sidekiq_unique_jobs/server.rb.start_metrics`
- **Proven by:** no test asserts the rendered column
- **Origin:** PR #942, fixed in c751b7b2

### Not a bug: `SidekiqUniqueJobs.reflect` can be called more than once without clobbering
- **Holds because:** `Reflections` stores one block per event name (`@reflections[:<event>] = block`), so a second `reflect` block registering `unlock_failed` replaces only that key. `Server.start_metrics` registering two events does not disturb an application's own reflections for the other twelve. What it does *not* do is chain two blocks for the same event — the last registration for a given name wins.
- **Where:** `lib/sidekiq_unique_jobs/reflections.rb` (the `REFLECTIONS.each` `class_eval`), `#dispatch`; `lib/sidekiq_unique_jobs/server.rb.start_metrics`
- **Proven by:** `spec/sidekiq_unique_jobs/reflections_spec.rb`
- **Origin:** PR #942 review thread (concern answered with reasons)

### Not a bug: the `/locks` route calls `LockMetrics.by_type` directly and does not need `Server.metrics`
- **Holds because:** `.by_type` and `.query` are class methods that read the minute buckets straight out of Redis. They hold no state and do not depend on the buffering instance `Server.start_metrics` creates, so the Web UI renders the panel in a process that never booted the server hooks — a web-only dyno, for instance.
- **Where:** `lib/sidekiq_unique_jobs/lock_metrics.rb.by_type`, `.query`; `lib/sidekiq_unique_jobs/web.rb` (`GET /locks`)
- **Proven by:** `spec/sidekiq_unique_jobs/lock_metrics_spec.rb:"groups by lock type"`, `:"sorts total last"`
- **Origin:** PR #942 review thread (concern answered with reasons)

### Not a bug: the metrics specs do not freeze time
- **Holds because:** the buckets are minute-granular and the code under test is precisely the bucket arithmetic — `flush` writes at `Time.now`, `query(minutes: 1)` reads the current minute back. Freezing the clock would make the examples pass whether or not the bucket key were computed correctly, which is the only thing they exist to check.
- **Where:** `lib/sidekiq_unique_jobs/lock_metrics.rb#flush`, `.query`; `spec/sidekiq_unique_jobs/lock_metrics_spec.rb`
- **Origin:** PR #942 review thread (suggestion rejected with reasons)
