# PR #313 审查：免费额度恢复 hint、确认门槛与 `--config` 钉定

> 日期：2026-11-11 · 审查对象：`modelstudioai/cli` PR #313（head `3b3cb92b`，4 个提交，基于 main `fa8330c2`）· 状态：讨论稿

## 结论与范围

结论：PR 可以合并，没有阻塞性 bug。合并前建议处理 P1、P2、P3 三项，其中 P3 是历史遗留问题，但被本 PR 放大了。

| # | 级别 | 问题 | 来源 | 建议 |
| --- | --- | --- | --- | --- |
| P1 | 需要处理（中） | API Key 回退到 `default` 时，恢复 hint 仍钉在命名 profile 上，可能关错账号的自动停止 | 本 PR 引入 | 回退发生时把 hint 的 profile 改为 `default` |
| P2 | 需要处理（中） | `usage freetier` 两个方向都要 `--yes`，是未声明的破坏性变更 | 本 PR 引入 | 在 CHANGELOG 写明；或改用 `prepare` 只拦 `--off` |
| P3 | 需要处理（中） | `--off` 停用失败或超时，stdout 仍输出 “Disabled”，退出码 0 | **历史遗留**，本 PR 放大 | 成功文案改为按结果判断，失败返回非 0 |
| S1 | 建议 | `mapApiError` 用 `type` 覆盖 `code`，识别依赖英文 message 正则 | 历史遗留 | 保留原始 `code` |
| S2 | 建议 | `chunk-delete.ts` 没走 `resolveRecoveryConfigName` | 本 PR 引入 | 统一走同一个函数 |
| S3 | 建议 | hint 去重靠 `includes("usage freetier")` | 本 PR 引入 | 改用结构化标记 |
| S4 | 建议 | `supportsFreeTierRecovery` 布尔值穿透通用 runtime | 本 PR 引入 | 改传 `hasCommand(path)` |
| S5 | 建议 | hint 里是 `<model>` 占位符 | 本 PR 引入 | 可选：带上实际模型名 |

PR 改动概览（11 个文件，+465 / −50）：

- 服务端返回 `AllocationQuota.FreeTierOnly` 时，`errorHint` 追加恢复指引（`usage free` 查状态、`usage freetier --off` 停用），hint 不带 `--yes`。
- `usage freetier` 声明 `risk: { level: "high", reason: "billing" }`，不带 `--yes` 时退出码 7。
- 所有恢复命令经 `recoveryCommand` 渲染，命名 profile 下追加 `--config <name>`；显式 `--config default` 也保留。
- 能力门控 `supportsFreeTierRecovery`：命令表里没有 `usage free` / `usage freetier` 的产品不渲染该 hint。

与上一轮审查的关系：[freetier-autostop-guard-review.md](freetier-autostop-guard-review.md) 审的是同一分支的第一个提交 `30f32bbe`。那一轮的 R1（hint 带 `--yes`）和 R3（不去重）已在后续提交修复；R2 和 S1 仍在，对应本文的 P2 和 S1。P1、P3 是本轮新发现。

审查方法：

- 只做了静态审查：完整阅读 diff 及 `error-handler.ts`、`middleware.ts`、`create-cli.ts`、`config/loader.ts`、`usage/freetier.ts`、`errors/api.ts` 等上下游代码，并用 `git blame` / `git show` 追溯 P3 的历史。
- **没有运行测试**：审查环境中安装依赖被权限策略拒绝。
- 没有读到 PR 描述、已有评论和 CI 状态（仓库未挂到审查会话）。
- 没有修改 `modelstudioai/cli` 的任何内容；下文代码片段是建议写法，未经编译验证。

## 需要处理

### P1 API Key 回退场景下，hint 钉到错误的 profile

- 位置：`packages/runtime/src/middleware.ts:206`（`selectApiKeyResolutionSources`）、`packages/runtime/src/create-cli.ts:349`（`resolveRecoveryConfigName`）、`packages/runtime/src/error-handler.ts:143`。
- 背景：命名 profile 的 `api_key_capabilities` 不包含当前命令时，`authStage` 会把 API Key 的来源回退到 `default`，并打一条 notice。控制台凭证不回退，仍用命名 profile。
- 现状：`recoveryConfigName` 在 `dispatch` 之前由 `buildSources(flags)` 算出，之后不再变化。回退发生后，hint 仍然拼 `--config <命名 profile>`。
- 后果分两种：
  - **免费额度**：耗尽的是 `default` 那把 key 所属账号的额度，hint 却是 `bl usage freetier --off --model <model> --config prod`。这条命令用 `prod` 的控制台账号执行。如果两个 profile 属于不同账号，用户会关掉另一个账号的自动停止并开始按量计费，而原来的报错不会消失。
  - **`missing-api-key`**：hint 是 `bl auth login --config prod`。登录到 `prod` 后，下次执行同一命令仍会回退到 `default`，问题没解决。
- 触发条件较窄（需要配置了 `api_key_capabilities` 的命名 profile），但后果是计费，所以列为需要处理。

解决方案：回退发生时，把恢复 hint 使用的 profile 改成 `default`。reporter 在 `create-cli.ts` 里创建，`authStage` 拿不到它，需要加一个回调。

1. `OutputReporter` 增加一个 setter（`packages/runtime/src/output/reporter.ts`）：

   ```ts
   export interface OutputReporter {
     // ...
     /** 覆盖恢复 hint 使用的 profile；API Key 回退到 default 时由 authStage 调用。 */
     setRecoveryConfigName(name: string | undefined): void;
   }

   export function createOutputReporter(options: OutputReporterOptions): OutputReporter {
     let recoveryConfigName = options.configName;
     return {
       // ...
       setRecoveryConfigName(name) {
         recoveryConfigName = name;
       },
       error(error) {
         // 原来读 options.configName 的两处改读 recoveryConfigName
       },
     };
   }
   ```

2. `RunContext` 增加一个可选回调，`dispatch` 组 ctx 时接上（`middleware.ts`、`create-cli.ts`）：

   ```ts
   // middleware.ts
   export interface RunContext {
     // ...
     /** 运行期改写恢复 hint 的 profile。 */
     setRecoveryConfigName?: (name: string | undefined) => void;
   }

   // create-cli.ts，dispatch 的 case "run" 内
   const ctx: RunContext = {
     // ...
     setRecoveryConfigName: (name) => reporter.setRecoveryConfigName(name),
   };
   ```

3. `authStage` 在回退时调用（`middleware.ts:208`）：

   ```ts
   if (selection.fallbackFrom) {
     writeApiKeyFallbackNotice(ctx, selection.fallbackFrom);
     // API Key 实际来自 default；恢复命令必须指向 default，而不是被回退的 profile。
     ctx.setRecoveryConfigName?.("default");
   }
   ```

   同一个 `authStage` 开头传给 `Client` 的 `consoleRecoveryHint` 不用改：控制台凭证没有回退，仍应指向命名 profile。

4. 补测试（`packages/runtime/tests/error-handler.test.ts` 或 middleware 测试）：
   - profile `prod` 的 `api_key_capabilities` 不含当前命令，`default` 无 API Key → hint 含 `auth login --config default`，不含 `--config prod`。
   - 同样配置下返回 FreeTierOnly → hint 含 `usage freetier --off --model <model> --config default`。

- 备选（改动更小）：回退时调用 `ctx.setRecoveryConfigName?.(undefined)`，hint 不带 `--config`。缺点是激活 profile 为 `prod` 时，不带 `--config` 的命令仍会落到 `prod`，问题只解决一半，所以不推荐。
- 已知局限：API Key 来自 `DASHSCOPE_API_KEY` 环境变量或 `--api-key` 时，key 所属账号与任何 profile 都无关，hint 无法保证指向正确账号。这种情况无法在 CLI 内解决，可以在 hint 里加一句“请确认控制台登录账号与该 API Key 属于同一账号”。

### P2 `usage freetier` 两个方向都要 `--yes`，是未声明的破坏性变更

- 位置：`packages/commands/src/commands/usage/freetier.ts:24`。
- 现状：命令声明了静态 `risk`，`confirmationStage` 对整条命令生效。不带 `--yes` 时，启用和停用都返回退出码 7。
- 问题：
  - 现有脚本或 Agent 调用 `bl usage freetier --model x`（启用）会从成功变成退出码 7。
  - “启用自动停止”是保护方向，不产生费用。`docs/agents/command-add-remove.md:79` 对 `risk.reason: "billing"` 的定义是“创建持续计费资源”，只有 `--off` 符合。
  - PR 没有改 CHANGELOG。

解决方案：二选一。

- **方案 A（改动小，推荐先做）**：保留双向门槛，把破坏性变更写清楚。
  - `CHANGELOG.md` / `CHANGELOG.zh.md` 增加一条 Breaking：`bl usage freetier` 现在要求 `--yes`，否则退出码 7 / `requires_confirmation`；`--dry-run` 不受影响。
  - PR 描述里说明“启用方向也要确认”是有意为之，以及理由（启用后额度用尽时调用会失败，属于可用性风险）。
- **方案 B（更符合规范）**：只对 `--off` 要求确认。静态 `risk` 做不到按 flag 区分，需要改用 `prepare`：
  - 命令声明 `prepare`，返回的计划里 `risk` 在 `--off` 时为 `high`，否则为 `null`；静态 `risk` 保留最大风险供 help 和 `--yes` 注入使用（见 `command-add-remove.md:77-78`）。
  - 代价：`prepare` 路径下 `--dry-run` 需要凭证（规范原文“dry-run 仍需要凭证”），会改变本 PR e2e 里“`--dry-run` 零网络”的断言；`exampleArgs`、notes、reference 文档也要按方向区分。
  - 如果不想在本 PR 承担这个代价，先做方案 A，方案 B 另开 PR。

### P3 `--off` 停用失败或超时，仍输出成功文案（历史遗留）

- 位置：`packages/commands/src/commands/usage/freetier.ts:164-165`。

  ```ts
  // Keep the existing disable response policy; the JSON report retains the raw response.
  if (off || (result && failureModels.length === 0)) {
    textLines.push(/* Disabled auto-stop for "<name>". */);
  ```

- 现状：`--off` 时条件恒为真。停用超时（`result` 为空）或服务端返回 `failureModels` 时：
  - stderr 输出 `FREETIER_OPERATION_TIMEOUT` / `FREETIER_OPERATION_FAILED` warning；
  - stdout 仍输出 `Disabled auto-stop for "xxx".`；
  - 退出码 0。
- 启用方向失败时不输出成功文案，但退出码同样是 0。

**归属结论：不是本 PR 引入的。**

| 证据 | 内容 |
| --- | --- |
| 本 PR 对该文件的改动 | 只有 `risk`、`exampleArgs`、`notes` 三处元数据，`run()` 一行未改 |
| `git blame` 第 164–165 行 | `f2c1aa03`（2026-10-29，`refactor(commands): unify usage and Token Plan output`） |
| `af95a9ec`（2026-08-17）之前的版本 | 停用分支是 `await pollFreeTierBatch(...)` 后直接 `process.stdout.write("Disabled auto-stop ...")`，不看返回值 |
| `docs/agents/output-contract.md:85` | 明文写“失败与超时用 warning 提示，保留原有退出策略” |

也就是说，这个行为从停用功能最早的版本就存在。10 月的重构加了 warning，但用注释刻意保留了成功文案。

**为什么仍建议随本 PR 处理**：本 PR 新增的 FreeTierOnly hint 主动引导用户和 Agent 执行 `usage freetier --off`，并把它标成计费类高风险操作。这条原本少有人走的路径变成了推荐路径。停用失败时，Agent 读到退出码 0 和 “Disabled”，会认为已恢复并重试模型调用，然后继续失败，且没有任何信号告诉它停用没生效。

解决方案：

1. 成功文案按结果判断，两个方向用同一条件（`freetier.ts:165`）：

   ```ts
   const succeeded = Boolean(result) && failureModels.length === 0;
   if (succeeded) {
     textLines.push(/* 原有 Disabled / Enabled 文案 */);
   } else {
     failed.push(name); // 新增：const failed: string[] = [];
   }
   ```

   同时删掉那行 “Keep the existing disable response policy” 注释。

2. 有失败时返回非 0 退出码。在循环结束、`ctx.output.result(...)` 之后抛错，保证部分成功的结果仍然输出：

   ```ts
   if (failed.length > 0) {
     throw new BailianError(
       ctx.localize(
         off
           ? {
               "en-US": `Failed to disable auto-stop for ${failed.length} model(s): ${failed.join(", ")}.`,
               "zh-CN": `${failed.length} 个模型停用自动停止失败：${failed.join("、")}。`,
             }
           : {
               "en-US": `Failed to enable auto-stop for ${failed.length} model(s): ${failed.join(", ")}.`,
               "zh-CN": `${failed.length} 个模型启用自动停止失败：${failed.join("、")}。`,
             },
       ),
       ExitCode.GENERAL,
     );
   }
   ```

   这一步改变了 `output-contract.md` 写明的退出策略，需要维护者确认。如果不接受改退出码，至少做第 1 步，并在停用的 JSON 结果里加 `failed: [model]` 字段，让 Agent 能从 stdout 判断。

3. 同步修改：
   - `docs/agents/output-contract.md:85`：把“保留原有退出策略”改为新策略；停用 JSON 增加 `failed` 字段的说明。
   - `packages/commands/tests/output/usage-output.test.ts`：现有用例只覆盖启用成功。补三条：`--off` 成功、`--off` 返回 `failureModels`（断言 stdout 不含 “Disabled”、有 `FREETIER_OPERATION_FAILED` warning、退出码非 0）、`--off` 超时。
   - CHANGELOG：记为 bug 修复，并注明退出码变化。

4. 如果不想扩大本 PR 范围，可以单独提修复 PR，但应先于或与本 PR 同版本发布。

## 建议项

### S1 识别依赖英文 message 正则，根因在 `mapApiError`

- 位置：`packages/core/src/errors/api.ts:18`。

  ```ts
  const rawCode = body.error?.type ?? body.error?.code ?? body.code;
  ```

- 问题：OpenAI 兼容接口返回 `{ type: "invalid_request_error", code: "AllocationQuota.FreeTierOnly" }` 时，`apiCode` 取到的是 `type`，真实 code 丢失。`isFreeTierOnlyExhausted` 因此加了 `/use free tier only/i` 等 message 正则兜底。服务端文案一改，识别就静默失效，且没有测试能发现。
- 解决方案：`ApiErrorContext` 增加 `apiErrorCode`，保留原始 `code`，不改 `apiCode` 的现有取值（避免影响 JSON 输出里的 `api_code`）：

  ```ts
  // base.ts
  export interface ApiErrorContext {
    httpStatus?: number;
    apiCode?: string;
    /** 原始 error.code；type 与 code 同时存在时 apiCode 取的是 type。 */
    apiErrorCode?: string;
    requestId?: string;
  }

  // api.ts
  const errorCode = body.error?.code ?? body.code;
  // ...
  api: {
    httpStatus: status,
    apiCode,
    apiErrorCode: typeof errorCode === "string" ? errorCode : undefined,
    requestId: body.request_id ?? body.requestID,
  },

  // error-handler.ts
  if (
    error.api?.apiCode === FREE_TIER_ONLY_API_CODE ||
    error.api?.apiErrorCode === FREE_TIER_ONLY_API_CODE
  ) return true;
  ```

  message 正则保留作兜底即可。

### S2 `chunk-delete.ts` 没走 `resolveRecoveryConfigName`

- 位置：`packages/commands/src/commands/knowledge/chunk-delete.ts:110`，传的是 `ctx.settings.configName`。
- 问题：显式 `--config default` 且激活 profile 为命名时，`settings.configName` 是 `undefined`，这条路径的 hint 会丢掉 `--config default`。这正是本 PR 第 4 个提交修的问题，这里漏了。
- 解决方案：命令层拿不到 `sources`。把结果放进 `Settings`，各处统一读取：
  - `Settings` 增加 `recoveryConfigName?: string`，在 `buildSettings` 里用 `resolveRecoveryConfigName(s)` 赋值（函数需要从 `error-handler.ts` 挪到 `core/config`）。
  - `chunk-delete.ts` 改传 `ctx.settings.recoveryConfigName`。

### S3 hint 去重靠字符串匹配

- 位置：`packages/runtime/src/error-handler.ts:145`，`!hint?.includes("usage freetier")`。
- 问题：已有 hint 只要出现这个词（哪怕不是恢复指引）就会跳过追加；反过来，命令层用别的措辞写了同样的指引，又会重复。
- 解决方案：恢复 hint 只由 runtime 这一处生成，那么去重只需要防止同一个错误被处理两次。在 `errorHint` 里比较完整的恢复文本：

  ```ts
  const recovery = freeTierRecoveryHint(commandPrefix, localize, configName);
  if (!hint?.includes(recovery)) {
    hint = hint ? `${hint}\n${recovery}` : recovery;
  }
  ```

  并在 `docs/agents/error-hint-change.md` 里写明：命令层不得自行拼 `usage freetier` 恢复指引。

### S4 `supportsFreeTierRecovery` 布尔值穿透通用 runtime

- 位置：`create-cli.ts:178` → `OutputReporterOptions` → `ErrorHintOptions`。
- 问题：通用 runtime 里出现了产品特定的布尔值。下次再加一种带命令的恢复 hint，又要多穿一个。
- 解决方案：改传通用的命令存在性判断：

  ```ts
  // create-cli.ts
  const hasCommand = (path: string) => path in commands;

  // error-handler.ts
  export interface ErrorHintOptions {
    hasCommand?: (path: string) => boolean;
    configName?: string;
  }
  const supportsFreeTier =
    options.hasCommand?.("usage free") && options.hasCommand?.("usage freetier");
  ```

  注意 `commands` 是静态命令表，不含 Command Pack 动态加载的命令。目前 usage 命令不在 pack 里，没有问题；如果以后要支持，需要改读 `loaded.commands`。

### S5 hint 里是 `<model>` 占位符

- 现状：hint 输出 `bl usage freetier --off --model <model>`，用户或 Agent 需要自己回填模型名。
- 解决方案（可选）：`errorHint` 所在层拿不到请求的模型名。如果要做，可以在抛出点给 `BailianError` 附带模型名（例如放进 `api` 上下文），`freeTierRecoveryHint` 有值时用实际模型名，没有时退回占位符。收益不大，可以不做。

## 确认没问题的部分

- hint 不带 `--yes`，调用方先撞确认门槛；skill 文档同步强调“不要自动补 `--yes`”。
- `kscli` 这类没有 usage 命令的产品不渲染 hint，有对应测试。
- TPS/TPM 的 `insufficient_quota` 不会误触发，有反例测试。
- 追加 hint 时保留原 message、exitCode 和已有 hint（如 video task id）。
- `resolveRecoveryConfigName` 对显式 `--config default`、隐式 default、命名 profile 三种情况的处理正确，有测试。
- profile 名受 `CONFIG_NAME_PATTERN` 约束（字母、数字、`-`、`_`），拼进命令行不需要转义。
- `confirmationStage` 在鉴权之前执行，`--dry-run` 跳过确认；e2e 覆盖了这两点。
- `skills/bailian-cli/reference/usage.md` 与 `tools/generate-reference.ts` 的输出一致。

## 建议的合并前清单

- [ ] P1：API Key 回退时，恢复 hint 的 profile 改为 `default`；补 2 条测试。
- [ ] P2：CHANGELOG 写明 `usage freetier` 要求 `--yes`（方案 A）；或改用 `prepare` 只拦 `--off`（方案 B）。
- [ ] P3：`--off` 失败不再输出 “Disabled”；与维护者确认是否改退出码；同步 `output-contract.md` 和测试。可拆成独立 PR，但需同版本发布。
- [ ] S2：`chunk-delete.ts` 统一走 `resolveRecoveryConfigName`。
- [ ] S1、S3、S4、S5：可在后续 PR 处理。
- [ ] 跑一遍 `pnpm run ready`（本次审查未能运行测试）。
