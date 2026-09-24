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
FENCED_MISE_TOOLS=(gh playwright)   # the default
```

## Tools inside the fence

Omarchy's `~/.local/bin/<tool>` scripts are mise wrappers that run
`mise use -g` (update or install) before starting the tool. That needs write
access to mise's config, installs and caches, which the fence keeps read-only
because those binaries also run unfenced. So inside the fence, PATH has mise's
real bin dirs (`mise bin-paths`, for the work dir) first and `~/.local/bin`
last. Tools in `FENCED_MISE_TOOLS` that aren't installed yet (playwright's
wrapper installs on first use) are installed by the wrapper before the fence
starts. Updating them is left to normal, unfenced use. A tool that is neither
installed nor listed won't run inside: add it to `FENCED_MISE_TOOLS`, or run it
once outside.

## Secrets and per-project tokens

No credentials are visible by default: no `gh` auth, SSH keys or agent,
cloud credentials or keyring. If an agent should push or deploy, give that
project a narrowly scoped token, e.g. a fine-grained GitHub PAT for just that
repo. Anything the agent can push to or deploy with is as good as write access.

### GitHub token

```bash
cd ~/Projects/projectA
fenced gh-token
```

This shows a link to GitHub's "new fine-grained token" page, pre-filled with
name (host, repo, date), owner, expiry (365 days, or `FENCED_TOKEN_DAYS`) and
permissions: contents, pull requests, issues and discussions (write), actions
and statuses (read). The link is meant to work over ssh and inside herdr: it's
copied to your *local* clipboard (OSC 52), shown as a short clickable link
(OSC 8), and printed in full. GitHub has no API for creating tokens, and the
link can't pre-select the repository, so on that page pick *Only select
repositories* → the repo, and generate.

Then paste the token at the prompt. It's saved only after it's verified to
have push access to the repo. Pasting when a token is already set replaces it,
which is also how you renew one. The token is stored for the main repo root, so
worktrees and subdirs share it.

The token deliberately lacks **workflows** write access. With it the agent
could edit `.github/workflows` and run arbitrary CI with the repo's secrets
(publish/deploy credentials, a possibly more powerful `GITHUB_TOKEN`,
unprotected environments). Without it, pushes that touch workflow files are
rejected; push those yourself, or make a token with it:
`FENCED_TOKEN_WORKFLOWS=1 fenced gh-token`. This isn't airtight: an existing
workflow that runs repo code (tests, build scripts) with secrets in its env can
still be abused through ordinary code changes. Only secrets behind protected
branches or environments with required reviewers are out of reach.

### Other variables

Pressing just Enter at the `fenced gh-token` prompt opens the project's env file
(`~/.config/claude-fenced/projects/<project path>.env`) in a terminal editor.
That's `FENCED_EDITOR`, else `$VISUAL`/`$EDITOR`; Omarchy's
`omarchy-launch-editor` is replaced by `nano`, because it starts GUI editors
like VS Code detached on the desktop. You can also edit the file directly, and
create one for a non-GitHub dir by hand (`mkdir -p` + `chmod 600`).

```bash
GH_TOKEN=github_pat_...                 # plain values
NPM_TOKEN=$(pass show npm/projectA)     # or pulled from a secret manager (sourced by bash)
```

How it works:
- The files mirror the project path (mode 600) and are sourced by bash
  **outside** the fence. Only the resulting variables go in. The directory is
  invisible inside, so projectA's agent can't read projectB's tokens.
- Files for the work dir and every parent directory are loaded, outermost
  first. A worktree also gets its main repo's file. Subdirs and worktrees
  inherit the project's token, and a `~/Projects.env` could hold shared values.
- With `GH_TOKEN` set, `gh` uses it, and git gets a credential helper for
  `https://github.com` plus a `git@github.com:` → https rewrite, all via
  `GIT_CONFIG_*` env vars. `git push` works with no gitconfig changes.
- It's automatic on every launch, so Orca, `claude -c` and so on get it too.
  The startup summary lists which files and variable names were loaded.
- The agent can of course read its own token (it's in its environment).

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
