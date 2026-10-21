# Design: Reload daemons in development (#373)

## Problem

`Rage::Daemon` subclasses only pick up code changes after a full server restart. Controllers and other `app/` classes already reload via `Rage::CodeLoader` + Zeitwerk. Daemons should behave the same way in development.

## Why this isn't "just reuse the reloader"

Daemon source files (`app/daemons/*.rb`) are already inside `Rage::CodeLoader`'s autoload path, so `Zeitwerk` already unloads/reloads the constant on `CodeLoader#reload`. Two gaps remain:

1. **Stale fiber.** Each daemon runs in a long-lived `Fiber` created by `Rage::Daemon.__execute`, which closes over the *class object* at boot time (`instance = new`, called on `self`, i.e. the original klass). After Zeitwerk reloads, `EventConsumer` the constant now points to a brand-new class object, but the running fiber's `self` is still the old, unloaded one. The loop keeps calling `new` on dead code — nothing changes.
2. **No trigger without HTTP traffic.** `Rage::CodeLoader#reload` currently only runs from `Rage::Reloader`, HTTP middleware that fires per-request (`lib/rage/middleware/reloader.rb`). A dev app whose only job is running daemons (no controllers being hit) would never call `check_updated!`/`reload` at all.

## Proposed design

### 1. Detect changes independent of HTTP traffic

Register a development-only timer via `Iodine.run_every(interval_ms)` (pattern already used elsewhere per `CONTRIBUTING.md`) that calls `Rage.code_loader.check_updated!` / `reload` on a short interval (e.g. 500ms–1s). This reuses the *existing* `CodeLoader` mechanism (per the issue's own constraint) rather than adding a separate file watcher — it just adds a second trigger source alongside the per-request one. Gated to `Rage.env.development?`, same as the reloader's `enable_reloading` flag, so there's zero footprint in production.

### 2. Track daemon source files individually

`CodeLoader#check_updated!` currently reports one aggregate boolean for the whole watched set. For daemons we need to know *which* daemon(s) changed, not just "something changed" — unrelated controller edits shouldn't restart daemons (Lean Happy Path / feature isolation principle in `ARCHITECTURE.md`).

Add a small per-daemon mtime snapshot, keyed by source file:
- At `__perform` time, resolve the file via `Object.const_source_location(name)` and record `{file => mtime}`.
- On each check, compare current mtime per tracked file; only daemons whose own file changed are candidates for restart.

### 3. Stop-and-restart sequence

For each daemon class with a changed file:
1. **Resolve the fresh constant first, before touching the running instance.** Call `Rage.code_loader.reload` (Zeetwerk swap) and then re-resolve `Object.const_get(klass.name)`. If this raises (syntax error, load-time exception), log it clearly via `Rage.logger.error` + `Rage::Errors.report`, leave the **old** daemon instance running untouched, and retry on the next detected change. This satisfies the issue's "don't silently disappear / fall back to previous version" requirement.
2. **Only if the reload succeeded**, interrupt the old daemon's fiber. Each daemon's `__execute` already has the shutdown pattern (`Iodine.on_state(:start_shutdown) { @__stopping = true; f.kill }`) — generalize this to a per-class `@__fiber` reference so a reload can kill just that one fiber (`@__stopping = true; f.kill`) without touching other daemons or the server shutdown path. Killing the fiber unblocks it even mid-`subscribe`/`sleep`, same as today's shutdown behavior. The existing `ensure __cleanup(instance)` on the ongoing `perform` call handles calling the user's `cleanup`.
3. **Update the registry.** `Rage.config.daemons.klasses` holds the class *reference* registered at boot (`config.daemons << RedisListener`). After Zeetwerk reload this reference is stale (points to the unloaded class). Replace the entry in-place with the freshly resolved constant.
4. **Restart.** Call `__perform` on the resolved class, same code path used at boot. Respects existing `scope :worker`/`:node` semantics unchanged — `:node` daemons still use `Rage::Internal.pick_a_worker`, but since the node-scope lock file is already held by this worker process from the first `__perform`, restart happens in the same process without needing to fight over the lock. *(Open question — see below.)*

### 4. Where this lives

- `Rage::CodeLoader` keeps its general-purpose job (reload the autoloader + components); it doesn't need to know about daemons.
- All daemon-specific restart logic lives in `lib/rage/daemon.rb`, registered via the existing `Rage.config.after_reload` hook (`CodeLoader#reload` already calls `Rage.config.run_hooks_for(:after_reload)`) — so daemon restart naturally participates whether the reload was triggered by the HTTP middleware or the new timer.

## Open questions for review

1. **`pick_a_worker` + restart re-entrancy.** `pick_a_worker` grabs an flock'd lock file once. On restart within the same process this is fine (no second `pick_a_worker` call needed — we just re-invoke `__execute`-equivalent directly). Need to double check we don't re-acquire/re-validate the lock on each restart, since re-running `__perform` as currently defined re-enters `pick_a_worker` and could behave oddly with an already-held lock from the same process (should be a non-blocking no-op re-acquire, but worth a direct test).
2. **Timer interval.** Proposing 500ms default; should this be configurable (`config.code_loader.daemon_reload_interval`) or hardcoded alongside the existing watch-glob cost (`check_updated!` already globs the full `app/` tree — on a big app this might be too expensive to run every 500ms independent of request traffic; may want to only glob `app/daemons/**` for the timer-driven check instead of the full autoload tree).
3. **Scope of "changed file."** If a daemon's `perform` calls into a plain PORO it doesn't own (e.g. `app/services/event_processor.rb`), that class already reloads via Zeetwerk on next *invocation* seen by callers — restarting the daemon itself isn't required for that case, only for changes to the daemon class file itself. Confirming this matches the issue's literal ask ("daemon's source file"), not transitive dependencies.

## Test plan

- `spec/daemon_spec.rb`: unit coverage for the new restart path — mock `Object.const_source_location`/mtime, assert fiber kill + `__perform` re-invocation, assert failure-to-reload keeps the old instance alive and logs.
- `spec/integration/daemons_spec.rb`: extend the existing integration harness to touch a daemon file in `spec/integration/test_app/app/daemons/*.rb`, wait, and assert new behavior is observed without a server restart (mirrors how controller reload is already integration-tested, if such a spec exists for controllers — need to check).
- Regression: existing crash/backoff restart path (`rescue => e ... break if @__stopping`) must stay unaffected — reload-triggered restart is a different path, add a test asserting it doesn't trip the backoff/exponential-sleep logic.

## Non-goals

- No change to production behavior — the whole mechanism is gated behind `Rage.env.development?`, matching `CodeLoader`'s existing `enable_reloading` gate.
- No new cross-process watcher — this is single-process, same-worker restart, consistent with how `:worker`-scope daemons already behave per-process today.
