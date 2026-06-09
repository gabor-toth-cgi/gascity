---
title: Managed Hook Trust Source
description: Stable trust model for provider-managed hook files.
status: Proposed
updated: 2026-06-09
---

# Managed Hook Trust Source

## Problem

Provider hooks let Gas City prime sessions, drain nudges and mail, and create
handoffs before compaction. Codex also protects local hook execution with
per-hook trust state. The AgenticFun example currently works around that prompt
with `--dangerously-bypass-hook-trust` in provider args.

That bypass is too broad for the SDK. It disables a provider safety check for
the whole session, it is Codex-specific, and it makes a consumer config carry an
SDK infrastructure concern. Gas City needs a stable trust source for hooks it
generates without trusting user-authored hooks and without encoding any role
behavior in Go.

## Recommendation

Use provider-native trust state for generated hook commands, seeded by the hook
installer. For Codex, `internal/hooks` should continue to own generated
`.codex/hooks.json` files and should also upsert the matching `[hooks.state]`
entries into `$CODEX_HOME/config.toml` or `~/.codex/config.toml`.

The trusted identity is not "this path is always safe." It is the provider's
specific command-hook identity:

- absolute hook file path
- provider event label
- matcher where the provider includes it in trust identity
- hook group and handler indexes
- provider-normalized command handler hash

Only commands recognized as Gas City managed hook commands should be seeded.
User-authored commands in the same file remain user-owned and should not receive
trust entries from Gas City.

## Alternatives

### Stable Trusted Hook Path

A stable generated hook path is useful because it keeps trust state from moving
unnecessarily, but path-only trust is not the trust boundary. A path can point at
new bytes after an upgrade or manual edit. The durable boundary should be the
provider's per-command hash at that path.

### Pre-Seeded Trust for Generated Hooks

This is the recommended path. It is deterministic, idempotent, and scoped to the
exact generated commands Gas City owns. It also lets existing user choices such
as `enabled = false` remain intact while updating stale command hashes after a
managed hook upgrade.

### Explicit Managed-Hook Trust Policy

A new Gas City config policy is premature. The needed decision is already
expressible from existing facts: the configured provider wants hooks installed,
the hook file contains recognized managed commands, and the provider has a
native trust store. Add a config policy only after a second provider needs a
different operator-controlled trust mode.

## Contract

- Config decides which providers install hooks. Go code must not branch on role
  names or role behavior.
- Hook content stays in provider overlays under the managed hook installer.
- Trust seeding runs only for recognized managed commands and only when
  installing or upgrading a managed hook file on the real filesystem.
- Existing user-authored hooks are preserved, and custom hook commands are not
  trusted by Gas City.
- Existing provider trust blocks are updated in place, preserving user-managed
  fields such as `enabled = false`.
- Writes to provider config are atomic and preserve the existing file mode when
  the file already exists.
- Malformed, unreadable, or unknown hook files are preserved rather than
  overwritten.
- Example cities should not carry global provider args that bypass hook-trust
  prompts once managed trust seeding is active.

## Implementation Slices

1. Seed Codex hook trust entries from `internal/hooks` when installing managed
   Codex hooks. Cover canonical bytes, current hashes, disabled-state
   preservation, custom-hook skipping, and repeated-install stability.
2. Extend `gc doctor --fix` or city startup repair so existing generated Codex
   hook files under city, rig, agent, and worktree directories are upgraded and
   re-seeded.
3. Remove `--dangerously-bypass-hook-trust` from the AgenticFun example city
   after the managed trust path is covered by tests or a documented smoke check.
4. Generalize a provider-specific trust seeding interface only after another
   provider needs native hook trust state. Until then, keep this as Codex-family
   hook installer behavior, not a new config primitive.
