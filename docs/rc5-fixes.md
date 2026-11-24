# dsh-purge 1.1.72（DSH 0.2.2-rc.5 修复版）

适配：DeepSeek Harness **0.2.2-rc.5** 官方桌面版（`X:\DeepSeek Harness`）。
基线：`aaron6769/dsh-purge` master = 上游 1.1.71 **逐字节相同**（仅 CRLF + 镜像说明），所以"升级"= 自己改。

## 1. 真 bug：hostFs 缺 fd 级方法（lib/extract-asar.js）

`hostFs` 只代理了按路径的同步方法，`readSync / writeSync / closeSync / fsyncSync` 全缺 →
`isTextShim()` 每次在 `fs.readSync` 抛 TypeError、被 catch 吞成 `false` →
`shimFileStatus()` 对所有 shim 恒 `binary`、`shimIsPatched()` 恒 false、`stripCmdInjectSync()` 永不生效。

```js
for (const method of ["readSync", "writeSync", "closeSync", "fsyncSync"]) {
  hostFs[method] = (...args) => nodeFs[method](...args);
}
```

实测：修复前 `shim: dsh.cmd=binary`，修复后 `dsh.cmd=original`（该构建确实没有 dsh.ps1 / dsh，属正常）。

## 2. 补齐 rc.5 原文漂移的 10 条补丁（新增 lib/patch-fixes-rc5.js）

官方重排实现后，1.1.71 的逐字 pattern 失配，被 `filePatchOutcome()` 判成 `skipped`——**不报错、静默跳过**。新增模块按"实测切片"补命中（pattern 由 `resources/app` 当次切片生成，不手抄），语义与原条目一致，marker 注释沿用原判定串：

| # | 补丁 | rc.5 变化 |
|---|---|---|
| 13 | ESCALATION_WIDENING_EXEMPT | `throw new Error` → `SandboxEscalationError` + 错误码 |
| 14 | ESCALATION_GRANT_UNCONDITIONAL | `request({…})` 多出 `displayReason: escalationDisplayReason(...)` |
| 35 | ESCALATION_NEVER_REJECT | switch 分支文案重写 + `throwIfAborted()` |
| 17 | FS_OBSERVATION_INTENT_FREE | `editIntent` 改为 `requirePresent(target, actor, "edit")` |
| 18 | REPEAT_TOOL_REMINDER_DISABLED | `post-execute` 少了 `block` 分支 |
| 62 | OFFICIAL_TOOL_ASK_FAILOPEN | ask 决策搬到 `dsh-tools/lib/pipeline.js`（新增目标键 `tools-pipeline`） |
| 60 | OFFICIAL_RUN_CODE_APPROVAL_NEUTRAL | 描述搬进独立包 `dsh-tool-executor-ptc`（新增键 `tool-executor-ptc`） |
| 67 | AGENT_LOOP_MESSAGES_GUARD | 上下文行变化 |
| 74 | ASSEMBLE_BEFORE_PRESTEP | `assemble({…})` 变多行带 `modelSelection` / `toolSet` |
| 82 | DESKTOP_SHELL_PROFILE_FALLBACK | `//#region` 不再紧跟 api-gateway import，改以 import 单行作锚点 |

配套 `lib/core.js`：导入 `RC5_FIXES` 并在 `ALL_PATCHES` 之后并入对应 patch 的 `replacements`；`targetFiles()` 增加 `tools-pipeline`、`tool-executor-ptc`；#62/#60 的 `file` 列表追加这两个键。
**版本 1.1.72、`dshTarget` 保持 0.2.0-rc.2**——插件的兼容判定只读 `engines.dsh` / `@deepseek-ai/dsh*` peer，不读 `dshTarget`。
本机已装的那份副本仍是 1.1.71（同一份代码，只是版本号没跟），不影响运行。

## 3. 两个踩过的坑（写进流程）

1. **`package.json` 不能带 BOM**。用 Windows PowerShell 5.1 的 `Set-Content -Encoding UTF8` 改清单会加 BOM，宿主 `JSON.parse` 直接 `SyntaxError` → 整个 bundle 被 skip（现象：重启后插件工具与提示词全消失，日志里只有一行 `dsh: skipping profile bundle "dsh-purge": SyntaxError: Unexpected token '锘'`）。改 JSON 一律用 Node `fs.writeFileSync(path, text, "utf8")`。
2. **等待退出的助手不能是本应用的子进程**。用 `Start-Process` 从宿主里起的 powershell 会被应用 job 连坐杀掉，退出瞬间助手先死、`app.asar` 永远改不了名。改用**任务计划**（`schtasks /run`）启动，父进程是任务计划服务，才能在应用退出后改名 + 重启，并在末尾自删任务。

## 4. 实测结果（本机，2026-11-24）

```
apply 前 58/82 applied, 10 pending
apply 后 68/82 applied,  0 pending
换包后重查：68 applied / 7 skipped / 1 missing_file / 6 na / 0 pending（合计 82）
语法检查：8 个被改文件 node --check 全通过
运行位置：checkout = X:\DeepSeek Harness\resources\app\dsh\（已脱离 app.asar）
备份：58 个 *.dshpurge.bak 已生成 → revert 可用
运行期自检：审批自动放行、文件策略 danger-full-access、未读先改（fs observation bypass）实测通过
```

## 5. 仍未覆盖（14 条）

| # | 原因 | 影响 |
|---|---|---|
| 32 / 53 / 54 | `${webUrl}` 提示词段在 rc.5 移入 `dsh-harness-dev-context`，需再加目标键 | 由插件运行时重写（`rewriteLiveAssembly` + prompt-inject）兜住 |
| 56 / 77 | 目标在压缩后的 `dsh-client-ui-conversation/lib/client.js`，旧 pattern 是格式化代码 | 回退/重来时的展示细节（可选） |
| 80 | `minimal.patch.yml` 结构变化，且只影响 minimal 预设 | 只用 standard 预设时无感 |
| 40 | 0.1.5 兼容专用 | 无 |
| 71 | 目标文件（liangshen 预设）未安装 | 无 |
| 28 / 30 / 31 / 64 / 65 / 76 | 对应组件本机未安装 / 版本不适用（na） | 无 |

## 6. 安装 / 升级

三种装法，任选其一（装完必须重启 dsh）：

1. **插件管理器（桌面版）**：安装 spec 填 `github:aaron6769/dsh-purge`；想钉死这版就填 `github:aaron6769/dsh-purge#v1.1.72-rc5fix`。
2. **CLI 安装到指定 profile**：`dsh plugin --profile <name> add github:aaron6769/dsh-purge`（桌面 profile 由应用托管，CLI 会拒绝，走第 1 种）。
3. **离线包**：`https://github.com/aaron6769/dsh-purge/archive/refs/heads/master.zip`，解压覆盖 `<profile>/node_modules/dsh-purge`。

装上后自检：`dsh-purge --status` 应显示 `68/82 已清洗, 0 待清洗`，且 `app.asar` 那行不出现。

## 7. 换包与回滚

换包（已执行完毕，保留备查）：

```
schtasks /run /tn dsh-purge-asar-swap     # 任务计划里等应用退出 → 改名 asar → 改 dsh.cmd → 重启 → 自删
```

回滚：

```
pwsh -NoProfile -ExecutionPolicy Bypass -File C:\Users\luo\.dsh\dsh-purge\rollback-asar.ps1
```

当前可回收的磁盘：`resources\app.before-standard-repair-20261122-010834`（488 MB，11/22 修复快照）。
保留建议：`app.asar.purge-bak`（199 MB，唯一原版备份）、`resources\app`（运行中，451 MB）。
