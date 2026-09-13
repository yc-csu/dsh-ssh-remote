# 安装与排错 · dsh-ssh-remote

DSH 插件的安装实战笔记。文中大部分结论对所有 `dsh-*` 插件都适用（同样适用于上游 `dsh-remote`）。

- English README: [README.md](./README.md)
- 中文 README: [README.zh.md](./README.zh.md)

## TL;DR

```bash
# 1) 先确认「正在运行」的 profile（见 §1，装错位置是头号坑）
# 2) 用目标 harness 自带的 dsh CLI 安装
dsh plugin --profile <profile> add git+https://github.com/yc-csu/dsh-ssh-remote.git
# 3) 完全重启 harness（不是刷新页面）
```

---

## 1. 先确认目标 profile

`--profile <name>` 必须和**正在运行的** harness 一致。装错 profile 的典型症状是「安装全部成功，但界面里什么都没有」。

| 部署形态 | profile 位置 |
| --- | --- |
| 桌面版 DSHEAC | `%APPDATA%\com.deepseek.dsh.desktop.aio\releases\<版本>\dsh-home\profiles\<name>` |
| 命令行 / 自部署 | `$DSH_HOME/profiles/<name>`（`DSH_HOME` 未设时默认 `~/.dsh`） |

确认办法：

```powershell
# 正在跑的 dsh 进程用的是哪个 --profile / --dsh-home
powershell -NoProfile -Command "Get-CimInstance Win32_Process -Filter \"Name='node.exe'\" | Select ProcessId,CommandLine | Format-List"
```

harness 日志里也能找到 `[plugin-update] mounted ... profile=<路径>` 这样的行。

> ⚠️ 只对命令行形态（`pnpm dsh web`）成立：`--profile web`。桌面版 DSHEAC 用的是 `web-desktop`，装进 `web` 不会生效。

## 2. `dsh` 可能不在 PATH，且必须指定 DSH_HOME

桌面版把 CLI 打包在应用内部，要显式给出 dsh-home，否则它会去改默认的 `~/.dsh`：

```powershell
$env:DSH_HOME = "$env:APPDATA\com.deepseek.dsh.desktop.aio\releases\<版本>\dsh-home"
node "D:\...\resources\app\node_modules\@deepseek-ai\dsh\lib\bin.js" `
  plugin --profile web-desktop add git+https://github.com/yc-csu/dsh-ssh-remote.git
```

`dsh plugin add` 实际做两件事：

1. 在 profile 目录里执行 `pnpm add <spec>`；
2. 把依赖中声明了 `dsh.bundle` 的包登记进 `package.json` 的 `dsh.profile.bundles`。

**所以手工 `pnpm add` 之后必须自己补 `bundles`**，否则插件永远不会被加载。

## 3. 目标 profile 的包管理器可能不是 pnpm

先看痕迹再决定用什么装：

- 有 `node_modules/.package-lock.json` → 这个 profile 是 **npm** 装的
- 有 `pnpm-lock.yaml` / `node_modules/.modules.yaml` → pnpm

如果 profile 里有**公共 registry 上不存在的依赖**（例如 `@local/*`）或带着大量 `overrides`，那么 `pnpm add` / `npm install` 会因为**重解析整棵依赖树**而失败。

判断方法：对每个直接依赖跑一遍 `pnpm view <name> version`，全部可达才适合直接装。

不能直接装时，改用「隔离手术式」：

```bash
# a) 临时目录里建一个干净工程，只装插件
mkdir stage && cd stage
printf '{"name":"stage","private":true,"version":"1.0.0"}' > package.json
printf 'packages:\n  - .\n\nnodeLinker: hoisted\nallowBuilds:\n  ssh2: true\n  cpu-features: true\n' > pnpm-workspace.yaml
pnpm add git+https://github.com/yc-csu/dsh-ssh-remote.git

# b) 只把目标 profile 里缺失的包复制过去（不要覆盖已存在的包）
# c) 手工编辑目标 profile 的 package.json：加 dependencies + 加 dsh.profile.bundles
```

## 4. 网络：代理与 GitHub 协议

**先查死代理：**

```bash
cat ~/.npmrc                       # proxy= / https-proxy=（注意可能是 CRLF 换行）
git config --global --get http.proxy
```

失效代理会让 npm / pnpm / git **全量 `ECONNREFUSED`**，而 `curl` 直连却是通的 —— 极易误判成「断网了」。另外 `pnpm config get proxy` 有时返回 `undefined` 但 pnpm 实际仍在走代理，所以用覆盖 userconfig 的方式最可靠（只影响本次调用，不动用户配置）：

```bash
printf '' > /tmp/empty-npmrc
npm_config_userconfig=/tmp/empty-npmrc pnpm add <spec>
GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0=http.proxy GIT_CONFIG_VALUE_0= git ls-remote <repo>
```

**spec 用显式 https：**

```bash
git+https://github.com/yc-csu/dsh-ssh-remote.git   # ✅ 推荐
github:yc-csu/dsh-ssh-remote                       # ⚠️ npm 可能解析成 ssh://git@github.com，没配 SSH key 直接失败
```

到不了 GitHub 时用 tarball（`pnpm pack` 后在目标 profile 里装 tgz）。**不要用 `file:` / `link:` 指向源码目录**：那是软链，Node 按 realpath 解析，插件自己 `import 'ssh2'` 会找不到依赖。若确实要用本地目录，请先在源码目录里跑一次 `pnpm install`，或用 tarball。

## 5. Windows 前提

- **Git for Windows 必须装在默认路径 `C:\Program Files\Git`**。harness 的 shell 路径硬编码为 `C:\Program Files\Git\bin\bash.exe`，装到 `D:\Git` 之类自定义路径不生效；装完还需**完整重启 harness**，否则一直报 `command "C:\Program Files\Git\bin\bash.exe" is not an executable file`。
- **ssh2 的原生加密绑定编译失败是正常的**（没有 VS Build Tools）：会回退纯 JS 实现，不影响功能，不要当成安装失败。
- pnpm 10+ 默认拦截依赖构建脚本，需要放行：

```yaml
# profile 的 pnpm-workspace.yaml
allowBuilds:
  ssh2: true
  cpu-features: true
```

## 6. 依赖与版本

- profile 至少要能解析到：`ssh2`（含 `asn1` / `bcrypt-pbkdf` / `tweetnacl`）、`ws`、`@deepseek-ai/schemastery`，以及 host 侧 peer：`@deepseek-ai/dsh-tools`、`dsh-system-prompt`、`dsh-host-webserver`、`@deepseek-ai/cordis`。其中 **`ssh2` 通常需要新装**，其余多由 harness 自带。
- 本插件面向 dsh `^0.1.0-rc.6`，已在 `0.1.5-rc.2` 上验证可用。换到更新的大版本时，注意内部 API（`ctx.tools.register`、webServer 路由、设置 schema）是否变动。
- 新版 DSH 已不再提供 `@deepseek-ai/dsh-client-runtime`（现为 `dsh-client-modules`），但插件沿用旧注入名**仍然可用**，不需要改。

## 7. 装完必须「完全重启」+ 验证

插件在 harness **启动时**装载，所以要**退出整个应用再启动**，不是刷新网页。

想免重启先确认，可做只读预检：

```bash
# 1) 确认插件已进入组合后的加载栈
DSH_HOME=<dsh-home> node <app>/node_modules/@deepseek-ai/dsh/lib/bin.js \
  --profile <name> --dump-config | grep -n dsh-ssh-remote

# 2) 确认宿主半边能真正加载（依赖都解析得到）
node --input-type=module -e "await import('file:///<profile>/node_modules/dsh-ssh-remote/lib/index.js')"
```

> `lib/client.js` 直接 import 会报 `window is not defined` —— 那是浏览器侧模块，属正常现象。

重启后的界面验证：

- 侧边栏底部出现「**远程终端**」，选机器可直接进交互 shell
- 设置 → **远程工作区** 能看到机器列表
- 让 agent 调用 `rw_list_dir` / `rw_exec`

## 8. 不要和上游 `dsh-remote` 同时启用

两者注册**完全相同的 10 个 `rw_*` 工具名**，同时启用会有重名冲突风险。`dsh-ssh-remote` 是 `dsh-remote` 0.5.4 的超集（多了交互式终端与密钥口令），二选一即可。

## 9. 安全

- 插件把机器凭据存在本机文件（`~/.dsh/remote-workspaces/machines.json`，或 dsh-home 下的同名路径），其中包含**明文密码与密钥口令** —— 不要提交到仓库、不要放进同步盘，必要时收紧文件 ACL。
- 给插件一台机器的凭据，等于允许 agent 以你的身份在那台机器上**执行 shell 命令**：只添加你信任的机器；私钥建议使用带 passphrase 的。
- 不要把私有 registry 的 token 写进会被提交的 `.npmrc`；本仓库 `.gitignore` 已排除常见凭据文件。

## 10. 卸载 / 回滚

```bash
dsh plugin --profile <name> remove dsh-ssh-remote   # 会一并把它从 bundles 里摘掉
```

手工安装的情况下：删除 `node_modules/dsh-ssh-remote`，并把它从 `package.json` 的 `dependencies` 与 `dsh.profile.bundles` 里移除，然后重启 harness。
