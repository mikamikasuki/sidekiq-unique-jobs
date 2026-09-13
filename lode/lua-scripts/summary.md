# Lua scripts

Seven scripts in `lib/sidekiq_unique_jobs/lua/` and eight partials in
`lua/shared/`. Every lock state transition the middleware makes is one of them,
so that a read and the write that depends on it cannot be interleaved.

## 1. Loading and calling

`Script::Caller#call_script(file_name, *)` (`script/caller.rb:41-52`) is mixed
into `Locksmith`, `OnConflict::Strategy`, `Redis::Entity`, `Fetch::Reliable`,
its `UnitOfWork`, and the spec-support `Testing` module. It accepts either
`(keys, argv, conn)` positionally or `keys:`/`argv:` keywords (`#extract_args`,
`81-91`), uses the caller's connection when one is given, and otherwise opens
one from `redis_pool` if the includer defines it.

`#do_call` (`56-60`) appends five values to ARGV before handing off:

```ruby
argv.dup.push(now_f, debug_lua, max_history, file_name, redis_version)
```

so **a script's own arguments occupy ARGV[1..N] and the injected ones start at
ARGV[N+1]**. `#normalize_argv` (`121-130`) then rewrites `false`/`nil` to `0`
and `true` to `1`, which is why the scripts that read the debug flag spell it
`tostring(ARGV[3]) == "1"` rather than testing truthiness.

`Script::Client#execute` (`script/client.rb:49-62`) times the call, and on a
`RedisClient::CommandError` retries down from `MAX_RETRIES` (3) through
`#handle_error` (`74-86`): `NOSCRIPT` drops the cached `Script` and reloads,
`BUSY` issues `SCRIPT KILL` first, and anything matching `LuaError::PATTERN` is
re-raised as a `Script::LuaError` carrying the offending source line with
`CONTEXT_LINE_NUMBER` (2) lines of context either side.

`Script::Scripts` caches one `Script` per name per `scripts_path` in a
`Concurrent::Map`, and `Script::Script#render_file` runs the file through
`Script::Template` — an ERB pass whose only helper is `include_partial`, which
inlines a partial **once** per `Template` instance (`template.rb:33-38`), so a
partial named twice contributes its source once.

## 2. The scripts

| Script | KEYS | ARGV (own) | Called from |
|---|---|---|---|
| `lock.lua` | `locked`, `digests` | `job_id, pttl, lock_type, limit, metadata` | `Locksmith#lock` |
| `unlock.lua` | `locked`, `digests` | `job_id, lock_type` | `Locksmith#do_unlock`, `#delete!` |
| `ack.lua` | `working`, `locked`, `digests` | `job, jid, digest, lock_type` | `Fetch::Reliable::UnitOfWork#acknowledge` |
| `fetch.lua` | `queue`, `working` | — | `Fetch::Reliable#fetch_nonblocking` |
| `delete_job_by_digest.lua` | `queue`, `schedule_set`, `retry_set` | `digest` | `OnConflict::Replace#delete_job_by_digest` |
| `recover.lua` | `working_pattern` | `current_identity` | nothing — recovery is Ruby (`Fetch::Reliable#recover_orphans`) |
| `find_digest_in_queues.lua` | `digest` | — | nothing |

Only `delete_job_by_digest.lua` and `find_digest_in_queues.lua` name
`include_partial`; between them they pull in `_common.lua`,
`_delete_from_queue.lua`, `_delete_from_sorted_set.lua` and
`_find_digest_in_queues.lua`. No partial includes another. The other four —
`_current_time.lua`, `_find_digest_in_process_set.lua`,
`_find_digest_in_sorted_set.lua`, `_hgetall.lua` — are included by nothing.

## 3. `lock.lua` — acquire or refuse

```text
HEXISTS locked job_id == 1        -> return job_id        (re-entrant: already ours)
HLEN locked >= limit              -> return nil           (bounded concurrency)
HSET locked job_id metadata
ZADD digests score digest         score = current_time + pttl when lock_type is
                                          until_expired AND pttl > 0,
                                          current_time otherwise
PEXPIRE locked pttl               only when pttl > 0
return job_id
```

The digest is not passed in: the script derives it with
`string.gsub(locked, ":LOCKED$", "")`. Two properties follow from the shape.
The same `job_id` locking twice succeeds without double-counting against
`limit`, which is what makes a retried server lock idempotent. And an
`until_expired` digest with a TTL is scored by its *expiry*, so the reaper's
`byscore(0, max_score)` window and the Web UI's ordering treat it as if it were
created at the moment it will lapse.

## 4. `unlock.lua` — release, force, or decline

```text
lock_type ~= "force" and lock_type == "until_expired" -> return job_id   (no-op)
not HEXISTS locked job_id:
    HLEN locked == 0 -> ZREM digests digest; return job_id   (tidy a stale index row)
    otherwise        -> return nil                           (someone else holds it)
HDEL locked job_id
HLEN locked == 0 -> ZREM digests digest; UNLINK locked
return job_id
```

The first line is why `until_expired` never releases early, and why
`Locksmith#delete!` passes the literal `"force"` in the lock-type slot. The
second block means a returned `job_id` does **not** prove this caller held the
lock: an already-empty hash also returns it, so the ZSET row is cleaned up
rather than left behind. Note the early-return branch does not `UNLINK` the
(empty) hash; the full path does.

## 5. `ack.lua` — leave the working list, then maybe unlock

`LREM working 1 job` always runs. It then returns early — leaving the lock
alone — when the digest is blank, when `lock_type` is `until_expired`, or when
it is `while_executing` / `until_and_while_executing`, because those types
release through their own path. Otherwise, **only if `HEXISTS locked jid`**, it
`HDEL`s this JID and, if the hash is now empty, `ZREM`s the digest and `UNLINK`s
the hash. It always returns `1`.

## 6. `fetch.lua` — move and report, never drop

`LMOVE queue working RIGHT LEFT` moves one job, and the script returns `nil` if
the queue was empty. Otherwise it decodes with `pcall(cjson.decode, …)` and
returns `{job, -1}` when the payload will not parse or carries no
`lock_digest`; when it parses but has no `jid`, `{job, 0}`; otherwise
`{job, HEXISTS <digest>:LOCKED jid}`. The caller treats `0` as "uniqueness
lapsed" and reflects, but whenever a job was moved it is returned — a missing
lock never costs a job.

## Related

- [`../lock-model/summary.md`](../lock-model/summary.md) — `Locksmith` and the lock classes
- [`../middleware/summary.md`](../middleware/summary.md) — `Fetch::Reliable`
- [`../practices.md`](../practices.md) — ARGV indexing and scan-to-cursor rules
