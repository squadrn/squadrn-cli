# Changelog

## 1.0.0-rc.6

- **Gemini CLI and personal accounts:** since 2026-06-18 Google refuses Gemini CLI for personal
  Google accounts, Google AI Pro and Ultra (*"This client is no longer supported for Gemini Code
  Assist for individuals"*); it serves Code Assist Standard and Enterprise licenses, paid API keys
  and Vertex AI. `account login`/`add` for a Gemini account say so before the login, and `doctor`
  warns about an account that signs in with Google without `GOOGLE_CLOUD_PROJECT`, pointing
  personal accounts to Antigravity CLI (`agy`). README and ADR 0013 record the scope.
- **Tab completion through the router:** `srn <Tab>` offers accounts by their short names first
  (`personal`, `main`; `provider/name` when ambiguous), described with the login last cached, then
  commands. After an account it offers `wt`, `grid` and the agent CLI's own subcommands
  (`srn personal <Tab>`, `srn personal wt <Tab>` completed nothing before). `-a` completes
  accounts and `-C` directories; `config set` values, `grid` arguments, worktree names of the
  account's provider. Completion never runs an agent CLI.
- **`srn completion install|uninstall`:** writes the scripts where the shell loads them (a zsh dir
  already on `fpath`, bash-completion's dir, fish's), adding a marked block above `compinit` in
  `~/.zshrc` (or to `~/.bashrc`) only when needed and after asking. New setting
  `completion = auto|off`: updates refresh the scripts unless off; they never edit startup files
  and say `completion install` when the shell does not load the dir. `doctor` warns when the login
  shell's completion is missing, out of date or not loaded; `self-uninstall` removes the block.
- **Guided account creation** ([ADR 0014](docs/adr/0014-account-creation.md)): `srn account add`
  without a name, at a terminal, asks the provider (installed CLIs with their version and account
  count; missing ones dimmed with their install command), the name (checked as you type, with the
  dir, the start command and a warning when another provider uses the name), then Recommended or
  Customize (profile, options, usage tracking, default account). A taken name offers repair, an
  unmanaged dir adoption, `main` the default account's setup. Nothing is written before the
  review; the login is offered once the account exists, and the end prints the same command
  without questions. Esc goes back, Ctrl-C cancels (exit 130) and changes nothing. Without a
  terminal the name is required (exit 2). The account picker's "+ add a new account" runs it too.
- `srn setup` offers the wizard ("Add an account?", then "Add another account?") as its new step 4;
  usage tracking and doctor become steps 5 and 6.
- At a terminal that understands ANSI, every question of `setup` and `account add`/`repair` is drawn
  in place: arrow keys or j/k, Space, Enter, y/n, Esc back, Ctrl-C cancel; answered questions
  collapse to one line. Other terminals keep the line prompts. Glyphs fall back to ASCII when the
  locale is not UTF-8, `TERM=dumb` or the Windows console code page is not 65001.
- `account add`/`repair --dry-run` prints the plan and changes nothing; `--json` prints one result
  object (or `{"error", "fix"}` with the usual exit codes) and asks nothing.
- **Breaking:** `account add` only creates an account; on an existing one it fails with
  `<account> already exists (repair: srn account repair <account>)`. The new `account repair
  [<account>|--all]` brings accounts up to date (what re-running `add` did, including `--relink`
  and switching `--profile`); scripts that re-ran `add` must call `repair`. The installer,
  `self-update` and `doctor` hints name `repair` (ADR 0014).
- **Breaking:** `account adopt` is removed, with no alias: `account add <name> --adopt` takes over
  an existing `~/.<provider>-<name>` dir (ADR 0014).
- `account default [<account>|--unset]` and the config key `default_account`: a bare `srn` (and an
  agent subcommand without an account) runs that account instead of the default provider's main
  account; `doctor` warns when it names an account that no longer exists. `account add --default`
  sets it.
- `account add -p/--provider <id>` names the provider (it must agree with one in the name);
  `--with` takes comma-separated values.

## 1.0.0-rc.5

- **Gemini CLI accounts** (`gemini/<name>`, [ADR 0013](docs/adr/0013-gemini-provider.md), core):
  `~/.gemini-<name>` is the account's `GEMINI_CLI_HOME`, with Gemini's files in its `.gemini/`;
  `settings.json` is the account's own copy of main's (comments are read, never written back to
  main's file), `GEMINI.md`, `commands`, `agents` and skills are shared from `~/.gemini`. The
  login is read from the account's files (`oauth_creds.json`, or an API key in the environment);
  `account login` runs `gemini` with a hint to finish the Google login and type `/quit`, and
  `account logout` moves the login files to the trash. Sessions never use Gemini's shared
  keychain store. `doctor` checks Gemini 0.70.0+, Node.js 20+, the keychain and Google Cloud
  credential variables, and says that Gemini sessions have no git pre-push guard (Gemini turns
  git hooks off).
- **Gemini profiles, MCP, trust and worktrees** (ADR 0013 §7–8): every profile keeps the git and
  tmux variables Gemini must not redact, turns worktrees on and writes a policy file
  (`squadrn-policy.toml`, passed with `--policy` to every session) that denies Gemini's file tools
  `.env` files, `~/.ssh` and every other account's credential files; `standard` denies the full
  secret list, asks before `git push`, turns yolo mode and auto-update off and keeps 30 days of
  history, `strict` 14 days and approval for every tool; `--with privacy` turns usage statistics
  and telemetry off. `shared_mcp` copies main's allowlisted servers into Gemini accounts. A
  session in a repository the account has no trust decision for asks once and trusts the
  repository root (and its worktrees). `srn wt` and grids run `gemini --worktree <name>`.
- **Push guard for Gemini CLI** (ADR 0013 §5): `account add`, the installer and `self-update`
  write a `BeforeTool` hook (`squadrn-guard`, shell tool only, 5 s) into each Gemini account's own
  `settings.json`; it blocks with exit 2, which Gemini honours even in yolo mode. A main
  `settings.json` with comments is left alone and the hook to add is printed. `doctor` reports a
  hook that is missing, turned off or changed. `self-uninstall` removes it.
- **Notifications for Gemini CLI** (ADR 0013 §6): `BeforeAgent`, `AfterAgent` and `Notification`
  (tool approval) hooks report working, done and needs-input; they always exit 0 with no output
  (Gemini would otherwise retry the turn) and deliver in the background. Gemini's own
  notifications are turned off while squadrn's are on. `srn ps` reads the state from the title
  Gemini gives its terminal (✋ input, ⏲ working, ◇ ready) when no hook reported one.
- When squadrn itself is missing, the plugin's fail-closed reply is now exit 2 with the reason on
  stderr, for every agent CLI, instead of Claude Code's JSON (which Gemini ignores). What is
  blocked does not change.
- Every profile denies reading all of another account's credential files, Gemini's included, and
  squadrn's own hook entries in main's Claude Code `hooks` are no longer copied into accounts.
- `srn account add codex/main` (or `claude/main`) sets up the default account instead of refusing
  it as a reserved name: it installs the push guard, asks about usage tracking and the login, and
  leaves its settings alone. `doctor` suggests it when the default account has no guard, instead of
  re-running the installer.

## 1.0.0-rc.4

- **Notifications rewritten** ([ADR 0012](docs/adr/0012-notifications.md)): one notification when
  an agent needs you (approval, question, reply), failed (usage limit, login, billing, server) or
  finished, titled *Claude Code · work · api/fix-auth*, with the command to run
  (*Usage limit · resets 15:45 · srn swap*). Nothing while you look at the session (tmux pane in
  view; on macOS `done` while its terminal is frontmost), at most one per session every 30 s
  except input, a newer one replaces the older, no repeated idle prompt after *Done*, and the
  agent's own text only with `notify.preview`. Was: *Claude: Done* after every turn.
- `[notify]` settings (`enabled`, `events`, `sound`, `when`, `quiet_hours`, `preview`, `method`),
  `srn notify status` and `srn notify test`, and a `doctor` line.
- macOS uses `terminal-notifier` when installed (grouped per session, click brings the terminal
  forward), else `osascript`; Linux replaces a session's notification and marks input urgent
  (`gdbus` when `notify-send` is missing); **Windows** shows toasts (app id registered by
  `self-install`).
- Account profiles turn Claude Code's own notifications off while squadrn's are on, so you no
  longer get both.
- iTerm2, WezTerm, Ghostty, Warp and kitty show input and failures themselves (terminal
  notification sequences) outside tmux. In tmux each agent's pane carries its state (grid borders
  colour it) and rings the bell when it needs you; **`srn next`** jumps to that pane, and a click on
  a `terminal-notifier` notification switches to it.
- **Codex accounts notify too:** approvals, finished turns and the agent's state through squadrn's
  Codex hooks (installed and trusted with the push guard) and Codex's `notify` program; Codex's own
  terminal notifications are turned off in account profiles.
- **`srn ps` lists every running agent**, also sessions started in a plain terminal, with a
  `STATE` column (`working`, `input`, `failed`, `done`). Columns are now ACCOUNT, WORKTREE, STATE,
  SESSION, PROCESS.
- **`srn account rm <account> --purge`:** logs the account out, deletes its dir permanently
  instead of moving it to the trash (links inside are removed, never followed; a linked dir is
  refused) and removes every record squadrn keeps of it: usage record, handoff ledger entries,
  in-use records, config change log entries and cached login status. The output lists what was
  removed. At a terminal it asks you to type the account unless `-y` is given; without one it
  needs `--purge --yes`. Works for Claude Code and Codex accounts.
- Fixes from a walk through every command:
  - The installer's first-run `doctor` checks the installed `srn`/`squadrn`, not the download.
  - `srn work` with `claude/work` and `codex/work` says the name is ambiguous instead of
    unknown; `swap --to work` picks the account that can take the conversation.
  - `account add --no-usage` is honoured when the main account tracks usage; `--relink` is asked
    on the plan, before the login; "done in" leaves out time spent on questions and the login;
    a main account without settings is no longer named as their source.
  - Files and dirs main gains later (skills, agents, …) are linked into existing accounts.
  - `wt ls` shows agents started in a plain terminal as running.
  - Windows: prompts only on a console you can see; a hidden console gets the
    needs-a-terminal error instead of waiting forever. `account rm --purge --yes` never prompts.
  - `setup` offers the self-protection deny rules and indents its questions and output;
    `completion` writes for the login shell on a fresh home; `gc` says when there is nothing to
    remove; `config` shows empty values as `-`; `doctor` lists each agent CLI once and marks
    missing optional tools as skipped; `self-uninstall` words Codex accounts and says what it
    keeps; clearer guard refusals, help text and account name rule.

## 1.0.0-rc.3

- **OpenAI Codex CLI:** `codex/<name>` accounts (`CODEX_HOME`, credentials kept per account),
  login, runs, `--worktree`, resume and swap between Codex accounts, MCP, usage limits in
  `srn usage` (read from Codex, cached for 5 minutes), and the push guard as a Codex pre-tool hook
  plus the git pre-push hook. Codex cannot ask, so its guard denies what it would ask about; on
  Windows a hook that fails to start lets the command run (`srn doctor` says so).
- **Profiles:** `srn account add` offers `minimal`, `standard` (default) and `strict`, with the
  `privacy` and `no-attribution` toggles (`--profile`, or `default_profile` in the config). The
  standard profile denies reading secrets (`.env`, `~/.ssh`, cloud and package credentials, the
  password store, other accounts' credential files) and asks before `git push`, `git reset --hard`
  and `rm -rf`.
- **Own settings per account:** an account's `settings.json` / `config.toml` is now its own copy
  built from main's settings and its profile, never a link. Squadrn keeps only the keys it owns in
  sync and keeps your own changes (`/model`, `/config`). Existing links are turned into copies on
  the first session (backup kept).
- **`account add` rewritten:** it plans, asks everything up front, applies with progress and
  verifies; a re-run says the account is up to date; a failed step leaves nothing half-created and
  names the retry command. A first add takes about 1.5s with Claude Code (was 4.2s), a re-run
  0.3s.
- **`srn setup`:** a guided first run (agent CLIs found, default provider and profile, adopting
  existing account dirs, usage tracking, doctor). The installer runs it after the first install at
  a terminal; `self-update` never does.
- **Windows:** the push-guard hook now starts `squadrn.exe` directly and blocks when it cannot run
  (Claude Code 2.1.295 or newer; doctor warns on older ones). With rc.1/rc.2 every Bash and
  PowerShell tool call could be blocked. Git for Windows is found under Git Bash's own `git.exe`
  too, and alias copies are recognised after a rename.
- Account names only exclude squadrn's own commands; agent subcommands go through
  `srn <account> <subcommand>`.

## 1.0.0-rc.2

- **Breaking:** the default command is now `srn`, with `squadrn` as its alias (was `squadrn` with
  the alias `srn`); the installers and `self-install --interactive` offer the new defaults. Existing
  installs keep their configured names; to switch, run `squadrn config set command srn`, then
  `srn config set aliases squadrn`. Making an alias the command (`config set command`, the
  installer's `--command` or its prompt) swaps the two names instead of failing.
- `doctor` recommends only `Edit(…)` deny rules for squadrn's files: Claude Code warned at every
  start that the `Write(…)` rules it also suggested are not matched. Settings that have the `Edit`
  rules pass, with or without `Write` ones; remove the `Write(~/.local/share/squadrn/**)` and
  `Write(~/.config/squadrn/**)` rules to silence the warning.
- **Breaking (JSON):** `grid ls --json` and `ps --json` use snake_case keys (`name`, `repo`,
  `panes`, `attached`; `id`, `session`, `account`, `name`, `command`); `config --json` and
  `config get --json` give lists as arrays and `git_hooks`/`update_check` as booleans.
- `wt rm` decides before removing anything: an unmerged branch is refused up front, or with
  `--force-branch` the worktree and branch go together ("deleted branch …"); a branch left without
  its worktree follows the same rule (merged: deleted; unmerged: `--force-branch`). Refusals name
  the real reason and the worktree.
- `account add` creates main's `settings.json` (`{}`) when it is missing, so accounts always link
  to it; `account add <account> --relink` moves an account's own copies of shared files aside
  (`.bak-<time>`) and links main's, and is what `doctor` and the session warning suggest.
- `account login` says who is logged in; an unmanaged `~/.claude-<name>` gets `account adopt
  <name>` suggested; `account rm` uses `~/.Trash` on macOS.
- `usage`: the current account counts for "best for this repo", resets not today show the weekday,
  logged-out accounts are not listed as untracked; `usage setup --all` asks once for every file.
- `config set` prints the value that applies (defaults marked); alias changes no longer read
  `command: srn → srn`; invalid names are reported once.
- `completion <shell>` covers the command and every alias, and an unknown shell is a usage error;
  `grid attach` explains a missing terminal and keeps tmux's own connection errors.
- `self-uninstall` checks that the marketplace registration is gone; `--purge` also removes the
  session registries it knows of. Development builds check `squadrn/squadrn-cli` for updates, report
  "no release yet" instead of failing, and call themselves `dev`.

## 1.0.0-rc.1

First public release of Squadrn: run Claude Code as several of your own accounts, in parallel git
worktrees, with a guard against destructive pushes. Apache-2.0.

- **Accounts:** `squadrn <account>` runs Claude Code as `provider/name` (`claude/main`,
  `claude/work`, …); `account add|login|logout|ls|rename|rm|adopt`; settings, skills, plugins and MCP
  servers shared with the main account.
- **Parallel work:** `wt <name>` runs an agent in its own worktree; `grid` lays several out in tmux
  (macOS, Linux); `ps`, `gc` (never deletes unmerged or in-use work), `deps`.
- **Usage limits:** `usage` shows each account's 5-hour and weekly usage from Claude Code's status
  line (`usage setup`), with the best account for the current repository.
- **Swap:** `swap [--session <id> | --worktree <name>] [--to <account>]` continues a conversation on
  another of your accounts, after stopping the old session; checkpoints for `/rewind` move with it.
- **Push guard:** a Claude Code hook (Bash and PowerShell) and a git pre-push hook stop force pushes
  and pushes to protected branches (`main`, `master`, `develop`, `release/*` by default, plus
  `git config squadrn.protectedBranches`); `doctor` checks the guard and server-side protection. A
  safety net, not a sandbox.
- **Install and update:** one-line installers (`curl … | sh`, `irm … | iex`) that verify the release
  signature and checksum (`--command`/`-Command`, `--alias`/`-Alias`, `--version`/`-Version`);
  `self-update [--tag <tag>]` (stable or beta channel), `--rollback`, `self-uninstall`; your own
  command name and alias (`squadrn`/`srn` by default).
- **Platforms:** macOS and Linux (amd64, arm64); Windows 10 21H2+/11 (amd64, arm64) in beta, without
  `grid`. Git 2.31 or newer.
- Release signing key `squadrn-release-20261119`.
