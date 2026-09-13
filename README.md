# dsh-ssh-remote

[![license](https://img.shields.io/github/license/yc-csu/dsh-ssh-remote)](LICENSE)
[![dsh-plugin](https://img.shields.io/badge/topic-dsh--plugin-7a3ef3)](https://github.com/topics/dsh-plugin)

**Remote-work assistant for [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) (DSH) — SSH 远程连接服务器插件。**

Fork of [`dsh-remote`](https://github.com/flymysql/dsh-remote) by [flymysql](https://github.com/flymysql), enhanced with:

- **交互式 SSH 终端（WebSocket PTY）** — 在 Web GUI 里直接打开远程机器的交互 shell（xterm.js 渲染），与 agent 的 exec/sftp 连接池相互独立
- **密钥口令（passphrase）支持** — 加密私钥也能在设置页填写口令后连接
- 完整的远程工作区流程：多机器 SSH 注册表 → 选远程目录 → 本地镜像（SFTP 双向同步）→ `rw_*` 工具直接操作

Manage several SSH machines, then pick a **remote workspace** and let the agent operate right there without leaving the harness — listing files, reading code, running builds & commands over the remote host, and keeping that remote directory mirrored into a real local workspace object.

## Features

- **Multi-machine SSH** — save any number of hosts (`host`/`port`/`user` + **private key** or **password** or **key + passphrase**). Passwords are stored locally and never shown back in the UI.
- **Interactive terminal** — a WebSocket-based PTY shell per machine, right inside the GUI (sidebar footer 「远程终端」).
- **Two-tab workspace picker** (fills the native "Add workspace" flow):
  - **本机 / Local** — native OS folder chooser / typed path → adopted as a normal DSH local workspace.
  - **远程 / Remote** — centered modal; pick a machine → path field pre-filled with `/` and live directory autocomplete; 浏览… floating browser; confirm → creates a **real local mirror** (`~/.dsh/remote-workspaces/<host>/...`) that passes `fs.realpath`, kept synced over SFTP.
- **Bidirectional SFTP sync** — `rw_sync` (remote → mirror) and `rw_push` (mirror → remote).
- **Model tools** — `rw_info`, `rw_connect`, `rw_pick_workspace`, `rw_list_dir`, `rw_read_file`, `rw_write_file`, `rw_exec`, `rw_sync`, `rw_push`, `rw_disconnect`.
- **Connection health** — 「测试连接」 validates host/user/key/password before you save a machine.
- The active `user@host:/path` is injected into every system prompt so the agent knows its working root.
- **No official `dsh-workspace` core is modified** — everything is delivered as a normal plugin.

## Install

```bash
# <profile> must be the profile your harness is actually running:
#   `web` for `pnpm dsh web`; the desktop build uses `web-desktop`
dsh plugin --profile <profile> add git+https://github.com/yc-csu/dsh-ssh-remote.git

# or from a local checkout
dsh plugin --profile <profile> add /path/to/dsh-ssh-remote
```

Then **fully restart** the harness (not a page reload) — the plugin activates on boot.

Two things that are easy to get wrong:

- `dsh plugin add` runs `pnpm add` in the profile directory **and** registers the package in `dsh.profile.bundles`. If you install by hand with pnpm/npm, add the bundle entry yourself, otherwise the plugin will never load.
- Prefer the explicit `git+https://…` spec: the `github:user/repo` shorthand can be resolved as `ssh://git@github.com` and fail on a machine without an SSH key.

Installing into the wrong profile is the usual reason a plugin "installs fine" but never shows up.

See **[INSTALL.md](./INSTALL.md)** (中文) for profile discovery, proxy pitfalls, npm-vs-pnpm profiles, Windows prerequisites and post-install verification.

> Note: the harness Web UI intentionally binds `127.0.0.1`; this plugin connects **out** to machines you maintain — no changes to the harness core.

## Quick start

1. **Add a machine** — Settings → 远程工作区 → add host/port/user + key or password → (optional) set it current.
2. **Open a workspace** — click **Add workspace** in the sidebar / conversation:
   - **本机** → system folder chooser (or type a local path) → local workspace.
   - **远程** → choose the machine → browse to a remote directory (or type `/path`) → "设为远程工作区" ⇒ a local mirror workspace is created and adopted.
3. **Work with the agent** — treat it like any workspace:
   - `rw_list_dir(path?)` / `rw_read_file` — inspect remote files
   - `rw_write_file(path, content)` — create or overwrite a remote file directly
   - `rw_exec(command)` — run remote shell commands
   - `rw_sync` / `rw_push` — pull/push the local mirror to and from the remote
4. **Interactive terminal** — click 「远程终端」 in the sidebar footer, pick a machine, and get a live PTY shell.

## Configuration

| Key | Type | Default | Meaning |
| --- | --- | --- | --- |
| `host` | string | `''` | default SSH host (else start disconnected) |
| `port` | int | `22` | default SSH port |
| `username` | string | `''` | default SSH user |
| `password` | string | `''` | default SSH password (non-empty overrides key) |
| `privateKeyPath` | string | `''` | private key path (`~/.ssh/id_rsa` when empty) |
| `passphrase` | string | `''` | passphrase for an encrypted private key |
| `workspace` | string | `''` | default remote workspace path |
| `commandTimeoutMs` | int | 20000 | per remote command timeout |
| `connectTimeoutMs` | int | 15000 | SSH connect timeout |

## Safety

Giving the plugin a machine's credentials lets the agent run **shell commands as your user** on that host. Only add machines you trust. Passwords are saved on the local machine file; treat it as sensitive (you may lock file ACLs).

## License

MIT — see [LICENSE](./LICENSE). This project is a fork of [`dsh-remote`](https://github.com/flymysql/dsh-remote) (MIT, Copyright (c) 2026 dsh-remote contributors).
