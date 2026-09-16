# Getting started with Devin Agent Progress

Install the local Herdr plugin, connect Devin CLI, and verify task estimates in the expanded agent sidebar.

## 1. Check the prerequisites

- macOS and Herdr 0.9.0 or newer
- Devin CLI with lifecycle-hook support
- Rust/Cargo and Apple's command-line build tools
- A running Herdr session

Run the setup commands from inside Herdr. Agent Progress uses the local Herdr socket, process information, and its local state directory. It does not need Python, Node, a hosted service, or another API key.

## 2. Build and link the checkout

```bash
cd "$HOME/Work/herdr-agent-progress"
cargo build --release --locked
herdr plugin link "$PWD"
```

Review Herdr's plugin preview. The first startup waits for explicit configuration before changing Devin or Herdr files.

## 3. Connect Devin CLI

Install Herdr's official Devin integration:

```bash
herdr integration install devin
```

Then configure progress reporting:

```bash
herdr plugin action invoke configure --plugin agent-progress
```

Configure adds Devin lifecycle hooks, adds the sidebar row, reloads Herdr's configuration, and starts the publisher. Existing hooks, comments, permissions, and unrelated sidebar settings are preserved.

Restart or resume Devin CLI so it loads the new hooks. Review normal hook trust and sandbox permission prompts; setup does not bypass them.

## 4. Confirm progress appears

Give Devin a task with several steps and expand Herdr's agent sidebar. Reports should look like:

```text
~15% · Reading code
~65% · Testing changes
100% · Done
```

An estimate can decrease when Devin discovers more work. An unfinished report becomes `stale` after five minutes without an update. A clarification or continuation keeps the current task generation; genuinely new work starts another generation. Post-compaction context and verified native resumes restore the current task.

## Check the setup

```bash
herdr plugin list
herdr integration status
herdr plugin action invoke doctor --plugin agent-progress
herdr plugin log --plugin agent-progress
```

`configured: true` verifies the plugin registration and marker. It does not prove that the current Devin process loaded the hooks, so confirm with a real task in the sidebar. In Devin CLI, `/hooks` lists loaded hooks and their source files.

## Alternate configuration files

The default files are:

```text
~/.config/devin/config.json
~/.config/herdr/config.toml
```

To use alternate paths, invoke the installed executable directly:

```bash
PROGRESS="$(herdr plugin config-dir agent-progress)/herdr-progress"
"$PROGRESS" configure \
  --devin-config /absolute/path/to/devin/config.json \
  --herdr-config /absolute/path/to/herdr/config.toml
```

`XDG_CONFIG_HOME`, `XDG_STATE_HOME`, `HERDR_CONFIG_PATH`, `HERDR_PLUGIN_CONFIG_DIR`, and `HERDR_PLUGIN_STATE_DIR` are respected. Herdr plugin actions supply the plugin directories.

Configure copies the executable into the plugin config directory and writes a stable launcher, so hooks do not depend on the checkout path or shell `PATH`.

## Upgrade

Stop the publisher in each running Herdr session before replacing the build:

```bash
herdr plugin action invoke stop --plugin agent-progress
cd "$HOME/Work/herdr-agent-progress"
cargo build --release --locked
herdr plugin link "$PWD"
herdr plugin action invoke configure --plugin agent-progress
```

Restart or resume Devin afterward. Stop clears the display and revokes launch bindings; the next verified `SessionStart` restores the matching native session's task with a fresh binding.

## Remove

```bash
herdr plugin action invoke unconfigure --plugin agent-progress
herdr plugin unlink agent-progress
```

Unconfigure removes matching owned hooks and rows, preserves user edits, clears published metadata, and revokes active bindings. It retains local state and the stable launcher for diagnostics.

## Troubleshooting

- **Build fails with an Xcode license message:** Run `sudo xcodebuild -license` in your terminal and accept Apple's license, then rebuild.
- **No progress row:** Confirm the plugin is enabled, install the Devin integration, run Configure, restart or resume Devin CLI, and check `/hooks`.
- **Doctor succeeds but no estimates appear:** Inspect the plugin log and Devin's hook or permission messages. Doctor does not prove a live hook reported a task.
- **A binding is rejected:** Resume or restart Devin CLI. Do not copy a binding from another pane or override the endpoint on a reporting command.
- **Setup refuses a file:** Inspect the reported path. Configure refuses symlinks, malformed files, conflicting progress rows, full 16-row layouts, and concurrent edits.
- **The estimate is stale or decreases:** Both are expected. Stale means no report for five minutes; a decrease means Devin found more work.
