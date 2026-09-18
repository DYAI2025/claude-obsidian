# claude-obsidian Hooks

Plugin hooks for the claude-obsidian wiki vault. All hooks are defined in `hooks.json`.

## Events

| Event | Type | Purpose |
|---|---|---|
| `SessionStart` | command | Loads `wiki/hot.md` into context by running `[ -f wiki/hot.md ] && cat wiki/hot.md \|\| true` (works for non-vault sessions without erroring). Matcher: `startup\|resume\|clear\|compact\|fork`. Hook-injected context does NOT survive compaction (only `CLAUDE.md` does); the `compact` source fires right after manual or automatic compaction, so the same hook restores the hot cache mid-session. |
| `PostToolUse` | command | After Write or Edit tool calls, runs `git add wiki/ .raw/` and commits. Guarded by `[ -d .git ]` so it never errors in non-git directories, and by `git diff --cached --quiet` so it never creates empty commits. Both `wiki/` and `.raw/` must exist: a missing pathspec makes `git add` fail and nothing is committed. The commit takes the whole index, so anything else that was already staged is committed too. |
| `Stop` | command | If tracked files under `wiki/` have uncommitted changes against `HEAD`, prints a `WIKI_CHANGED` reminder to update `wiki/hot.md`. Claude Code sends a Stop hook's plain stdout on exit 0 to the debug log, not to Claude, so Claude does not see this reminder. Untracked new pages do not trigger it, and changes the PostToolUse hook already committed do not either. |

No hook uses `type: "prompt"`. Claude Code does not support prompt-type hooks on `SessionStart` or `PostCompact` (on `SessionStart` it fails with "no conversation context is available"), and a prompt hook returns an ok/reason decision rather than loading files, so it cannot restore the hot cache. The former prompt hooks on `SessionStart` and `PostCompact` were removed for that reason.

## Known Issue: Plugin Hooks STDOUT Bug

`anthropics/claude-code#10875` documents that **plugin hook STDOUT may not be captured** by Claude Code, while identical inline hooks in `settings.json` work correctly.

**Impact**: If this bug is active in your Claude Code version, the SessionStart hook may not inject `wiki/hot.md` into context (at startup, on resume, or after compaction).

**Workaround**: The command-type SessionStart hook (`cat wiki/hot.md`) is the canonical safety check. It relies on STDOUT capture for context injection, so test against this issue if hot cache restoration fails. As a fallback, copy the hook config from `hooks.json` into your user-level `~/.claude/settings.json` instead of relying on plugin hooks.

**Test for the bug**: After installing the plugin, open a fresh Claude Code session in a directory containing a populated `wiki/hot.md`. Ask Claude "what's in the hot cache?". If Claude has no idea, the STDOUT bug is active in your version.

## Non-Vault Sessions

The SessionStart command hook uses `[ -f wiki/hot.md ] && cat wiki/hot.md || true` so it always exits 0, even when no vault is present. This makes the plugin safe to install globally without breaking non-vault Claude Code sessions.
