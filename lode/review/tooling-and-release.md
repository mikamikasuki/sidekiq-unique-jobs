Accepted review findings about the build, the benchmarks and what has to be
green before a release.

### The style gate that actually runs is RuboCop
- **Holds because:** `lint.yml` runs `bin/rubocop -P` on every push and pull request, and `release.yml` runs `bundle exec rubocop -P` before it will build the gem — so a RuboCop offence blocks a release, not just a merge. The `Rakefile` still defines a Reek task, but `reek` is absent from the `Gemfile` and there is no `.reek.yml`, so the `rescue LoadError` branch installs a stub that prints "Should be running reek" and exits 0. `rake style` therefore means RuboCop today.
- **Where:** `.github/workflows/lint.yml`, `.github/workflows/release.yml` (the `test` job), `Rakefile` (the `reek` / `rubocop` / `style` tasks)
- **Safe direction:** if Reek comes back into the `Gemfile`, the stub disappears and `rake style` starts failing on smells — treat a green `rake style` as evidence about RuboCop only.
- **Origin:** cubic learnings da685d45 / f8a4523e, rewritten against the current tree (the original said Reek must pass)

### The packaged gem contains no specs, no `myapp`, no rake tasks and no gemspec
- **Holds because:** the gemspec's file glob takes only `lib/sidekiq*`, `bin/uniquejobs`, `README`, `LICENSE` and `CHANGELOG`, and `release.yml` unpacks the built gem and fails the job if a `*.rake`, `.git*` or `*.gemspec` file, or a `spec`/`test`/`myapp` directory, survived. The glob is the intent; the unpack step is what makes it a rule rather than a hope.
- **Where:** `sidekiq-unique-jobs.gemspec` (`spec.files`), `.github/workflows/release.yml` ("Verify gem contents")
- **Proven by:** the release workflow itself; `rake build` runs the same unpack locally
- **Origin:** repository state, recorded here because it constrains where new files may live

### A release tag must match `SidekiqUniqueJobs::VERSION`
- **Holds because:** the gem is published by trusted publishing from a GitHub Release, with a Sigstore attestation over the built file. A tag that disagrees with `version.rb` would publish a version nobody can trace back to a commit. `release.yml`'s build job compares `v`-stripped `github.event.release.tag_name` against the constant and fails the job on a mismatch.
- **Where:** `.github/workflows/release.yml` ("Verify tag matches gem version"); `Rakefile`'s `release` task, which writes `version.rb` and then creates the tag
- **Origin:** repository state

### Not a bug: the benchmark scripts under `bin/` leave stale locks behind
- **Holds because:** they are throwaway measurement harnesses, not library code: each run flushes Redis between sections, so a lock left held at the end of a section is gone before the next one measures anything. They also deliberately measure one reaper pass rather than looping, because that is how the reaper actually runs.
- **Where:** `bin/benchmark`, `bin/benchmark_improvements`, `bin/compare_performance`
- **Origin:** PR #942 review threads (three, all rejected with reasons)
