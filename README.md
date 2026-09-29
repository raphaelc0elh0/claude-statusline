# claude-statusline

A two-line status line for [Claude Code](https://claude.com/claude-code): context
window usage on the first line, plan rate limits on the second.

```
Opus 5 (1M context) │ 115k/1M (11%) │ myproject (main*) │ ⏱ 39m │ ◑ default
5h ●○○○○ 12% ⟳4:33pm │ wk ●●○○○ 34% ⟳aug 27 │ fable ●●●●● 100% ⟳aug 27
```

Line 1 — model, context used / window size (percentage), directory with git
branch (`*` if dirty), session duration, reasoning effort.
Line 2 — every rate limit on one line: the 5-hour window (`5h`), the weekly
limit (`wk`), one bar per model that has its own weekly limit on the plan (e.g.
`fable`), and extra-usage credits (`extra`) when the account has them enabled.
Each bar has 5 dots (20% each).

Percentages are color-coded: green below 50%, orange at 50%, yellow at 70%,
red at 90%.

## Install

Requires `bash` 4+, `jq`, and `curl`. `git` is optional (branch display).

```bash
curl -fsSL https://raw.githubusercontent.com/raphaelc0elh0/claude-statusline/main/statusline.sh \
  -o ~/.claude/statusline.sh
chmod +x ~/.claude/statusline.sh
```

Then point `~/.claude/settings.json` at it:

```json
{
  "statusLine": {
    "type": "command",
    "command": "bash \"$HOME/.claude/statusline.sh\""
  }
}
```

## How it works

Claude Code pipes a JSON blob to the status line command on stdin. This script
reads:

| Field | Used for |
| --- | --- |
| `.model.display_name` | model name |
| `.context_window.context_window_size` | window size (falls back to 200000) |
| `.context_window.current_usage.{input_tokens,cache_creation_input_tokens,cache_read_input_tokens}` | tokens used — the three are summed |
| `.cwd` | directory name and git branch lookup |
| `.session.start_time` | session duration |
| `.rate_limits.{five_hour,seven_day}` | the `5h` and `wk` bars |

Per-model weekly limits are not in the stdin payload: they come from the usage
API below (`.limits[]` entries with `kind == "weekly_scoped"`), labeled with
`.scope.model.display_name`.

With no stdin it prints `Claude` and exits 0, so it degrades quietly.

## The usage API call (read this before you run it)

The script queries usage over HTTP, at most once every 60s (cached). It needs
that call for the per-model bars, and falls back to it for the `5h`/`wk` bars
when `.rate_limits` is absent from stdin. **That path reads your Claude OAuth
access token** — from `$CLAUDE_CODE_OAUTH_TOKEN`, the macOS Keychain,
`secret-tool` on Linux, or `~/.claude/.credentials.json`, in that order — and
sends it as a bearer token to `https://api.anthropic.com/api/oauth/usage`.

Two things worth knowing:

1. That endpoint is **not a documented public API**. It is what Claude Code
   itself calls, and this script identifies as Claude Code to reach it. It can
   change or disappear without notice; when it does, the bars stop rendering
   and the rest of the line keeps working.
2. The token never leaves your machine except in that request to Anthropic. The
   response is cached for 60s in `${XDG_RUNTIME_DIR:-/tmp}/claude-$(id -u)/`,
   a per-user directory created with mode `700` — the cached usage data is not
   readable by other users on the machine.

If you would rather not have a status line script touching your credentials at
all, delete the body of `load_usage_data` (under `# ── Usage API (cached) ──`).
You lose the per-model bars and extra-usage credits; recent Claude Code versions
send `.rate_limits` on stdin, so the `5h` and `wk` bars keep working.

## License

MIT — see [LICENSE](LICENSE).
