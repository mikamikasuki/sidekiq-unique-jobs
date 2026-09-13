# Web UI

A Sidekiq Web extension adding a **Locks** tab: browse the digests ZSET, inspect
one lock's holders, delete a lock or a single holder, and read an hour of lock
metrics.

## 1. Registration

`lib/sidekiq_unique_jobs/web.rb` is not loaded by the gem's entry point; an
application must `require "sidekiq_unique_jobs/web"`. The file ends with a
`Sidekiq::Web.configure` block registering the extension (`web.rb:60-74`),
wrapped in `rescue NameError, LoadError` so requiring it without `sidekiq/web`
present logs rather than raises.

```ruby
config.register_extension(
  SidekiqUniqueJobs::Web, name: "unique_jobs", tab: ["Locks"], index: %w[locks],
)
```

`index:` is `locks` with **no trailing slash**; the slash it used to carry made
Sidekiq 8 render `href="/locks/"` (see
[`../review/web-and-metrics.md`](../review/web-and-metrics.md)).

## 2. Routes

`Web.registered(app)` (`web.rb:10-56`) defines five GET routes. Every parameter
goes through `h` before use, and the two "delete" actions are GETs because the
views submit them as GET forms.

| Route | Does | Redirects to |
|---|---|---|
| `GET /locks` | reads `filter` (default `*`), `count` (default 100), `cursor`, `prev_cursor`; calls `Digests#page`; loads `LockMetrics.by_type(minutes: 60)`; renders `locks.erb` | — |
| `GET /locks/delete_all` | `Digests#delete_by_pattern("*", count: digests.count)` | `locks` |
| `GET /locks/:digest` | builds `SidekiqUniqueJobs::Lock.new(digest)`; renders `lock.erb` | — |
| `GET /locks/:digest/delete` | `Digests#delete_by_digest` | `locks` |
| `GET /locks/:digest/jobs/:job_id/delete` | `Lock#unlock(job_id)` | `locks/<digest>` |

## 3. Paging

`Digests#page(cursor:, pattern:, page_size:)` (`digests.rb:85-99`) issues
`ZCARD` and `ZSCAN` in one `MULTI` and returns
`[total_size, next_cursor, locks]`, where each lock is
`Lock.new(digest, time: score)`. `locks.erb:19` renders `_paging.erb` only when
`@locks.any? && @total_size > @count`, and that partial builds its links with
`Helpers#cparams` (`web/helpers.rb:63-70`), which merges the current `params`,
filters to `SAFE_CPARAMS` (`filter count cursor prev_cursor poll direction`) and
CGI-escapes each value — an unknown query parameter is dropped rather than
echoed back into a link.

Because this is `ZSCAN`, `count` is a hint: a page may come back shorter or
longer than asked, and the cursor, not the row count, says when iteration ends.

## 4. `SidekiqUniqueJobs::Lock` — the read model

`Lock` (`lock.rb`) is what the views talk to; it is *not* the middleware's lock
class (that is `SidekiqUniqueJobs::Lock::BaseLock` and its subclasses, nested
under the same constant).

- `#locked_jids(with_values: false)` (`121-123`) → `HKEYS`, or `HGETALL` when
  passed `with_values: true`, on `<digest>:LOCKED`. The views pass `true`.
- `#info` (`130-132`) memoises `#build_info` (`153-160`), which parses the
  **first** hash entry's metadata into a `LockInfoStub`, and returns an empty
  stub when the hash is empty or the value is not a JSON object. With the
  default `lock_limit` of 1 there is only one entry; at a higher limit the page
  shows the first holder's worker, queue, type, limit, ttl and args.
- `#created_at` (`106-111`) prefers the score handed in by `Digests#page`
  through `initialize`'s `time:`, and otherwise reads `"time"` out of the first
  entry's metadata, falling back to `now_f`. `lock.erb:62` renders that one
  value on every holder row rather than a per-JID timestamp.
- `#unlock(job_id)` (`75-85`) `HDEL`s the field and then, when the hash is
  empty, `ZREM`s the digest and `UNLINK`s the hash — the same pairing every
  other deletion path uses. `#del` (`92-99`) drops both keys outright.

`Helpers#display_lock_args` (`web/helpers.rb:81-92`) returns an explanatory
string for `nil` or a non-`Array`, HTML-escapes and truncates each element, and
falls back to `"Illegal job arguments: …"` if formatting raises — a malformed
payload cannot break the page.

## 5. Metrics panel

`locks.erb:1` renders `_metrics.erb` first, from `@metrics_by_type`.

`LockMetrics` (`lock_metrics.rb`) writes into per-minute Redis hashes keyed
`uniquejobs:metrics|%y%m%d|%-H:%M` (UTC), each field `"<lock_type>|<event>"`
plus a `"total|<event>"` companion, with an 8-hour `EXPIRE` (`METRICS_TTL`).
There are two writers: `LockMetrics.record` (`59-74`) fires inline from
`Locksmith` for `locked`, `lock_failed` and `unlocked`; an instance created by
`Server.start_metrics` buffers `unlock_failed` and `execution_failed` under a
mutex and `#flush` (`35-51`) pipelines them every 60 seconds — re-merging the
batch back into `@counters` if the pipeline raises, so a Redis blip costs no
counts. Both writers swallow Redis errors rather than break a lock.

`LockMetrics.by_type(minutes: 60)` (`109-119`) reads the last 60 buckets, groups
by lock type and sorts with `"total"` last. The view's four numeric columns are
Acquired (`locked`), Denied (`lock_failed`), Released (`unlocked`) and Failures
— the last being `execution_failed + unlock_failed` summed (`_metrics.erb:23`),
so neither failure kind is invisible. `EVENTS` also declares `:reaped`, which
nothing records.

## Related

- [`../lock-model/summary.md`](../lock-model/summary.md) — what writes the metadata this page reads
- [`../review/web-and-metrics.md`](../review/web-and-metrics.md) — accepted review findings here
