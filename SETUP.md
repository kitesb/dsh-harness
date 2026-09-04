# DSH 源码仓 SETUP · 编译 / 测试 / 同步 / 推送环境

> 本文件属于 **DSH 源码 fork（`kitesb/dsh-harness`，上游 `deepseek-ai/DeepSeek-Harness`）**，
> 只记录"编译、测试、跟上官方、推送"这套**源码环境**。协作空间/人格/记忆体那些在
> `kitesb/dsh-framework` 的 SETUP.md（两个环境的记录已分离，别混）。

## 一、本机已装 / 换机器必做

| 依赖 | 为什么 | 状态 |
|---|---|---|
| Node ≥ 22.18（本机 24.20.0） | DSH 运行必需 | ✅ |
| **PowerShell 7**（`C:\Program Files\PowerShell\7\pwsh.exe`） | 源码测试 `runner.spec.ts` 硬编码调 `pwsh`（6 用例）；只有 5.1 时沙箱 runner 报 `CreateProcessAsUserW Win32 2` 全红 | ✅ 7.4.6（MSI `ADD_PATH=1`，**Machine PATH 已永久写入**） |
| **Windows 开发者模式** | symlink 测试（`executor.spec.ts` 建 link fixture）非管理员 EPERM | ✅（设置→隐私和安全性→开发者选项） |

## 二、Git 布局（fork 与官方分离）

```
origin    https://github.com/kitesb/dsh-harness.git          (fetch)
origin    ssh://git@ssh.github.com:443/kitesb/dsh-harness.git (push)
upstream  https://github.com/deepseek-ai/DeepSeek-Harness.git (fetch)
upstream  DISABLED-official-repo                              (push ← 防误推官方)
```

新机器重建：
```powershell
git remote rename origin upstream      # 若 clone 自官方
git remote add origin https://github.com/kitesb/dsh-harness.git
git remote set-url --push origin ssh://git@ssh.github.com:443/kitesb/dsh-harness.git
git config remote.upstream.pushurl DISABLED-official-repo
```

## 三、跟上官方最新（rebase 工作流）

```powershell
git fetch upstream
git rebase --onto upstream/master <旧基点> master   # 本地修复重放到官方最新
pnpm install                                        # 官方依赖树变了必须重跑
pnpm run build:lib:host                             # 见第四节，rebase 后必做
git push -u origin master                           # 需 danger-full-access（见第五节）
```

- **不要 squash / 删历史**：fork 和官方共享历史是你能无缝同步的根基，压掉就断了。
- rebase 冲突时**先核对官方是否已自研同类修复**——本次 llm-deepseek 就撞上：官方已用 `acceptIdentity()` 修了 delta 层，只保留官方缺的层（closeBlock 降级 + serialize 过滤），并把 `CallId` 改名成官方 `ToolCallId`。
- rebase 需干净工作树：本地未提交改动用 `git -c rebase.autoStash=true` 自动暂存/回贴，**别手动 stash 别人的本地改动**。

## 四、rebase 后必做：build:lib:host + 僵尸包清理

- `pnpm install` 之后**必须** `pnpm run build:lib:host`，否则新包的 `lib/` 没产物 → `workspace-write` 下 shell 工具崩（`Cannot find module '...win32-process/lib/index.js'`）。
- **僵尸包陷阱**：上游删除/迁移包时，git 只删 `src/`，被 `.gitignore` 的 `lib/` 原地残留 → tsdown 扫 workspace 捡到旧 `lib/types` 报 `MISSING_EXPORT: "... 不是由 @deepseek-ai/xxx 导出"`。
  识别与清理：
  ```powershell
  # 目录有 lib/ 但 git 无任何跟踪文件（ls-files=0）→ 纯残留，整个删
  git ls-files packages/<组>/<包>   # 计数 0 即僵尸
  Remove-Item packages/<组>/<包> -Recurse -Force
  pnpm run build:lib:host
  ```
  本次（2026-09-02 rebase 到 0.1.2-alpha.5）清过 9 个：`client/runtime`、`code-runtime-python`、`examples/acp-demo`、`examples/agent-spine-demo`、`examples/jsonrpc-demo`、`host/apiproxy`、`session-persistence-sqlite`、`tool-subagent-report`、`test-support/acp-snapshot`。
- **插件契约体检（rebase 后自动触发）**：官方删 API（如 0.1.2-alpha.1 删 dsh-settings 具名导出）会让存量社区插件"启用即崩宿主"。`.git/dsh-hooks/post-rewrite` 会在每次 rebase/amend 完成后自动跑 `plugin-compat-check`（本机安装，见 `kitesb/dsh-framework/SETUP.md` §二·五·3 的安装/重建指引），结果落在 `~/.dsh/plugin-compat-check-last.log`。**rebase 后先看它一眼**：报了"启用即崩宿主 N 个"就先别启用/更新那些插件，等作者适配（或按框架仓 SETUP 用 `--fix` 预写 disable）。该 hook 退出码恒 0、不阻断任何 git 操作，纯告警。

## 五、推送通道（实测约束，别浪费时间调 HTTPS 参数）

`workspace-write`（受限模式）下 GitHub **两条路都推不动**：

| 通道 | 现象 | 根因 |
|---|---|---|
| HTTPS | `RPC failed; curl 55 Send failure: Bad access`（下载正常、上传断） | 公司 DLP/NAC（进程 `OnacAgent`）掐上传流；SpeedCat 代理 `127.0.0.1:7892` 是浏览器级不服务 CLI；`http.version=HTTP/1.1`、`postBuffer` 调参无效 |
| SSH | `ssh.exe: couldn't create signal pipe, Win32 error 5` | 沙箱禁命名管道，SSH 客户端起不来 |

**唯一可行**：会话升到 `danger-full-access`（绕过沙箱）→ SSH over 443 push（`~/.ssh/id_rsa` 已绑 GitHub，`ssh -T git@ssh.github.com` 验证过）。推完降回。协作空间仓 `kitesb/dsh-framework` 的推送也是同一套。

## 六、本地修复状态（已 rebase，哈希别用旧的）

> **`package.json` 常驻本地改动 = 正常，别推**：`pnpm install`（本机 corepack/pnpm 11.23.0）会把
> 根 `package.json` 的 `"packageManager": "pnpm@11.7.0"` 自动回写成 `pnpm@11.23.0`（本机实际版本）。
> 这是**工具链副产物，不是功能修复**，且本机没网装不了 11.7.0——**约定：永远保持本地未提交（`M`），
> 不要 commit/push 它**，否则等于给 fork 塞一个无意义的包管理器版本升级。每次 `git status` 看到
> `M package.json` 属预期。

| commit | 内容 | 备注 |
|---|---|---|
| `cbdaf6ea15` | `fix(llm-deepseek)`：guard empty id/name | 官方自修 delta 层；本 commit 只剩官方缺的 closeBlock 降级 + serialize 过滤 |
| `41637ef92c` | `fix(sandbox)`：windows-acl runner `--import` 传 file:// URL | 官方未修（0.1.2-alpha.5 仍裸 `tsx/esm`）；两臂实测 |
| 未提交 | `profile-boot.ts` composeProfile() 前置 `plugin-compat-check --interactive` | 启动时自动体检→坏插件用户选择→写 managed block，根治"装完重启→崩→修→崩"死循环。上游 rebase 时此 diff 可能冲突，保留 fork 侧。依赖 `dsh-framework` 仓的 compat-check 脚本（路径见框架仓 SETUP §二·五·7）

> 本 SETUP.md 是 fork 私有文档（未 PR 上游），rebase 官方时若与上游文件冲突可安全丢弃本文件的冲突侧。
