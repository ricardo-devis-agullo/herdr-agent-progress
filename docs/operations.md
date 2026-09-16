# Operations and development

For initial setup, follow [Getting started](getting-started.md). This build intentionally supports Devin CLI only.

## How it works

`herdr-progress` is one Rust executable with bundled instructions. Devin's `SessionStart` and `PostCompaction` hooks inject the complete bound context. `UserPromptSubmit` distinguishes new work from continuation, and `PostToolUse` provides throttled progress reminders. Devin calls the CLI after meaningful milestones and before substantive replies.

A detached publisher writes namespaced metadata to Herdr's native sidebar. It notices queued reports within approximately 250 milliseconds and scans for stale reports every 15 seconds. It does no work in Herdr's render loop.

Reporting uses Devin's normal tool permissions. Denied access to the local Herdr socket, process information, or plugin state does not stop the user's task. Configure does not change permission rules, disable the sandbox, or bypass hook trust.

## Install from a checkout

```bash
cargo build --release --locked
herdr plugin link "$PWD"
herdr integration install devin
herdr plugin action invoke configure --plugin agent-progress
```

Configure edits Devin's user configuration at `~/.config/devin/config.json` by default. Use an alternate file when needed:

```bash
target/release/herdr-progress configure --devin-config /absolute/path/to/config.json
```

`XDG_CONFIG_HOME`, `XDG_STATE_HOME`, `HERDR_CONFIG_PATH`, `HERDR_PLUGIN_CONFIG_DIR`, and `HERDR_PLUGIN_STATE_DIR` are respected. Configure writes a stable launcher and copied executable in the plugin config directory.

The installer preserves unrelated JSON comments, hooks, and settings. It validates the complete Herdr sidebar candidate before changing live configuration and refuses symlink paths, malformed files, edited conflicting rows, full layouts, and concurrent edits.

## Commands

```text
herdr-progress --instructions                 # --skill is an alias; prints only
herdr-progress context --binding B
herdr-progress status --binding B
herdr-progress begin --binding B --expected-task none --title 'Task title'
herdr-progress report --binding B --task ID --percent 65 --activity 'Testing changes'
herdr-progress report --binding B --task ID --unknown --activity 'Assessing task'
herdr-progress clear --binding B --task ID
herdr-progress activate --pane PANE_ID
herdr-progress doctor
herdr-progress start
herdr-progress stop
herdr-progress unconfigure
```

Use the absolute launcher path printed by the hook. `begin` returns an opaque task ID. Continuations and clarifications keep that ID. New work passes the exact current ID to `--expected-task`; there is no unconditional replacement. A completed task requires a new generation before further reports.

`activate` is a human-only fallback that checks a live Devin process and native session before printing bound instructions. It does not type into the pane. Printing `--instructions` alone cannot create a binding.

Administrative commands accept `--endpoint SOCKET`. Reporting commands require the invoking Devin process's inherited Herdr environment and reject endpoint overrides.

## Devin identity

Every bound operation verifies:

```text
Herdr Unix socket identity
stable terminal ID
official herdr:devin session ID
foreground Devin launch PID and start time
caller ancestry from that launch
```

A Herdr Devin pane can expose a parent `devin` process and a nested `devin acp` process. The adapter selects the unique outer Devin launch from that foreground process tree. Independent candidate roots are treated as ambiguous and rejected.

Compaction within a launch reuses the binding. A verified resume in a new process rotates the binding and restores only the matching native session's task. Old bindings cannot read, replace, report, or clear successor state. Pane moves resolve by terminal ID.

The binding is an opaque consistency token, not a security boundary against other processes under the same OS account.

## State and publication

SQLite transactions serialize task generations and publication decisions. Sequence numbers are persisted before sends, so a timeout or crash cannot reuse a sequence. State lives in the plugin state directory, including `progress.sqlite3` and `publisher.log`.

The publisher uses metadata source `agent-progress` and manages four tokens:

```text
agent_progress_percent
agent_progress_freshness
agent_progress_activity
agent_progress_summary
```

The default dim row displays `$agent_progress_summary`. Percentage and freshness precede activity so sidebar truncation preserves the most important status. Summary text is capped at 80 display columns and activity at 40.

Per-endpoint file locks prevent competing publishers. A crash releases the lock; Start, a valid hook, or a reporting command restarts the publisher. Disabling the registration makes hooks inert and causes the publisher to clear its tokens before exiting when the endpoint remains reachable.

## Remove

```bash
herdr plugin action invoke unconfigure --plugin agent-progress
herdr plugin unlink agent-progress
```

Unconfigure removes matching owned hooks and rows while preserving later user edits. It retains local state and the stable launcher for diagnostics.

## Verification

```bash
cargo test --locked
cargo clippy --all-targets --locked -- -D warnings
cargo fmt --check
```

Tests cover Devin lifecycle-hook configuration, nested Devin launch selection, caller ancestry, task replacement races, tombstones after clear, stale/completed behavior, launch rotation, endpoint isolation, publisher sequence reservation, configuration preservation, row limits, and reporter recursion filtering.
