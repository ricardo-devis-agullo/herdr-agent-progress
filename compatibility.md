# Compatibility and verification

This fork intentionally implements one automatic adapter: Devin CLI in Herdr on macOS.

## Supported client

| Client | Native mechanism | Configuration | Runtime evidence | Validation status |
| --- | --- | --- | --- | --- |
| Devin CLI | `SessionStart`, `PostCompaction`, `PostToolUse`, and `UserPromptSubmit` lifecycle hooks with `additionalContext` | `~/.config/devin/config.json`, or `--devin-config` | Official `herdr:devin` session ID, stable terminal ID, foreground Devin launch PID/start time, and verified caller ancestry | Implemented against Devin CLI 3000.10.27 hook documentation and the live Herdr Devin process/session shape; automated Rust tests pass. |

Devin hook payloads provide a stable `session_id`. Herdr's official Devin integration reports that same session under `agent_session.source = "herdr:devin"`. The progress hook accepts a binding only when those values match.

A live Herdr pane may contain nested foreground processes such as:

```text
devin
└── devin acp
    └── hook or tool command
```

The adapter binds to the unique outer Devin launch. Multiple independent Devin roots are rejected as ambiguous.

## Platform boundary

- macOS is listed in the plugin manifest and is the current target.
- Herdr 0.9.0 provides the required native session, process, terminal, and sequenced metadata APIs.
- Unix socket identity is checked using canonical path, device, inode, and change time.
- Windows remains unavailable because equivalent endpoint and process-launch identity checks are not implemented.
- Other coding agents are intentionally unsupported by this fork.

## Verification boundary

The automated suite covers lifecycle hook generation and removal, empty Devin-compatible matchers, post-compaction context reinjection, nested launch selection, caller ancestry, socket replacement, task-generation races, resume behavior, stale estimates, completed tasks, publisher retries, metadata clearing, and configuration preservation.

The implementation was checked against a live Herdr Devin pane exposing `agent = "devin"`, `agent_session.source = "herdr:devin"`, and a nested `devin`/`devin acp` process tree. On 2026-09-16, the local checkout completed its normal release build, linked into Herdr, configured the four Devin hooks and Devin-only sidebar override, accepted a live bound report, and published `100% · Done` through Herdr's native metadata tokens.
