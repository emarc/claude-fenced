# claude-fenced

Run `claude`, `codex` (or `gemini`, ...) in yolo mode, sandboxed with
[fence](https://github.com/fencesandbox/fence) so the agent can only touch the
project you started it in. No container: the agent uses the real host
filesystem, toolchains and network, so there are no image builds, no file-lock
problems and no state syncing (compare `../claude-contained`).

```
cd ~/Projects/projectA
claude                                   # projectA read-write, projectB invisible
claude --fence-ro ../shared-lib          # plus a read-only folder
claude --fence-rw ../projectA-docs -c    # plus a read-write folder, then `claude -c`
codex                                    # same thing for codex
claude --fence-shell                     # bash in the same sandbox, to look around
claude --fence-dry-run                   # show the generated fence config
```

## Install

Requires `fence`, `bubblewrap`, `socat` and `jq`.

```bash
# fence: official release binary (the AUR `fence` package lags behind)
curl -fsSL https://cli.fencesandbox.com/install.sh | sh    # or: yay -S fence
sudo pacman -S --needed bubblewrap socat jq

# Put the wrapper's bin/ ahead of ~/.local/bin (Omarchy's mise wrappers)
echo 'export PATH="$HOME/Projects/claude-fenced/bin:$PATH"' >> ~/.bashrc
```

`bin/claude`, `bin/codex`, `bin/gemini` are symlinks to `fenced`; the tool is
picked from the name it was called as. For another tool, add a symlink (it gets
the generic profile: no yolo flag, `~/.<tool>` read-write) or a profile in `fenced`.

To bypass the fence for one run: `FENCED_OFF=1 claude`.

## What the agent can see

| Access | Paths |
|---|---|
| read-write | current directory; the main repo's `.git` when in a git worktree; `--fence-rw` / `--add-dir` dirs; tool state (`~/.claude`, `~/.codex`, `~/.gemini`); package caches (`~/.npm`, `~/.m2/repository`, `~/.gradle/caches`, `~/.cargo/registry`, ...) |
| read-only | `--fence-ro` dirs; mise tool installs and config; git config; system dirs (`/usr`, `/etc`, ...) |
| read-only inside writable state | tool config that would run code in *later* sessions: `~/.claude/{settings.json,hooks,plugins,skills,statusline*,...}`, `~/.codex/{config.toml,skills,...}` |
| invisible | everything else in `$HOME` (`~/.ssh`, `~/.config/gh`, `~/.aws`, other projects, ...), `/run/user/$UID` (D-Bus, keyring, Hyprland, gnupg sockets) |
| network | unrestricted in and out; servers bind on the host, so a dev server on `0.0.0.0:PORT` is reachable over Tailscale |

The wrapper refuses to run in `$HOME` or `/`, or to grant them.

Extra dirs can also come from `FENCED_RO` / `FENCED_RW` (colon-separated) or
`~/.config/claude-fenced/config`:

```bash
FENCED_RO_DIRS=(~/Projects/shared-lib)
FENCED_RW_DIRS=()
```

## Secrets

No credentials are visible by default: no `gh` auth, SSH keys or agent,
cloud credentials or keyring. If the agent should push or deploy, give it a
narrowly scoped token for that project (e.g. a fine-grained GitHub token for one
repo) via env var, rather than exposing `~/.config/gh`. Anything the agent can
push to or deploy with is as good as write access, so scope accordingly.

## Integrations

- **Remote control**: `remoteControlAtStartup: true` in `~/.claude/settings.json`
  works inside the fence (needs only outbound HTTPS).
- **Orca** recognises the fenced agent: it looks for a foreground process whose
  argv[0] basename is `claude`/`codex`. `allowPty` keeps the real binary in a
  foreground process group, and fence itself runs with argv[0] = the tool name.
- **herdr**: its status hook talks to `~/.config/herdr/herdr.sock`, which is
  hidden by default because herdr's socket API can open panes and run commands
  outside the fence. `--fence-herdr` / `FENCED_HERDR=1` exposes it (untested).

## Caveats

- **`~/.claude.json` is moved** to `~/.claude/.claude.json` on first run, with a
  symlink left at the old path. Claude writes it via a lock and temp file next
  to it, which can't be created in `$HOME` inside the fence. Unfenced claude
  follows the symlink, so both keep using the same file. Inside the fence
  `CLAUDE_CONFIG_DIR=~/.claude` points claude at it directly.
- `/tmp` is private per session, so you can't hand files over through `/tmp`.
- Fence always write-protects `.git/hooks`, shell rc files and `.vscode`/`.idea`
  dirs, so e.g. `husky install` fails.
- Shell rc files are empty inside the fence, so the environment comes from
  the shell that launched the wrapper.
- Claude's cross-session messaging (`/run/user/$UID/cc-socks`) is unavailable,
  so fenced sessions can't message other sessions.
- Changing settings from inside a fenced claude (`/config`) fails for
  `settings.json`. Edit it from outside.
- `~/.claude.json` must stay writable and can define MCP servers, so a
  malicious agent could still plant a command that a later session runs. That
  later session is also fenced when it's started through this wrapper.
- fence is defense-in-depth against a misbehaving or prompt-injected agent,
  not a hard boundary against targeted kernel exploits.
