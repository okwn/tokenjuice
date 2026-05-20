# Host Integration Playbook

This document is the implementation checklist for adding a new host integration (like codex, claude-code, codebuddy, cursor, pi).

## when to use this

Use this before opening a PR that adds `tokenjuice install <host>` or any new hook/adapter path.

## design decisions first

Define the host hook model before writing code:

- Can the host rewrite shell input before execution?
- Can the host replace shell output after execution?
- Are hooks file-based, extension-based, or API-based?
- Does the host return plain text, structured JSON, or both?

Pick one integration mode:

- **post-tool compaction** (preferred when shell output can be replaced safely) — hosts: codex, pi, opencode, openclaw, openhands, copilot-cli, aider, avante, cline, continue, droid, gemini-cli, junie, zed
- **pre-tool command wrapping** (use `tokenjuice wrap` when post replacement is unavailable) — hosts: claude-code, codebuddy, cursor, vscode-copilot

For pre-tool wrapping, preserve shell semantics (for example `bash -lc '<cmd>'`) and ensure classification normalization can recover the nested command.

## implementation checklist

For a new host adapter in `src/hosts/<host>/index.ts`:

- **install flow**
  - Write/update host config atomically
  - Preserve unrelated keys
  - Keep installation idempotent (replace prior tokenjuice entry, keep non-tokenjuice entries)
- **doctor flow**
  - Detect `disabled` / `warn` / `broken` / `ok` states
  - Validate expected command against installed command
  - Report missing executable paths
  - Return a single repair command
- **runtime hook flow**
  - Parse hook payload defensively
  - Skip non-target tools/events early
  - Preserve explicit raw bypass behavior
  - Never throw hard on hook input parse failures

Then wire CLI + exports:

- `src/cli/main.ts`
  - `install <host>`
  - `doctor <host>`
  - Usage text
  - Runtime hook entry command if needed
- `src/index.ts`
  - Runtime/install/doctor exports
  - Result/report type exports
- `src/hosts/shared/hook-doctor.ts`
  - Add the host to aggregate doctor report

## host environment variables

Each host integration respects a set of environment variables for config directory resolution and shell override. The precedence for config dir resolution is host-specific; the examples below show the primary override first and fallbacks in order.

| Host | Config Override | Shell Override | Notes |
| --- | --- | --- | --- |
| **codex** | `CODEX_HOME` → `~/.codex` | (N/A — post-tool) | |
| **pi** | `PI_CODING_AGENT_DIR` → `~/.pi/agent` | (N/A — post-tool) | |
| **opencode** | `OPENCODE_CONFIG_DIR` | (N/A — post-tool) | |
| **openclaw** | (inferred from hook install path) | (N/A — post-tool) | |
| **openhands** | `OPENHANDS_CONFIG_DIR` | (N/A — post-tool) | |
| **claude-code** | `CLAUDE_CONFIG_DIR` → `CLAUDE_HOME` → `~/.claude` | `TOKENJUICE_CLAUDE_CODE_SHELL`, then `SHELL` | Pre-tool wrap |
| **codebuddy** | `CODEBUDDY_CONFIG_DIR` → `~/.codebuddy` | `TOKENJUICE_CODEBUDDY_SHELL`, then `SHELL` | Pre-tool wrap |
| **cursor** | `CURSOR_HOME` | `TOKENJUICE_CURSOR_SHELL`, then `SHELL` | Pre-tool wrap |
| **vscode-copilot** | `COPILOT_HOME` → `~/.copilot` | (N/A — pre-tool wrap via chat.useHooks) | Uses `~/.copilot/hooks/` |
| **copilot-cli** | `COPILOT_HOME` | (N/A — post-tool) | Uses `~/.copilot/hooks/` |
| **aider** | `AIDER_CONFIG_DIR` | (N/A — post-tool) | |
| **avante** | `AVANTE_CONFIG_DIR` | (N/A — post-tool) | |
| **cline** | `CLINE_CONFIG_DIR` | (N/A — post-tool) | |
| **continue** | `CONTINUE_CONFIG_DIR` | (N/A — post-tool) | |
| **droid** | `DROID_CONFIG_DIR` | (N/A — post-tool) | |
| **gemini-cli** | `GEMINI_CLI_CONFIG_DIR` | (N/A — post-tool) | |
| **junie** | `JUNIE_CONFIG_DIR` | (N/A — post-tool) | |
| **zed** | `ZED_CONFIG_DIR` | (N/A — post-tool) | |

## doctor output guide

The `doctor <host>` command reports one of four states:

### `ok`

The hook is installed and pointing at a valid, matching command. No action required.

```
tokenjuice doctor codex
✓ codex hook installed
  expected: tokenjuice codex-hook
  detected: /usr/local/bin/tokenjuice codex-hook
  status: ok
```

### `disabled`

No hook entry found for this host. Run `tokenjuice install <host>`.

```
tokenjuice doctor codex
✗ no codex hook found
  fix: tokenjuice install codex
  status: disabled
```

### `broken`

A hook entry exists but does not match the expected command, or one or more executable paths in the command are missing.

```
tokenjuice doctor codex
✗ codex hook command mismatch or missing paths
  expected: tokenjuice codex-hook
  detected: /stale/bin/tokenjuice codex-hook
  missing paths: /stale/bin/tokenjuice
  fix: tokenjuice install codex
  status: broken
```

### `warn`

A hook entry exists and points at a valid command, but an advisory note applies (for example, a feature flag is off in the host config).

```
tokenjuice doctor codex
⚠ codex hook installed but feature flag is off
  expected: tokenjuice codex-hook
  detected: /usr/local/bin/tokenjuice codex-hook
  advisory: set hooks = true in ~/.codex/config.toml, or run: codex exec --enable codex_hooks
  status: warn
```

## failure modes per host

### Post-tool hosts (codex, pi, opencode, openclaw, openhands, copilot-cli, aider, avante, cline, continue, droid, gemini-cli, junie, zed)

| Failure | Symptom | Resolution |
| --- | --- | --- |
| No hook installed | `doctor` → `disabled`; compaction never fires | `tokenjuice install <host>` |
| Hook installed but host feature flag off | `doctor` → `warn`; hook fires but output is not compacted | Enable feature in host config; see doctor advisory |
| Hook points to stale/missing binary | `doctor` → `broken`; commands pass through uncompressed | `tokenjuice install <host>` to refresh path |
| Hook command mismatch (path changed) | `doctor` → `broken`; installed command differs from expected | `tokenjuice install <host>` to update |
| Host deleted/uninstalled | `doctor` → `disabled`; hook file still present but host gone | `tokenjuice uninstall <host>` |
| Multiple tokenjuice hook entries (duplicate install) | `doctor` → `ok`; double-compaction wastes tokens | `tokenjuice install <host>` (idempotent, deduplicates) |

### Pre-tool wrap hosts (claude-code, codebuddy, cursor, vscode-copilot)

| Failure | Symptom | Resolution |
| --- | --- | --- |
| No hook installed | `doctor` → `disabled`; commands run unaltered | `tokenjuice install <host>` |
| Hook installed but host feature flag off | `doctor` → `warn`; pre-tool hook never fires | Enable feature in host settings |
| Hook points to stale/missing binary | `doctor` → `broken`; commands pass through uncompressed | `tokenjuice install <host>` |
| Shell not found (`sh` absent) | Wrap returns command unchanged; no compaction | Install `sh`, or set `TOKENJUICE_<HOST>_SHELL` to a valid shell |
| Already-wrapped command double-wrapped | trace shows nested `tokenjuice wrap`; ratio unchanged | Pre-tool hook detects prior wrap and skips (expected behavior) |
| Platform mismatch (Windows host, WSL required) | `doctor` → `broken`; hook denies on non-WSW Win | Use WSL for cursor/codebuddy pre-tool hooks |

## test strategy (required)

Add host-specific tests and aggregate tests:

- `test/hosts/<host>.test.ts`
  - Install idempotency
  - Preserve unrelated config fields
  - Doctor status matrix (`disabled`, `warn`, `broken`, `ok`)
  - Runtime behavior (rewrite/skip/bypass paths)
- Aggregate coverage
  - Update tests for `doctorInstalledHooks` if the new host is included there

## critical test isolation rules

New adapters often fail CI/local due to leaked machine config. Isolate host homes explicitly:

- Set and reset host env vars in each suite (`CODEX_HOME`, `CLAUDE_CONFIG_DIR`, `CLAUDE_HOME`, `CURSOR_HOME`, `PI_CODING_AGENT_DIR`, `OPENCODE_CONFIG_DIR`, `XDG_CONFIG_HOME`, `COPILOT_HOME`, `TOKENJUICE_CLAUDE_CODE_SHELL`, `TOKENJUICE_CURSOR_SHELL`, `SHELL`, etc.)
- `COPILOT_HOME` affects **copilot-cli only**; VS Code Copilot Chat ignores it and always resolves under `$HOME/.copilot/hooks/`. If your adapter should not depend on `COPILOT_HOME`, add an explicit test that sets it to a decoy path and asserts the install does not land there.
- Avoid reading real `~/.<host>` in tests
- Use temp dirs for all config paths
- Restore `PATH` after each test

### shared hook dirs (hazard)

copilot-cli and vscode-copilot both read every `*.json` under `~/.copilot/hooks/`. Install each host under a per-host filename (`tokenjuice-cli.json` and `tokenjuice-vscode.json`) so neither host's install overwrites the other. Doctor for both hosts scans sibling files and reports stray tokenjuice entries.

If you add a new host to aggregate doctor logic, existing aggregate tests may start failing unless they set that host's env home to temp storage.

## regression gates before merge

Minimum gate for a new host adapter:

```bash
pnpm typecheck
pnpm vitest run test/hosts/<host>.test.ts
pnpm vitest run test/hosts/codex.test.ts test/hosts/claude-code.test.ts test/hosts/codebuddy.test.ts test/hosts/cursor.test.ts test/hosts/pi.test.ts
```

If you changed normalization/classification paths, also run:

```bash
pnpm vitest run test/core/command.test.ts test/core/classify.test.ts test/core/trace.test.ts
```

## manual verification flow

Run from repo root:

```bash
pnpm build
node dist/cli/main.js install <host>
node dist/cli/main.js doctor <host>
```

For command-path diagnostics, use trace:

```bash
node dist/cli/main.js wrap --format json --trace -- bash -lc "git status --short"
```

Verify:

- Normalized command/argv match expected command intent
- Matched reducer is specific (not generic fallback for common commands)
- Raw mode preserves full output:

```bash
node dist/cli/main.js wrap --format json --trace --raw -- <command>
```

For truncation-related debugging, verify both boundaries explicitly:

- Reducer truncation: if output includes `... omitted ...` / `... lines omitted ...`, rerun with `--raw` first.
- Capture truncation: if output includes `[tokenjuice: output truncated]`, rerun with a larger capture ceiling (for example `--max-capture-bytes 52428800`).
- Do not treat these as the same failure mode; reducer bypass and capture-size tuning solve different problems.

## docs updates required in same PR

When adding a host integration, update:

- `README.md` command examples and support table
- `docs/spec.md` supported host hooks table
- Dedicated design doc if host behavior differs materially (like cursor pre-tool wrapping)