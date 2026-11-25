# agent-sandbox

Run AI agents (opencode, claude, codex, crush) in a [bubblewrap](https://github.com/containers/bubblewrap) sandbox. The agent sees only what you explicitly allow — your real `$HOME`, SSH keys, other projects, and host sockets stay hidden.

## What the agent sees

| Visible | Access |
|---|---|
| Current project directory | read/write |
| `/usr`, `/bin`, `/lib*` | read-only |
| Minimal `/etc` subset (`ETC_ITEMS`) | read-only |
| Sandbox HOME (per-project) | read/write |
| Shared auth tokens | read/write |
| Agent binary (if in `$HOME`) | read-only |

**Hidden:** real `$HOME`, SSH keys, `~/.gitconfig`, other projects, `/media`, `/mnt`, `/run/user` (ssh-agent, D-Bus, X11/Wayland), host processes.

## Install

```bash
sudo apt install bubblewrap passt
chmod +x agent-sandbox && sudo mv agent-sandbox /usr/local/bin/
```

`bubblewrap` required; `passt` recommended (enables `--net isolated`). Without it, network is shared with the host.

**Ubuntu 24.04+ (AppArmor):** if `bwrap` can't create namespaces, the script prints a profile. Apply it:

```bash
sudo tee /etc/apparmor.d/bwrap >/dev/null <<'PROFILE'
abi <abi/4.0>,
include <tunables/global>
profile bwrap /usr/bin/bwrap flags=(unconfined) {
  userns,
  include if exists <local/bwrap>
}
PROFILE
sudo apparmor_parser -r /etc/apparmor.d/bwrap
```

## Usage

```bash
cd ~/projects/myproj
agent-sandbox                        # opencode
agent-sandbox -- opencode run "fix the tests"
agent-sandbox -- claude              # another agent
agent-sandbox --shell                # bash inside sandbox
agent-sandbox --net host             # shared network (e.g. ollama)
agent-sandbox --dry-run              # print bwrap command
```

### Options

| Option | Description |
|---|---|
| `-n, --net MODE` | `auto` (default): `isolated` if `passt` installed, else `host`. `isolated` = own netns via `pasta`, internet works but no host `localhost`/abstract sockets. `host` = shared network. `none` = no network. |
| `-p, --allow-port PORT` | In `isolated`, allow `127.0.0.1:PORT` on host. Proxy ports from `HTTP(S)_PROXY`/`ALL_PROXY`/GNOME settings are auto-allowed. |
| `--ro PATH[:DEST]` | Extra read-only mount. |
| `--rw PATH[:DEST]` | Extra read/write mount. |
| `-e, --env NAME[=VAL]` | Pass/set env var. |
| `--home DIR` | Use `DIR` as sandbox HOME (e.g. shared across projects). |
| `--shell` | Run `bash` instead of agent. |
| `--dry-run` | Print `bwrap` command and exit. |
| `-h, --help` | Help. |
| `-- CMD [ARGS...]` | Command to run (default: `opencode`). |

## Sandbox HOME & auth tokens

Per-project HOME at `$XDG_DATA_HOME/agent-sandbox/homes/<name>-<sha256(path)[:12]>/` (default `~/.local/share/agent-sandbox/`, override with `AGENT_SANDBOX_ROOT`).

These files are shared across all projects via symlinks into `shared/` — **log in once**:

- `~/.local/share/opencode/auth.json`
- `~/.claude/.credentials.json`
- `~/.codex/auth.json`

## Network

- **isolated** (recommended): own netns, NAT via `pasta`. Internet works; host `localhost` and abstract sockets unreachable. Only explicitly allowed host ports are forwarded.
- **host**: shared network — needed for e.g. `ollama` on `localhost`, but exposes all local services and X11/D-Bus abstract sockets.
- **none**: no network.

Proxy: if set only in GNOME (`gsettings`), it's copied into env vars. `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, `NO_PROXY` (both cases) are honored.

## Environment

Passed through: `TERM`, `COLORTERM`, `TERM_PROGRAM`, `LANG`, `LANGUAGE`, `LC_*`, `TZ`; `HTTP(S)_PROXY`, `ALL_PROXY`, `NO_PROXY`; `NODE_EXTRA_CA_CERTS`, `EDITOR`, `VISUAL`; `*_API_KEY`, `*_BASE_URL`, `OPENCODE_*`, `ANTHROPIC_*`, `OPENAI_*`; git author name/email (from global config, not the file). Everything else is cleared (`--clearenv`). Add more with `-e`.

## Agent binaries

If the agent lives in your `$HOME` (`~/.opencode/bin`, `~/.npm-global`, `~/.local/bin`), only the ELF binary is mounted read-only into `/run/agent-sandbox/bin` — not its install directory. Script wrappers (e.g. node) can't be passed this way; the script warns and suggests `--ro` or installing the agent inside the sandbox (`agent-sandbox --shell`).

## Security

- Sandbox root is read-only (`--remount-ro /`); writes only to explicitly mounted paths.
- Separate `--proc`, `--dev`, `/tmp` (tmpfs), `/var/tmp` (tmpfs), `/run`.
- Generated `passwd`/`group` contain only your user and `nobody`.
- `--unshare-user`, `--unshare-pid`, `--unshare-ipc`, `--unshare-uts`, `--unshare-cgroup-try`.
- TIOCSTI guard: `--new-session` unless kernel has `dev.tty.legacy_tiocsti=0` (TUI resize may not work with `--new-session`).
- `isolated` mode uses `--no-map-gw` — pasta gateway not mapped to host loopback.

> **Note:** the script refuses to run from `/`, your `$HOME`, or inside `$STATE_ROOT` — too much would be exposed.

## Dependencies

`bash` 4+, `bubblewrap`, `passt` (optional but recommended), coreutils (`sha256sum`, `readlink`, `od`, `mktemp`), `gsettings` (optional, GNOME proxy detection), `git` (optional, commit identity).

## License

Provided as-is. Isolation reduces risk but doesn't eliminate it — don't give the agent more privileges than you need.
