# Practices

Patterns this codebase holds to that `../.claude/rules/` does not already state.
Style, file size, error handling and the TDD loop live in
[`../.claude/rules/coding-style.md`](../.claude/rules/coding-style.md) and
[`../.claude/rules/testing.md`](../.claude/rules/testing.md); this file is the
rest.

## Redis correctness

- **A state transition is one script.** Anything that reads lock state and then
  writes based on what it read belongs in `lib/sidekiq_unique_jobs/lua/`.
  `Locksmith` holds no read-modify-write: `#lock` and `#do_unlock` each make a
  single `call_script`. Ruby-side `multi`/`pipelined` is for unconditional
  writes only (`Digests#delete_by_digest`, `BatchDelete#batch_delete`,
  `Lock#del`). `Lock#unlock` (`lock.rb:75-85`) is the one read-then-write left,
  and it is an operator action from the Web UI, not part of the lock protocol.
- **A script's own ARGV comes first.** `Script::Caller#do_call` appends five
  injected values, so a script that takes N arguments reads `current_time` at
  `ARGV[N+1]`. `lock.lua` takes five and reads `ARGV[6]`;
  `delete_job_by_digest.lua` takes one and reads `ARGV[2]`;
  `find_digest_in_queues.lua` takes none and still reads `ARGV[2]`, because
  `call_script` is what supplies them. Changing a script's argument count moves
  every injected index in that file.
- **Booleans and nil never reach Redis.** `Script::Caller#normalize_argv`
  (`script/caller.rb:121-130`) maps `false`/`nil` to `0` and `true` to `1`,
  which is why the two scripts that take the debug flag spell it
  `tostring(ARGV[3]) == "1"` (`delete_job_by_digest.lua:13`,
  `find_digest_in_queues.lua:7`) before `_common.lua`'s `log_debug` tests
  `debug_lua ~= true`.
- **Dropping the last holder means dropping the ZSET row too.**
  `Digests#delete_by_digest`, `Lock#del`, `Lock#unlock`, `BatchDelete` and the
  tails of `unlock.lua` and `ack.lua` each `ZREM uniquejobs:digests <digest>`
  in the same breath as they remove `<digest>:LOCKED`. `unlock.lua`'s
  stale-index branch is the one that `ZREM`s without an `UNLINK`, and only
  because the hash it would unlink is already empty — Redis deletes a hash when
  its last field goes. Removing one without the other leaves either an
  unreachable lock or a phantom row in the Web UI.
- **Scan to `"0"`.** Every `SCAN`/`ZSCAN` loop exits on the cursor, never on an
  empty batch: Redis may legitimately return zero keys mid-iteration.
  `UpgradeLocks#batch_scan` (`upgrade_locks.rb:122-131`),
  `#merge_expiring_digests` (`179-202`), `Fetch::Reliable#recover_orphans`
  (`fetch/reliable.rb:174-210`) and `Orphans::Reaper#in_sorted_set?`
  (`orphans/reaper.rb:94-111`) are the shape to copy.

## Failing safe

- **When the reaper cannot tell, it keeps the lock.** Every predicate under
  `Orphans::Reaper#belongs_to_job?` that can run out of information returns
  `true` — timeout budget spent, queues past `MAX_QUEUE_LENGTH`. (`locked?` is
  the one that returns `false`: an empty LOCKED hash is positive knowledge that
  nothing holds the lock.) Deleting a live lock lets a duplicate run; keeping a
  dead one costs a key until the next pass.
- **Background work swallows its own errors.** `Server.reap` logs and returns
  `0`; `#register_reaper_process`, `#refresh_reaper_mutex`,
  `#release_reaper_mutex`, `#reaper_registered?` and `#resurrect_reaper` each
  rescue and continue, with `reaper_registered?` returning `true` on failure so
  a blind process does not steal the reaper role. Nothing in `Server` may raise
  into Sidekiq's startup or timer threads.
- **The after-unlock callback never re-raises.** `BaseLock#callback_safely`
  (`lock/base_lock.rb:134-144`) reflects, warns and returns the JID, because the
  lock is already gone and a retry would run the job twice.
- **Fetch never drops a job it has moved.** `fetch.lua` returns the job with a
  `-1` or `0` validity flag rather than swallowing it; `UnitOfWork#acknowledge`
  (`fetch/reliable.rb:233-253`) falls back to a bare `LREM` on any error;
  `Fetch::Reliable#validate_lock` rescues and continues.

## Configuration and compatibility

- **Add a config key in three places.** The `Concurrent::MutableStruct` member
  list (`config.rb:5-29`), a `SCREAMING_CASE` default constant
  (`config.rb:96-150`), and the positional argument in `Config.default`
  (`config.rb:196-221`) — all in `lib/sidekiq_unique_jobs/config.rb`, and the
  positions must line up.
- **Aliases are additive.** A new name for an existing lock goes in the matching
  `LOCKS_*` hash; renaming or removing an alias breaks `sidekiq_options` in
  applications that upgrade. Applications add their own with
  `Config#add_lock` / `#add_strategy`, which raise `DuplicateLock` /
  `DuplicateStrategy` rather than overwrite.
- **Deprecated keys are recognised, not accepted.** `unique_args` and
  `unique_prefix` are still read by `LockDigest#initialize`, and
  `unique_args_method` by `LockArgs#lock_args_method`.
  `Lock::Validator::DEPRECATED_KEYS` (`lock/validator.rb:13-18`) lists four —
  `unique`, `unique_args`, `lock_args`, `unique_prefix` — and turns each into a
  validation error naming its replacement. `unique` itself has no reader outside
  that map.

## Dead weight to leave alone (or remove deliberately)

Several artefacts survive with no live caller. Do not build on them, and do not
read a grep hit there as evidence of a live path; confirm a Ruby caller first.

- `lua/recover.lua` and `lua/find_digest_in_queues.lua` — no `call_script`
  names them. Recovery is Ruby (`Fetch::Reliable#recover_orphans`), and it
  `RPUSH`es where `recover.lua` `LPUSH`es.
- Four of the eight `lua/shared/` partials — `_current_time.lua`,
  `_find_digest_in_process_set.lua`, `_find_digest_in_sorted_set.lua`,
  `_hgetall.lua` — are named by no `include_partial`.
- `UntilAndWhileExecuting#ensure_relocked` (`until_and_while_executing.rb:58-66`),
  whose only reference is the commented-out call above it.
- `UpgradeLocks`'s v6 path — `#upgrade_v6_locks`, `#upgrade_v6_lock`,
  `#delete_unused_v6_keys`, `#delete_supporting_v6_keys`, `#delete_suffix` and
  `OLD_SUFFIXES`. `#call` runs only `upgrade_v8_to_v9` and
  `merge_expiring_digests`.
- `Config#locksmith_executor` builds a bounded `ThreadPoolExecutor` that nothing
  submits work to; only `#shutdown_executor`, called from
  `SidekiqUniqueJobs.reset!`, still touches it.
- `Unlockable#unlock!` is byte-identical to `#unlock` — the bang does not force
  anything. `#delete!` is the one that forces.
