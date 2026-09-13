# dsh-ssh-remote

**DeepSeek Harness (DSH) 的 SSH 远程连接服务器插件** — 基于 [`dsh-remote`](https://github.com/flymysql/dsh-remote)（作者 [flymysql](https://github.com/flymysql)）的增强 fork。

新增能力：

- **交互式 SSH 终端（WebSocket PTY）**：在 Web GUI 侧边栏底部「远程终端」直接打开远程机器的交互 shell（xterm.js 渲染），与 agent 的 exec/sftp 连接池相互独立，互不干扰
- **密钥口令（passphrase）**：加密私钥可在设置页填写口令后正常连接
- 完整保留原版功能：多机器 SSH 注册表、远程工作区（本地 SFTP 镜像双向同步）、`rw_*` 模型工具集

## 功能

- **多机器 SSH 管理** — 保存任意多台主机（`host`/`port`/`user` + **私钥**或**密码**或**密钥+口令**），密码本地存储，UI 不回显
- **交互式终端** — 每台机器一个基于 WebSocket 的 PTY shell，直接在 GUI 里用
- **双 Tab 工作区选择器**（融入原生「添加工作区」流程）：
  - **本机** — 系统目录选择器 / 手输本地路径 → 直接作为本地工作区
  - **远程** — 居中弹窗：选机器 → 路径以 `/` 预填并实时补全目录；「浏览…」浮层预览；确认后创建**真实本地镜像**（`~/.dsh/remote-workspaces/<host>/...`，通过 `fs.realpath` 校验），由 SFTP 保持同步
- **双向 SFTP 同步** — `rw_sync`（远程 → 镜像）、`rw_push`（镜像 → 远程）
- **模型工具** — `rw_info` / `rw_connect` / `rw_pick_workspace` / `rw_list_dir` / `rw_read_file` / `rw_write_file` / `rw_exec` / `rw_sync` / `rw_push` / `rw_disconnect`
- **连接健康检查** — 设置页「测试连接」按钮，保存前先验证 host/user/key/password
- 当前 `user@host:/path` 注入每条系统提示词，agent 始终知道自己的工作根目录
- **不改动 `dsh-workspace` 核心** — 完全以普通插件形式分发

## 安装

```bash
# <profile> 必须与你正在运行的 harness 一致：
#   命令行形态（pnpm dsh web）→ web；桌面版 DSHEAC → web-desktop
dsh plugin --profile <profile> add git+https://github.com/yc-csu/dsh-ssh-remote.git

# 或本地目录
dsh plugin --profile <profile> add /path/to/dsh-ssh-remote
```

安装后需**完全重启** harness（不是刷新页面）生效，插件在启动时装载。

两个最容易踩的坑：

- `dsh plugin add` = 在 profile 目录跑 `pnpm add` **并**把包登记进 `dsh.profile.bundles`。若你手工用 pnpm/npm 装，必须自己补 `bundles`，否则插件永远不会被加载。
- spec 建议用显式 `git+https://…`。`github:user/repo` 简写可能被解析成 `ssh://git@github.com`，没配 SSH key 的机器会直接失败。

装到错误的 profile，是「安装成功但界面里什么都没有」的头号原因。

详见 **[INSTALL.md](./INSTALL.md)**：如何定位目标 profile、代理陷阱、npm/pnpm 混合 profile、Windows 前提、装后验证。

> 说明：harness Web UI 刻意只绑定 `127.0.0.1`；本插件是**主动外连**你维护的机器，不改 harness 核心。

## 快速上手

1. **添加机器** — 设置 → 远程工作区 → 填 host/port/user + 密钥或密码 →（可选）设为当前
2. **打开工作区** — 侧边栏 / 对话里点「添加工作区」：
   - **本机** → 系统目录选择器（或手输本地路径）→ 本地工作区
   - **远程** → 选机器 → 浏览到远程目录（或直接输 `/path`）→「设为远程工作区」⇒ 创建本地镜像并采纳
3. **让 agent 干活** — 像普通工作区一样使用：
   - `rw_list_dir(path?)` / `rw_read_file` — 查看远程文件
   - `rw_write_file(path, content)` — 直接新建/覆盖远程文件
   - `rw_exec(command)` — 执行远程 shell 命令
   - `rw_sync` / `rw_push` — 本地镜像与远程双向同步
4. **交互式终端** — 侧边栏底部点「远程终端」，选机器，得到实时 PTY shell

## 配置

| Key | Type | 默认 | 含义 |
| --- | --- | --- | --- |
| `host` | string | `''` | 默认 SSH 主机（空则启动时不连接） |
| `port` | int | `22` | 默认 SSH 端口 |
| `username` | string | `''` | 默认 SSH 用户 |
| `password` | string | `''` | 默认密码（非空时优先于密钥） |
| `privateKeyPath` | string | `''` | 私钥路径（空则用 `~/.ssh/id_rsa`） |
| `passphrase` | string | `''` | 加密私钥的口令 |
| `workspace` | string | `''` | 默认远程工作区路径 |
| `commandTimeoutMs` | int | 20000 | 远程命令超时 |
| `connectTimeoutMs` | int | 15000 | SSH 连接超时 |

## 安全

给插件一台机器的凭据，等于允许 agent 以你的身份在那台机器上**执行 shell 命令**。只添加你信任的机器。密码保存在本机文件里，请按敏感信息对待（可收紧文件 ACL）。

## 许可

MIT — 见 [LICENSE](./LICENSE)。本仓库是 [`dsh-remote`](https://github.com/flymysql/dsh-remote)（MIT，Copyright (c) 2026 dsh-remote contributors）的 fork。
