# Changelog

## Unreleased

### Fixes

- Route `node scripts/run-vitest.mjs` output through the Vitest reducer so Rolldown plugin-timing warnings do not drown out passing test summaries.
- Keep Tokenjuice's Codex hook compatible with current Codex hook and approval surfaces, including `hooks`, `PermissionRequest`, Windows commands, async hooks, and approval/sandbox doctor reporting.
- Compact whole JSON fallback output without dropping non-zero exit status.
- Add timeout safety caps to Tokenjuice-installed Codex, Claude Code, and Copilot CLI hooks, with doctor warnings for stale entries.
