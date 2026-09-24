# claude-fenced — context for agents

## What this is

`fenced` is a bash wrapper that runs coding agents (claude, codex, gemini, ...)
in yolo mode inside a [fence](https://github.com/fencesandbox/fence) sandbox
(bubblewrap + Landlock + seccomp on Linux). The goal: never block on permission
prompts, because the agent can only reach the project it was started in.
User-facing docs are in `README.md`.

Sibling project `../claude-contained` does the same with Apple Containers or
Docker on a work Mac. This project targets a dedicated Linux box (Omarchy, Arch)
used for agentic coding, reached over Tailscale. Almost all of claude-contained's
complexity (images, per-folder Codex homes, flock supervisors) exists because
file locks don't work across virtiofs. fence bind-mounts the real host fs, so
that doesn't apply here. Don't port it over.

## Layout

- `fenced`: the whole implementation
- `fence-shell.rc`: rc file for the interactive `--fence-shell`. `~/.bashrc`
  is unreadable in the fence, and fence strips `PS1` from the environment.
- `bin/{claude,codex,gemini}`: symlinks to `../fenced`; tool = `basename $0`.
  Installed by putting `bin/` first on PATH (ahead of `~/.local/bin`).

## Requirements (from the user)

- Only the cwd plus explicitly granted ro/rw folders; projectB must be
  invisible while working on projectA. Secrets are the main read concern.
- Full network in and out. Inbound matters: dev servers are opened from a
  laptop via Tailscale.
- Tool config and history shared with normal, unfenced use.
- Drop-in `claude`/`codex` names; Orca (github.com/stablyai/orca) must still
  recognise the agent.
- Remote control: done via `remoteControlAtStartup: true` in
  `~/.claude/settings.json`, not by the wrapper.

## How it works (fenced, top to bottom)

1. **Resolve the real binary.** Walk PATH, skipping ourselves. Omarchy's
   `~/.local/bin/<tool>` is a mise wrapper (`mise use -g` to update, then
   `mise x`). When found, the wrapper runs the update step unfenced, then
   `mise which <tool>`. The real binary keeps basename `<tool>`, which Orca needs.
2. **Tool profile.** Sets the yolo flag, the state dir (rw), and "protect"
   paths (rw state made read-only again: hooks, settings, plugins, statusline,
   codex `config.toml`). Those would otherwise let a fenced agent get code run
   by a later, possibly unfenced, session.
3. **Inside a fence already** (`FENCE_SANDBOX` set, e.g. the agent runs
   `claude -p`): exec the real tool directly. Nested bwrap is not attempted.
4. **Parse `--fence-*` options.** Only leading ones, so prompts aren't mangled.
   Tool `--add-dir`/`--cd` values also get rw access.
5. **Build the fence JSON** with jq and pass it via `--settings /dev/fd/3`.
   Settings are always explicit, because otherwise fence auto-loads a
   `fence.json` from the cwd or its parents, and a repo could ship one.
6. **Per-project env**: source `~/.config/claude-fenced/projects<dir>.env`
   for the work dir and its ancestors (plus the worktree main repo's chain),
   outside the fence. `GH_TOKEN` also sets up git via `GIT_CONFIG_COUNT/KEY/VALUE`
   (a credential helper that echoes the token, and ssh→https `insteadOf`).
   Verified: `git credential fill` returns the token inside, and the env dir
   is invisible inside.
   `fenced gh-token` sets it. GitHub has no PAT-creation API (as of
   2026-09), so it shows a prefilled `settings/personal-access-tokens/new?...`
   URL (via OSC 52 clipboard, an OSC 8 link, and plain text; used over ssh/herdr) (name ≤40 chars and unique, `target_name`, `expires_in`, permission
   params; `expires_in` max is 365 in practice, though the docs say 366).
   Repository selection can't be prefilled. The pasted token is checked with
   `gh api repos/O/R` (`.permissions.push`) before it's saved. Enter alone
   opens the env file in a terminal editor, with `omarchy-launch-editor`
   replaced by nano, since it starts GUI editors detached.
   Not `fenced gh`: `fenced <tool>` runs a tool.
7. **`exec -a <tool> fence ...`**, so argv[0] is the tool name for Orca's fast path.

## Verified facts (fence 0.1.67, kernel 7.2, 2026-09-25)

Re-verify these if fence is upgraded.

- `defaultDenyRead: true` really hides `$HOME`: unlisted paths don't exist
  inside. System dirs, `~/.local/bin` and `~/.npm/_logs` are added by fence itself.
- `allowedDomains: ["*"]` means **no network namespace**. Outbound is direct,
  including node (which ignores the proxy). Listening sockets bind on the host:
  a server on `0.0.0.0` inside was reachable on the Tailscale IP with no `-p`.
  Consequence: host localhost services are reachable too.
- **Escape found and fixed:** fence ro-binds all of `/run`, but ro mounts don't
  stop `connect()` on sockets. `systemd-run --user` over the session D-Bus ran a
  command outside the sandbox. `denyRead: ["/run/user/$UID"]` blocks it, and the
  wrapper also unsets `DBUS_SESSION_BUS_ADDRESS`. The abstract X11 socket was
  refused. `/run/docker.sock` is safe only because the user is not in the
  `docker` group; revisit if that changes.
- A symlink in `allowRead` is **not** reproduced inside: the path is simply
  absent. That, plus claude creating `~/.claude.json.lock` and
  `~/.claude.json.tmp.*` next to the file (with an unlocked in-place write as
  fallback when `$HOME` isn't writable), led to the `~/.claude/.claude.json`
  relocation plus `CLAUDE_CONFIG_DIR`. Checked: unfenced claude resolves the
  symlink and writes temp files next to the target, keeping the symlink.
  Fenced claude logs "written atomically".
- claude's `-p` works, auth works, and remote control only needs HTTPS.
  Both yolo flags are accepted before subcommands (`claude mcp list`, `codex login`).
- Orca detection (source-read plus PTY simulation, not tested in a real Orca pane):
  Orca needs `allowPty: true`. Without it bwrap uses `--new-session` and the tool
  loses the foreground `+` in `ps`, so Orca ignores it.
- Git worktrees: committing from inside works once the main repo's
  `--git-common-dir` is granted rw.
- Claude's `uds-messaging` falls back to a private `/tmp/cc-socks-0`
  (cross-session messaging is isolated; intentional).

## Deliberate choices / open items

- Package caches are rw, a cross-project poisoning risk accepted for
  usability. Dirs whose binaries run unfenced (`~/.cargo/bin`, mise installs,
  `~/.cache/mise` downloads) stay ro or hidden.
- herdr's socket is opt-in (`--fence-herdr`), because its API can spawn panes.
  Exposing the socket via `allowWrite` is untested.
- `~/.claude.json` stays writable and can hold MCP server commands, so
  poisoning it remains possible.
- Codex remote control (`codex remote-control start`, an experimental daemon)
  is not wired up.
- No automated tests yet. Test manually with `--fence-dry-run` and
  `--fence-shell -c '...'` (e.g. `ls ~/Projects`, `timeout 5 systemd-run --user true`).
