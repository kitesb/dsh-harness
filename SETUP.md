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
git fetch upstream        # ⚠️ 2026-09-14 实测 github.com:443 直连被 DLP 掐（Could not connect），
                          # 改走 ssh 通道一次性 fetch：
                          # git fetch ssh://git@ssh.github.com:443/deepseek-ai/DeepSeek-Harness.git master
git rebase --onto upstream/master <旧基点> master   # 本地修复重放到官方最新
pnpm install                                        # 官方依赖树变了必须重跑
pnpm run build:lib:host                             # 见第四节，rebase 后必做
git push -u origin master                           # 需 danger-full-access（见第五节）
```

- **不要 squash / 删历史**：fork 和官方共享历史是你能无缝同步的根基，压掉就断了。
- rebase 冲突时**先核对官方是否已自研同类修复**——本次 llm-deepseek 就撞上：官方已用 `acceptIdentity()` 修了 delta 层，只保留官方缺的层（closeBlock 降级 + serialize 过滤），并把 `CallId` 改名成官方 `ToolCallId`。
- **rebase 前先 `git show FETCH_HEAD:<file>` 核对官方是否已修本地持有的修复**（2026-09-14 例：官方仍缺 closeBlock 降级/serialize 过滤、sandbox dev 臂仍裸 `tsx/esm`，两个修复都保留重放；唯一冲突在 profile-boot.ts——官方改了 `prepareProfile` 签名，解法=官方新签名 + 保留 compat-check 块）。
- rebase 需干净工作树：本地未提交改动用 `git -c rebase.autoStash=true` 自动暂存/回贴，**别手动 stash 别人的本地改动**。

## 四、rebase 后必做：build:lib:host + 僵尸包清理

- `pnpm install` 之后**必须** `pnpm run build:lib:host`，否则新包的 `lib/` 没产物 → `workspace-write` 下 shell 工具崩（`Cannot find module '...win32-process/lib/index.js'`）。
- **⚠️ build:lib:host 只够宿主面，web GUI 还要全量 build（2026-09-14 事故修订）**：rebase 到 0.1.5-rc.2 后只跑 `build:lib:host`，启动 web 报 "client bundles not found"——`packages/client/ui-open-in-app`、`packages/api/workspace-files`、`packages/client/resources`、`ui-sidebar-right/documentpreview/files` 等 6 个 `lib\client.js` 只有 `pnpm run build`（全量）才产出。**rebase 后标准序：`pnpm install` → `pnpm run build`（全量）→ 冒烟实测一次真实启动（`pnpm dsh --profile web` 后台起+杀），不是只 `--dump-config`**。也别单独跑 `pnpm run clean`（会把 client bundle 清掉，回到同一坑）。
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

## 六、本地修复状态（2026-09-14 已 rebase 到官方 0.1.5-rc.2，哈希别用旧的）

> 当前基点：upstream master `c291e7961a`（dsh v0.1.5-rc.2，2026-09-10 发布）。
> 2026-09-14 rebase：1593 commit 大跨度重放，8/8 成功；唯一冲突 profile-boot.ts（`prepareProfile` 签名变化，手工合入官方新签名 + 保留 compat-check 块）。
> rebase 后验证：serialize 57 + translate 42 全绿；sandbox-windows-acl runner+provider-chain 16 全绿；`pnpm dsh --profile web --dump-config` exit 0；无僵尸包；`packages/sandbox/sandbox-local` 测试套已被官方移出 Windows 车道（vitest.config.ts `windowsUnsupportedPackages`），Windows 侧验证靠 runner/provider-chain spec + 运行时实测。

> **`package.json` 常驻本地改动 = 正常，别推**：`pnpm install`（本机 corepack/pnpm 11.23.0）会把
> 根 `package.json` 的 `"packageManager": "pnpm@11.7.0"` 自动回写成 `pnpm@11.23.0`（本机实际版本）。
> 这是**工具链副产物，不是功能修复**，且本机没网装不了 11.7.0——**约定：永远保持本地未提交（`M`），
> 不要 commit/push 它**，否则等于给 fork 塞一个无意义的包管理器版本升级。每次 `git status` 看到
> `M package.json` 属预期。

| commit | 内容 | 备注 |
|---|---|---|
| `cae3803094` | `fix(llm-deepseek)`：guard empty id/name | 官方已修 delta 层（`acceptIdentity`）；本 commit 只剩官方缺的 closeBlock 降级 + serialize 过滤（0.1.5-rc.2 仍缺，2026-09-14 核实） |
| `15a73c9d91` | `fix(sandbox)`：windows-acl runner `--import` 传 file:// URL | 官方未修（0.1.5-rc.2 dev 臂仍裸 `tsx/esm`，2026-09-14 核实）；build 后生产臂 runner.js 在位 |
| `196db1d1ee` | `feat(cli)`：composeProfile 前置 `plugin-compat-check --interactive` | 启动时自动体检→坏插件用户选择→写 managed block，根治"装完重启→崩→修→崩"死循环。2026-09-14 rebase 冲突已手工合入（`prepareProfile` 新签名）。依赖 `dsh-framework` 仓的 compat-check 脚本（路径见框架仓 SETUP §二·五·7）

> 本 SETUP.md 是 fork 私有文档（未 PR 上游），rebase 官方时若与上游文件冲突可安全丢弃本文件的冲突侧。

> **环境类踩坑速查**：本仓相关的操作问题（推送通道、pnpm 对账、compat-check 体检、GitHub 镜像安装等）的**修法**统一收录在框架仓 `kitesb/dsh-framework/LESSONS.md`（换机器后若某步卡住，先 grep 现象关键词定位）。

## 七、fork 的 GitHub Actions：默认禁用是有理由的，别点"启用"（2026-09-14 实测）

**现象**：push 到 origin/master 后 Actions 页一片红叉——`E2E (real DeepSeek API)` 报
`DEEPSEEK_API_KEY is empty ... Configure the repo secret DEEPSEEK_API_KEY_EXTERNAL`；
`CI master` 的 python-runtime、`Sandbox` 的 macOS leg 同理。

**根因**：官方 22 个 workflow 是给官方仓设计的：
- `e2e.yml` 在可信事件（push master）上**故意 fail-loud**：secret 缺失会让 e2e 套件自跳过 =
  "假绿"（false green），官方宁可红。而 **GitHub fork 不继承 secrets**——本 fork 永远不会有这个 key。
- `ci-master.yml` 的 python-runtime job 消费同一 secret（`build-exe-for-python-sdk.yml` 预检同款
  fail-loud），开了必红。

**为什么 09-14 之前从没见过**：GitHub 对 fork 仓**默认禁用 Actions**（Actions 页有
"I understand my workflows, go ahead and enable them" 横幅）。2026-09-14 横幅被点掉后，
当天的 push 才开始触发 CI（API 实证：fork 的 Actions 历史里 09-14 之前零 run，
09-03 建仓、09-07/09-11 的 push 都没跑过）。

**处置（推荐 A）**：
- **A. 关掉**：仓库 Settings → Actions → General → Actions permissions → **Disable actions**。
  个人镜像仓不需要云端 CI——本地已有验证门（lefthook pre-push typecheck + 目标 vitest 套件 +
  rebase 后真启动冒烟，见 §四）。想跑随时再开。
  **✅ 2026-09-14 已执行并实证**：关闭后推空提交 `ea6552c556` 探针，GitHub runs 列表**零新增**
  （对照：关闭前 05:52 UTC 的 push `031c42daf3` 仍触发了全套 5 个 workflow）。注意：关闭**不会**
  追杀已在队列/在跑的旧 run（031c42d 的 CI master 长时间 queued、Sandbox in_progress 属预期，
  跑完或队列超时自灭；碍眼可去 Actions 页手动 Cancel）。另：workflow state API 仍显示
  "active" 属正常——仓库级总开关不写进单个 workflow 的 state 字段，匿名也读不了 permissions API，
  **判"关没关"唯一可靠证据就是探针 push 后 runs 零新增**。
- B. 逐个禁用吵闹的 workflow（Actions 页选 workflow → ⋯ → Disable）——22 个里挑红叉，打地鼠。
- ~~C. 配 secret~~：**别**——那会让每次 push 真烧 DeepSeek API 额度跑 e2e（官方 key 2026-09-04
  起额度已失效），且 macOS darwin leg 该红还是红（Sandbox 的 darwin parity 全量单测在本 fork 红，
  上游 master 8-13 后没跑过该 workflow、无法对比判断是否 0.1.5 既有问题；job 日志需鉴权未读到
  具体测试名，属未定项，本地 Windows 门全绿）。
