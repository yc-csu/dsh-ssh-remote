# Changelog

## Unreleased

- Docs: new [`INSTALL.md`](./INSTALL.md) — install & troubleshooting guide (profile discovery, bundle registration, proxy pitfalls, npm-vs-pnpm profiles, Windows prerequisites, post-install verification).
- Docs: corrected the install commands in README / README.zh — `<profile>` must match the running harness (the desktop build uses `web-desktop`, not `web`), and the spec now uses explicit `git+https://` to avoid npm resolving `github:` as `ssh://`.
- Chore: `.gitignore` now excludes credential files (`machines.json`, `.env*`, `*.pem`, `*.key`, `id_rsa*`, `.npmrc`, `.credentials.yaml`).

## 0.6.0 — 2026-08-16

Fork of `dsh-remote` 0.5.4 published as `dsh-ssh-remote`:

- Renamed package / plugin id / routes to `dsh-ssh-remote` (independent bundle, coexists with `dsh-remote`).
- **New: interactive WebSocket PTY terminal** — open a live SSH shell for any saved machine from the sidebar footer 「远程终端」 (xterm.js rendered; independent of the agent exec/sftp pool).
- **New: key passphrase support** — encrypted private keys connectable via a `passphrase` field in the machine form.
- Added `ws` runtime dependency for the terminal server half.
- Docs rewritten (README / README.zh).

## 0.5.4 (upstream dsh-remote)

Multi-machine SSH registry, remote-workspace picker (local mirror + SFTP bidirectional sync), `rw_*` model tools, connection health check.
