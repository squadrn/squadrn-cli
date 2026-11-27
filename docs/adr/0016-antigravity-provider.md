# 0016 — Antigravity CLI provider

Status: proposed

## Context

Since 2026-06-18 Google refuses Gemini CLI for personal Google accounts, Google AI Pro and Ultra
([ADR 0013](0013-gemini-provider.md) §10). Those users moved to **Antigravity CLI** (`agy`), a
closed-source Go binary, distributed by Google's own installer
(`curl -fsSL https://antigravity.google/cli/install.sh | bash`, into `~/.local/bin`), which
updates itself in the background. This ADR adds it as squadrn's fourth provider.

It differs from every provider so far in one way that shapes the design: **it has no variable
to relocate its config dir** (google-antigravity/antigravity-cli#155, open since 2026-05-24).
Everything hangs off the home directory, so an account can only be isolated by running `agy`
with another `HOME`, which every command the agent runs inherits.

Facts checked on 2026-11-21 against agy 1.5.14 and on 2026-11-27 against agy 1.5.18 (P0; macOS
x86_64). **P** = observed in the probe below, **D** = the docs agy ships in its builtin
`agy-customizations` skill, **S** = read from the binary's strings, **W** = Google's web docs or
the issue tracker, not verified locally. A fact marked W is not relied on before P0 verifies it.

| Fact | |
|---|---|
| `agy --version` prints the bare version (`1.5.14`); no `login`, `logout` or auth-status subcommand. Subcommands: `acp`, `agent(s)`, `changelog`, `help`, `install`, `mcp`, `mic-serve`, `models`, `plugin(s)` | P |
| Flags: `-p/--print`, `--output-format text\|json\|stream-json`, `--print-timeout`, `-c/--continue`, `--conversation <id>`, `--mode accept-edits\|auto\|plan`, `--model`, `--effort`, `--sandbox`, `--dangerously-skip-permissions`, `--add-dir`, `--settings <json or file>` ("for this session only, never saved"), `--log-file`, `--project` | P |
| All state is under `$HOME/.gemini/`: `antigravity-cli/` (`settings.json`, `installation_id`, `conversations/<id>.db` SQLite, `conversation_summaries.db`, `brain/<id>/` transcripts, `cache/`, `log/`, `presence/<id>.lock`) and `config/` (`config.json`, `mcp_config.json`, `projects/`, `hooks.json`, `plugins/`). Nothing is written outside `HOME` (checked with `env -i`) | P |
| Paths come from `os.UserHomeDir()` (`$HOME` on Unix); no `GEMINI_CLI_HOME` or `AGY_*_HOME` | S, P |
| Login: Google OAuth with PKCE, redirect `https://antigravity.google/oauth-callback`, the code is pasted into the terminal (no localhost listener: works over SSH and in tmux). `-p` without a login starts that flow and waits 60 s, then exits 1 with a JSON `"status":"ERROR"` | P |
| Credentials go through `zalando/go-keyring` (macOS: the `security` tool by absolute path, so a stub first on `PATH` does not intercept it), with a file fallback when the keychain write fails | S |
| With `HOME` pointing elsewhere, macOS has no default keychain (`security default-keychain`: "A default keychain could not be found"), and after a real login the token was in **`$HOME/.gemini/antigravity-cli/antigravity-oauth-token`** (589 bytes); the user's login keychain had no Antigravity item before or after | P |
| With the real `HOME` (squadrn's main account) the login presumably goes to the login keychain; Linux (Secret Service over D-Bus) and Windows (Credential Manager) do not depend on `HOME` | S, W (not verified) |
| Hooks: `$HOME/.gemini/config/hooks.json` (or a plugin's `hooks.json`; hooks in `settings.json` are not loaded) is an object of **named** hooks, each `{enabled?, SessionStart, PreToolUse, PostToolUse, PreInvocation, PostInvocation, Stop}`; tool events take `[{matcher (regex on the tool name), hooks: [{type: "command", command, timeout (s, default 30)}]}]` | D, P |
| A hook runs through `sh -c` (Windows `cmd /c`) in the dir of its `hooks.json`; `~` is expanded; payloads are camelCase JSON on stdin; hooks are synchronous | D |
| `PreToolUse` stdin: `toolCall: {id, name: "run_command", args: {CommandLine, WaitMsBeforeAsync, toolSummary}}`, `cwd`, `workspacePaths`, `conversationId`, `transcriptPath`, `artifactDirectoryPath`, `modelName`, `lastUserInput`, `stepIdx`, `hookEventName` | P |
| `PreToolUse` stdout `{"decision": "allow"\|"deny"\|"ask"\|"force_ask", "reason", "permissionOverrides", "overwrite"}`; `ask` respects the user's cached "always allow", `force_ask` does not | D |
| A `deny` blocked `git push --force origin main` under `--dangerously-skip-permissions`; the agent received "tool call denied by pre-tool hook: <reason>" | P |
| `Stop` stdin: `terminationReason` (`NO_TOOL_CALL`, `ERROR`, `USER_CANCELED`, `MAX_INVOCATIONS`, …), `error`, `fullyIdle`, `continuationActive`; `{"decision":"continue"}` keeps the agent going, anything else lets it stop. `PreInvocation` stdin: `invocationNum`, `initialNumSteps` | D |
| A `PreToolUse` hook that exits non-zero (1 or 2), times out or prints invalid JSON **blocks** the tool call ("Encountered error in tool execution: JSON hook … failed"); exit 0 with empty stdout lets it through | P |
| A `hooks.json` agy cannot parse (e.g. a flat event given the grouped shape) is **ignored as a whole**, silently: every hook in it is off and the only trace is `Failed to parse hooks file …` in `antigravity-cli/log/cli-*.log`. `SessionStart`, `PreInvocation`, `PostInvocation` and `Stop` take a flat list of handlers; `PreToolUse`/`PostToolUse` take `{matcher, hooks}` groups | P, D |
| Tools that run commands: `run_command` (`CommandLine`, `WaitMsBeforeAsync`) and **`manage_task`** with `Action: "send_input"` (`Input`, `TaskId`), which writes to a running command's stdin. Others seen: `read_file`, `write_file`, `update_steps` | P |
| `run_command` gets `GIT_EDITOR=false` and `GIT_PAGER=cat` and nothing else from agy; `core.hooksPath` is untouched and a repository's `pre-push` hook runs | P |
| `/usage` (alias `/quota`) costs no model request (0 tokens, quota unchanged across calls, ~3 s). `--output-format json` gives `command.data.groups[]` ("Gemini Models", "Claude and GPT models"), each with `buckets[]` of `{id, window: "weekly"\|"5h", remaining_fraction, reset_time}` | P |
| Without a login, `-p` prints `Authentication required` on stderr at once, then waits 60 s even with stdin closed and exits 1 (`"error":"authentication failed or timed out"`) | P |
| `/logout` is refused in print mode ("clears stored credentials, an effect that outlives the run"); `--disable-slash-commands` turns every slash command, `/usage` included, into prompt text | P |
| Permission rules `"deny": ["command(mkdir)"]` match by command prefix and are enforced in `-p` mode; deny beats ask beats allow (W) | P |
| `--settings` **replaces** the `permissions` of `settings.json` instead of merging: a deny in `settings.json` applies alone, not when `--settings` carries its own `permissions` | P |
| A conversation copied to another home (`conversations/<id>.db` and `brain/<id>/`, plus that home's own login) resumes there with `--conversation <id>` and keeps its context | P |
| agy does not set the terminal title (no OSC 0/2; it only queries OSC 11 and 7501) | P |
| 1.5.18 added `update` and `remote-control` subcommands, `--remote-control`, `--remote-agent`, `--input-format stream-json` and `--disable-slash-commands`; still no auth subcommand | P |
| `AGY_CLI_DISABLE_AUTO_UPDATE=true` stops the background self-update | W, used in the probe |
| `agy plugin import gemini\|claude` converts other CLIs' plugins; `agy plugin install\|uninstall\|list\|enable\|disable\|validate` | P |

## Decision

### 1. Spec and account layout

`provider.Antigravity`: ID **`agy`** (account refs `agy/work`; the binary's name, short to type),
name "Antigravity CLI", binary `agy`, install hint the official installer, `DirPrefix` **`.agy-`**
(never `.gemini-`, which Gemini CLI accounts own).

An account is a home directory:

- **main** is the user's real home, run with nothing changed. It exists when
  `~/.gemini/antigravity-cli` does. Its state dir is `~/.gemini`, shared on disk with Gemini CLI's
  main account (`gemini/main` is `~/.gemini` too). The two CLIs use different files; squadrn never
  removes or relinks `~/.gemini` for either.
- **other accounts** are `~/.agy-<name>`, run with `HOME=~/.agy-<name>`; their state is
  `~/.agy-<name>/.gemini/`. The marker and squadrn's record stay in `~/.agy-<name>`.

`Spec` gains `HomeEnv`: the provider isolates accounts by replacing `HOME`, so `ConfigEnv` is
`HOME`, and the main account's state dir is a separate field (`MainState: ".gemini"`) from the
dir that proves it exists (`MainDir: ".gemini/antigravity-cli"`). `Account.StateDir()` uses it.
Provider files are then relative to the state dir: `antigravity-cli/settings.json`,
`config/hooks.json`, `config/mcp_config.json`, `antigravity-cli/antigravity-oauth-token`.

### 2. Restoring the user's home for the agent's commands

Replacing `HOME` hides the user's own configuration from every command the agent runs. For a
secondary account squadrn adds, only when the variable is unset and the target exists:

| Variable | Value | Restores |
|---|---|---|
| `GIT_CONFIG_GLOBAL` | `<home>/.gitconfig` (or the XDG one) | identity, aliases, signing, credential helpers |
| `GH_CONFIG_DIR` | `<home>/.config/gh` | `gh` login |
| `XDG_CONFIG_HOME` | `<home>/.config` | tools that follow XDG |
| `NPM_CONFIG_USERCONFIG` | `<home>/.npmrc` | registry auth |
| `CARGO_HOME`, `RUSTUP_HOME`, `GOPATH` | `<home>/.cargo`, `<home>/.rustup`, `<home>/go` | toolchains |

`ssh` needs nothing: OpenSSH takes the home from the password database, not `$HOME`. Squadrn's
layer-2 entries (`GIT_CONFIG_*`) are unaffected. Anything else that reads `~` (shell rc files,
`~/.netrc`, cloud CLIs) sees the account's home; `doctor` and `account add` say so once, and the
main account has none of this. Linking dotfiles into the account home was rejected: partial,
fragile, and it puts secrets in a second place.

When agy gains a config-dir variable (#155), squadrn switches to it and drops this section.

### 3. Credentials

- `Spec.Credentials`: `antigravity-cli/antigravity-oauth-token`; every profile of every provider
  denies the other accounts' copies.
- A secondary account counts as isolated only when its login is in that file. After a login,
  `account add`/`login` check that the file exists; if agy reports a login but the file is absent,
  the login went to a shared OS keyring, and squadrn says so and marks the account
  (`CredentialStore`). On Linux and Windows this is expected until P0 shows otherwise: `doctor`
  warns that secondary accounts may share one login there.
- The main account's login stays wherever agy keeps it (the keychain on macOS).

### 4. Auth

No auth command, so:

- **Status**: secondary accounts: logged in when the token file exists (never read). Main: the
  keychain cannot be read without prompting, so squadrn runs `agy -p /usage --output-format json`
  with stdin closed and a 15 s deadline, reading stderr as it comes: `Authentication required`
  means logged out and the process is killed at once (it would otherwise wait 60 s); a `SUCCESS`
  result means logged in; anything else is "unknown". `/usage` costs no model request.
- **Login**: `agy -p /usage --output-format text`, which runs the OAuth flow, prints the quota
  and exits by itself.
- **Logout**: secondary accounts: the token file goes to the trash (`FileLogout`). Main: agy
  refuses `/logout` in print mode, so squadrn prints "run agy, then /logout".
- Squadrn never passes `--disable-slash-commands` to these calls.

### 5. Push guard

**Layer 1** is a named hook `squadrn` in each account's `config/hooks.json` (an object of named
hooks, so squadrn owns one key and never edits the user's others):

```json
{"squadrn": {"PreToolUse": [{"matcher": "^(run_command|manage_task)$", "hooks": [{"type": "command",
  "command": "\"<data dir>/plugin/bin/squadrn-hook\" guard-git --agent agy", "timeout": 5}]}]}}
```

- `guard-git --agent agy` reads `cwd` and the command: `toolCall.args.CommandLine` for
  `run_command`, `toolCall.args.Input` for `manage_task` with `Action: "send_input"` (other
  actions are allowed). Input sent to a running command is checked line by line as a command,
  since the agent can start `sh` and type a push into it. It replies with agy's JSON. **Ask maps
  to `force_ask`**, so a cached "always allow" never waives a push the guard wants confirmed;
  deny stays deny. agy is the first provider since Claude Code where the guard can ask instead
  of deny.
- Fail-closed holds on agy's side: a hook that exits non-zero, times out or prints invalid JSON
  blocks the call. Only exit 0 with empty stdout passes, so the shim always prints a decision,
  and on any error it prints a deny and exits 2.
- A `hooks.json` agy cannot parse turns **every** hook in it off, silently. Squadrn writes the
  file atomically, checks its own entry against the documented shape (flat lists for
  `PreInvocation`/`Stop`, `{matcher, hooks}` groups for tool events) before writing, and
  refuses to write over a file it cannot parse. `doctor` fails when the file does not parse or
  when agy's newest log contains `Failed to parse hooks file`.
- Not a plugin: `agy plugin install` adds a marketplace step for one hook. `hooks.json` is
  deterministic and per account. Revisit if squadrn ships agy skills.
- Windows: the command runs under `cmd /c`, as for Codex (ADR 0011).

**Layer 2** is the universal git pre-push hook. Unlike Gemini CLI, agy leaves `core.hooksPath`
alone (it only sets `GIT_EDITOR` and `GIT_PAGER`), so all three layers apply.

The standard profile also adds `"ask": ["command(git push)"]` to the settings: a second,
independent prompt before every push.

### 6. Notifications

Hooks in the same named `squadrn` entry:

- `PreInvocation` with `invocationNum` 0 → working;
- `Stop` with `NO_TOOL_CALL` → done (input when the response ends in a question), `ERROR` →
  failed (a usage limit when the error says so), `USER_CANCELED` → nothing;
- the guard's own `ask`/`force_ask` → needs input.

agy has no event for its own permission prompts and does not set the terminal title, so there
is no pane-title fallback (`PaneTitleStates` stays empty): a session waiting on agy's own prompt
shows as working. Hooks are synchronous: the notify hook prints `{}`, never
`"decision":"continue"` (which would keep the agent running), exits 0 within 2 s and delivers in
the background.

### 7. Profiles

`--settings` replaces the `permissions` of `settings.json` instead of merging (P0), so a
profile passed that way would silently drop the user's own rules. Profiles are therefore merged
into the account's own `settings.json` (plain JSON) as for the other providers (ADR 0011), and
squadrn never passes `--settings`. Rules use agy's syntax, `command(<prefix>)`, matched by
prefix.

| Profile | Keys |
|---|---|
| minimal | credential isolation (§3); deny reads of `.env`, `.env.*`, `~/.ssh/**` and other accounts' token files |
| standard | minimal, the guard, `ask` for `git push`, background self-update off in secondary accounts (`AGY_CLI_DISABLE_AUTO_UPDATE=true`: parallel panes would update one binary at once) |
| strict | standard, `--sandbox`, no `--dangerously-skip-permissions` passthrough |

### 8. Usage, swap, MCP, shared items

- **Usage**: `RateLimiter` from `agy -p /usage --output-format json`, at most every 5 minutes
  unless `--refresh`, as for Codex. Quota comes in model groups ("Gemini Models", "Claude and
  GPT models"), each with a 5-hour and a weekly bucket; `usage.Snapshot` takes, per window, the
  group with the least `remaining_fraction` (used = 1 − remaining) and its `reset_time`, and
  `usage` shows the groups separately.
- **Swap**: a conversation is `conversations/<id>.db` plus `brain/<id>/`; copied into another
  account it resumes there with `--conversation <id>` (P0; the summary index was not needed).
  `Caps.SessionTransfer` is on.
- **MCP**: `config/mcp_config.json` (`mcpServers`, `serverUrl` for remote servers); `shared_mcp`
  copies the allowlisted servers with `agy mcp add`.
- **Shared from main**: `GEMINI.md` (agy reads `~/.gemini/GEMINI.md` and `AGENTS.md`), global
  skills and agents once their paths are confirmed. Never shared: the token file, `installation_id`,
  conversations, `brain/`, caches, `hooks.json`, `config.json`.

### 9. Provider selection

Under [ADR 0015](0015-provider-selection.md) agy is a selectable provider, detected by `agy` on
`PATH`. Setup offers it to users who have Gemini CLI configured for a personal account (the
ADR 0013 §10 warning).

## Probe so far

agy 1.5.14, downloaded through the official installer's manifest (sha512 matched, signed by
Google, Apple team `EQHXZ8M8AV`), run with `env -i`, a temporary `HOME` and
`AGY_CLI_DISABLE_AUTO_UPDATE=true`:

- without a login: `--version`, `--help`, `plugin --help`; the state tree above; `-p` started
  the OAuth flow and timed out after 60 s with a JSON error;
- one real login with a personal account: `-p` answered, the token landed in
  `antigravity-oauth-token` inside the temporary home, and the login keychain was untouched;
- a `PreToolUse` hook in `config/hooks.json` received the payload above and denied
  `git push --force origin main` under `--dangerously-skip-permissions`.

The temporary home holding the token was deleted afterwards.

## P0

Run on 2026-11-27 against agy 1.5.18 on macOS, again with `env -i`, a temporary `HOME`, one real
login and auto-update off; about 25 `-p` runs. Results are in the fact table and the decisions
above:

| # | Question | Answer |
|---|---|---|
| 1 | hook failure | non-zero exit, timeout, invalid JSON block; empty stdout with exit 0 passes; an unparsable `hooks.json` disables all its hooks silently |
| 2 | git environment | only `GIT_EDITOR`/`GIT_PAGER`; `pre-push` runs: layer 2 applies |
| 3 | command tools | `run_command`, `manage_task` (`send_input`) |
| 4 | `/usage` | free; JSON shape above; without a login, stderr says so at once, then 60 s |
| 5 | logout | none non-interactive (`/logout` refused in print mode) |
| 6 | `--settings` | replaces `permissions`; rule syntax `command(<prefix>)` confirmed |
| 7 | conversation copy | resumes in another home |
| 8 | terminal title | not set |

Still open, not blocking the core:

9. on Linux (a container with and without D-Bus): where the token goes with another `HOME`;
   until checked, `doctor` warns there as §3 says;
10. the main account on macOS: the keychain item's name and whether a second home's login
    touches it; checked when the first user sets up `agy/main` through squadrn.

## Plan

1. **P0** done (above); 9 and 10 remain.
2. **Core**: `HomeEnv`/`MainState`, the home-restoring environment (§2), file auth, login, logout,
   credential check, doctor (layer-2 status, shared-keyring warning), fake `agy`, `agy.txtar`.
3. **Guard**: the named hook, `--agent agy` with `force_ask`, the shim's fail-closed reply.
4. **Notifications and usage**.
5. **Profiles, MCP, swap**.

## Consequences

- Personal Google accounts get a supported path in squadrn again, through agy.
- agy accounts other than main run with a different `HOME`. Squadrn restores the common tools,
  and says plainly that others see the account's home until agy can relocate its config.
- On Linux and Windows, secondary agy accounts may share one login until P0 shows otherwise.
- The provider model gains `HomeEnv` and `MainState`, which no other provider uses.
