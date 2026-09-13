Accepted review findings about `myapp/`, the Rails app in this repository used
to drive locks by hand while developing the gem.

### `myapp/` is a localhost-only development tool, and its security relaxations are deliberate
- **Holds because:** it exists so a maintainer can push jobs, watch digests appear and disappear, and reproduce issues against a real Sidekiq. It is never deployed, never reachable from the internet, and not packaged in the gem (`release.yml` fails the build if a `myapp` directory reaches the `.gem`). `skip_before_action :verify_authenticity_token` is there because Turbo submissions from Phlex `button_to` helpers need it when the CSRF meta tag is absent; there is no authentication on the routes for the same reason. Scanner findings about CSRF or missing auth under `myapp/` are expected, not defects.
- **Where:** `myapp/app/controllers/locks_controller.rb` (`skip_before_action`), `myapp/config/routes.rb`; `.github/workflows/release.yml` ("Verify gem contents")
- **Origin:** PR #929 review threads (four, all rejected with reasons)

### Any parameter that reaches `constantize` is checked against the `DEMO_JOBS` allowlist first
- **Holds because:** `constantize` on request input is remote code execution, and "it only runs on localhost" is not a reason to leave the pattern in a repository people read for examples. `LocksController::DEMO_JOBS` names nine job classes; `#enqueue` returns early `unless DEMO_JOBS.key?(job_name)` before constantizing, and `#load_test` constantizes only names it drew from `DEMO_JOBS` itself.
- **Where:** `myapp/app/controllers/locks_controller.rb` (`DEMO_JOBS`, `#enqueue`, `#load_test`)
- **Safe direction:** an unknown job name must be refused, never constantized to find out whether it exists.
- **Proven by:** no test
- **Origin:** PR #929, fixed in dee6874a (a follow-up scan finding on the same lines was answered as a false positive)

### Tailwind class names in the views are written out in full, never interpolated
- **Holds because:** Tailwind's JIT compiler scans source text for class names. `"text-#{color}"` produces a class that exists at runtime and in no stylesheet, so the element renders unstyled with nothing to debug. `STAT_COLORS` maps each state to a complete literal (`text-info`, `text-warning`, …) and the view fetches from it with a literal default.
- **Where:** `myapp/app/views/locks/show_view.rb` (`STAT_COLORS`)
- **Origin:** PR #929, fixed in b474e849

### Not a bug: `myapp/` request and system specs do not disable uniqueness
- **Holds because:** `SidekiqUniqueJobs.use_config(enabled: false)` belongs in specs that enqueue jobs or exercise lock behaviour. These examples assert HTTP status and rendered markup; the gem's config has no effect on them, and wrapping them implies a coupling that does not exist. The same reasoning applies in reverse to the gem's own `lock_config_spec.rb`, where the `on_conflict` examples are *about* uniqueness and must not disable it.
- **Where:** `myapp/spec/requests/`, `myapp/spec/system/`; `spec/sidekiq_unique_jobs/lock_config_spec.rb`
- **Origin:** PR #929 and PR #936 review threads (suggestions rejected with reasons)

### Not a bug: `myapp/bin/dev` exports `PORT` while `Procfile.dev` unsets it
- **Holds because:** foreman assigns an incrementing port per process when `PORT` is set, which would move the Rails server off 3000. Exporting it for the process manager and unsetting it for the web process is the pattern `cssbundling-rails` and `jsbundling-rails` generate; the two lines are cooperating, not contradicting.
- **Where:** `myapp/bin/dev`, `myapp/Procfile.dev`
- **Origin:** PR #929 review thread (concern answered with reasons)

### Not a bug: `myapp` adds to `config.autoload_paths` without also adding to `eager_load_paths`
- **Holds because:** under Zeitwerk (Rails 7+) an autoload path is eager-loaded automatically; listing it twice changes nothing.
- **Where:** `myapp/config/application.rb`
- **Origin:** PR #929 review thread (suggestion rejected with reasons)
