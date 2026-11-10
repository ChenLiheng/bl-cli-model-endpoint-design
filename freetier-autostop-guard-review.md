# `feat/freetier-autostop-guard` 分支审查：问题与解决方案

> 日期：2026-11-10 · 审查对象：`modelstudioai/cli` 分支 `feat/freetier-autostop-guard`（`30f32bbe`，基于 main `fa8330c2`）· 状态：讨论稿

## 结论与范围

结论：分支方向正确、实现干净，但有 1 个与“guard”目标直接冲突的问题（R1），建议合并前修掉；另有 2 个需要处理的问题和 3 个建议项。

| # | 级别 | 问题 | 建议 |
| --- | --- | --- | --- |
| R1 | 需要处理（高） | 错误 hint 直接给出带 `--yes` 的停用命令，Agent 照抄即绕过确认门槛 | hint 去掉 `--yes` |
| R2 | 需要处理（中） | 确认门槛对“启用”方向也生效，是未声明的破坏性变更 | 保留门槛，补 CHANGELOG 与双向风险文案；或改用 `prepare` 只拦 `--off` |
| R3 | 需要处理（低） | 文档写“按标记去重”，代码只拼接不去重 | 补去重逻辑 |
| S1 | 建议 | `mapApiError` 用 `type` 覆盖 `code`，识别依赖英文 message 正则 | 保留原始 `code`，正则只作兜底 |
| S2 | 建议 | 测试缺口：zh-CN hint、`create-cli.ts` 能力门控推导 | 补 2 组用例 |
| S3 | 建议 | `exampleArgs` 六条的中英文完全相同 | 改回字符串数组 |

分支改动概览（1 个提交，10 个文件，+274 / −32）：

- `usage freetier` 声明 `risk: { level: "high", reason: "billing" }`，不带 `--yes` 时退出码 7（`requires_confirmation`）。
- 服务端返回 `AllocationQuota.FreeTierOnly` 时，`errorHint` 追加恢复指引（`usage free` 查状态、`usage freetier --off` 停用）。
- 新增能力门控 `supportsFreeTierRecovery`：命令表里没有 `usage free` / `usage freetier` 的产品（如 `kscli`）不渲染该 hint。
- 同步更新 skill 参考文档、错误 hint 变更规范，新增单测与 e2e。

审查方法：

- 只做了静态审查：完整阅读 diff，以及 `error-handler.ts`、`middleware.ts`（`confirmationStage`）、`preparation.ts`、`mapApiError`、`tools/generate-reference.ts`、e2e 隔离环境等上下游代码。
- **没有运行测试**：审查环境中安装依赖被权限策略拒绝，下文所有结论均来自读代码。
- 没有修改 `modelstudioai/cli` 的代码；下文的代码片段是建议写法，未经编译验证。

## 需要处理

### R1 错误 hint 带 `--yes`，替 Agent 绕过确认门槛

- 位置：`packages/runtime/src/error-handler.ts:66`（`freeTierRecoveryHint`）。
- 现状：FreeTierOnly 错误的 hint 渲染为

  ```text
  Check status: bl usage free --model <model>
  If Auto-Stop is ON and you accept pay-as-you-go: bl usage freetier --off --model <model> --yes
  ```

- 问题：同一分支在 `skills/bailian-cli/reference/usage.md` 写了“Never add `--yes` automatically”，但 hint 是 Agent 出错后最先读到、最可能照抄的文本。照抄这条命令就会一步关掉自动停止并开始按量计费，完全不经过退出码 7 和风险文案。这个分支新增的门槛，在它最该生效的场景（Agent 遇到额度耗尽后自行恢复）里被自己的 hint 绕开了。
- 次要：`skills/bailian-protocol/SKILL.md:229` 的汇报示例改成了 `--off --yes`。那是事后汇报的语境，影响小，但同样在示范“带 `--yes` 是常态”。

解决方案：hint 里不写 `--yes`，让调用方先撞上确认门槛。

```ts
function freeTierRecoveryHint(commandPrefix: string, localize: Localize): string {
  const statusCommand = formatCommand(commandPrefix, "usage free --model <model>");
  const disableCommand = formatCommand(commandPrefix, "usage freetier --off --model <model>");
  return localize({
    "en-US": `Check status: ${statusCommand}\nTo continue on a pay-as-you-go basis, disable Auto-Stop (requires confirmation): ${disableCommand}`,
    "zh-CN": `查看状态：${statusCommand}\n如需按量计费继续调用，可停用 Auto-Stop（需确认）：${disableCommand}`,
  });
}
```

- 效果：Agent 执行 hint 中的命令 → 退出码 7 + 风险文案 + 通用的“add --yes to the original command”提示 → 按 skill 规范停下来向用户确认。人类用户多敲一次命令，代价很小。
- 配套修改：
  - `packages/runtime/tests/error-handler.test.ts` 中 3 处 `--yes` 断言改为断言 hint **不含** `--yes`。
  - `skills/bailian-protocol/SKILL.md:229` 的示例改回 `bl usage freetier --off`，或在句中写明“经用户确认后”。

### R2 确认门槛对“启用”方向也生效

- 位置：`packages/commands/src/commands/usage/freetier.ts:24`（静态 `risk`）。
- 现状：`risk` 是静态声明，运行时的 `confirmationStage` 不区分 flag。所以 `bl usage freetier --model x`（启用自动停止，不会产生费用）现在也返回退出码 7。
- 问题：
  - 破坏性变更：现有脚本和 Agent 流程里不带 `--yes` 的调用会全部中断，而分支没有更新 CHANGELOG。
  - 风险文案（“Disabling it allows pay-as-you-go calls…”）只解释了停用方向，启用方向被拦时用户看到的理由对不上。

解决方案（二选一）：

**方案 A（推荐）：保留双向门槛，把它声明清楚。** 启用自动停止也有后果——额度用尽后调用会直接失败，对线上业务是中断风险，双向拦截说得通。改动最小：

1. 风险文案覆盖两个方向：

   ```ts
   message: {
     "en-US":
       "This changes free-tier auto-stop. Enabling it makes calls fail once free quota is exhausted; disabling it allows pay-as-you-go charges after free quota is exhausted.",
     "zh-CN":
       "此操作会变更免费额度自动停止。启用后，免费额度用尽时调用将失败；停用后，免费额度用尽将继续按量计费调用。",
   },
   ```

2. `CHANGELOG.md` / `CHANGELOG.zh.md` 增加 breaking 条目：`bl usage freetier` 现在需要 `--yes`，不带时退出码 7。
3. 重新生成 `skills/bailian-cli/reference/usage.md`（Risk message 行会变）。

**方案 B：只拦 `--off`，用 `prepare` 按方向返回风险。** 仓库里已有这个模式（如 `knowledge/doc-sync.ts`：保留静态 `risk` 用于帮助和 `--yes` flag，`prepare` 返回实际风险）：

```ts
async prepare(ctx) {
  const off = Boolean(ctx.flags.off);
  const models = parseModels(ctx.flags.model); // 抽出 run 里现有的解析逻辑
  return {
    data: { off, models },
    preview: { action: off ? "disable-auto-stop" : "enable-auto-stop", models, all: Boolean(ctx.flags.all) },
    risk: off ? FREETIER_RISK : null,
    notices: [],
  };
},
```

方案 B 的代价需要事先知道：

- 有 `prepare` 的命令会跳过 `confirmationStage`，确认发生在鉴权**之后**。未登录时不带 `--yes` 会先得到 AUTH（3）而不是 7，分支里的 e2e 用例“无 --yes 返回 7”要相应调整。
- `--dry-run` 的输出会变成运行时统一的 `{ plan, risk, notices }`，并且 dry-run 也要求鉴权通过；现有的 `{ api, data }` 输出契约（`docs/agents/output-contract.md`）要同步改。
- 帮助和参考文档里仍会显示 `Risk: high`，需要用 note 说明“仅 `--off` 需要确认”。

选择依据：如果“启用方向不能破坏现有调用”是硬要求，选 B；否则选 A，成本低得多。

### R3 文档写“按标记去重”，代码没有去重

- 位置：`docs/agents/error-hint-change.md:109` 对 `packages/runtime/src/error-handler.ts:101-104`。
- 现状：文档要求“追加时若已有 hint，拼接并按标记去重”；代码只做 `hint ? \`${hint}\n${recovery}\` : recovery`。
- 影响：目前 `errorHint` 每个错误只调用一次，不会真的重复；但文档是给后续改动者（含 Agent）的规范，写了不存在的行为会误导。若将来某个命令自己在 `error.hint` 里放了同样的恢复指引，就会输出两遍。

解决方案：按文档补上去重（与同函数里 AUTH 分支的 marker 写法一致），三行即可。

```ts
if (options.supportsFreeTierRecovery && isFreeTierOnlyExhausted(error)) {
  // 已含恢复指引（命令自带或上游已追加）时不重复追加。
  if (!hint?.includes("usage freetier")) {
    const recovery = freeTierRecoveryHint(commandPrefix, localize);
    hint = hint ? `${hint}\n${recovery}` : recovery;
  }
}
```

并补一条单测：`error.hint` 已含 `usage freetier` 时，输出里该字符串只出现一次。如果不想加逻辑，就把文档里“并按标记去重”删掉，二者保持一致即可。

## 建议项

### S1 识别依赖英文 message 正则，根因在 `mapApiError`

- 位置：`packages/core/src/errors/api.ts:18`、`packages/runtime/src/error-handler.ts:53-60`。
- 现状：`mapApiError` 取 `body.error?.type ?? body.error?.code ?? body.code` 作为 `apiCode`。OpenAI 兼容接口返回 `type: "invalid_request_error"`、`code: "AllocationQuota.FreeTierOnly"` 时，真实 code 被 `type` 盖掉，于是 `isFreeTierOnlyExhausted` 只能靠 `/use free tier only/i` 等正则兜底。分支里的单测“type shadows code”正是在固化这个绕路。
- 风险：
  - 正则绑定英文服务端文案，文案改动或返回中文时会漏报。
  - 正则对所有 `BailianError` 的 message 生效，不限于模型调用错误（误报概率低，但范围比需要的大）。

解决方案：在 `mapApiError` 里把原始 `code` 单独保留，识别优先用它。

```ts
// packages/core/src/errors/api.ts
const rawErrorCode = body.error?.code ?? body.code;
return new BailianError(apiMsg, ExitCode.GENERAL, undefined, {
  api: {
    httpStatus: status,
    apiCode, // 保持现有取值与输出，不破坏 api_code 契约
    errorCode: typeof rawErrorCode === "string" ? rawErrorCode : undefined,
    requestId: body.request_id ?? body.requestID,
  },
});

// packages/runtime/src/error-handler.ts
function isFreeTierOnlyExhausted(error: BailianError): boolean {
  if (error.api?.apiCode === FREE_TIER_ONLY_API_CODE) return true;
  if (error.api?.errorCode === FREE_TIER_ONLY_API_CODE) return true;
  return /use free tier only/i.test(error.message); // 仅兜底：异步任务等不经 mapApiError 的路径
}
```

- `errorCode` 是否进入 JSON 错误输出需要单独决定；不输出则对外契约零变化。
- 这一条可以不随本分支合并，单独提 PR。

### S2 测试缺口

- zh-CN hint 没有覆盖：`render()` 测试辅助函数没传 `localize`，中文文案（含全角冒号、命令拼接）从未被断言。补一条传入 zh-CN `localize` 的用例。
- `packages/runtime/src/create-cli.ts:178` 的门控推导（`"usage free" in commands && "usage freetier" in commands`）没有测试；现有用例都是直接给 reporter 传布尔值。补两条 `createCli` 级用例：命令表含两条 path 时 hint 出现，缺任一条时不出现。这样将来命令改名或拆到 command pack 时能第一时间发现 hint 静默消失。

### S3 `exampleArgs` 的冗余本地化

- 位置：`packages/commands/src/commands/usage/freetier.ts:59-83`。
- 现状：六条示例都写成 `{ "en-US": X, "zh-CN": X }`，两种语言内容完全相同。`exampleArgs` 的类型是 `LocalizedText[]`，接受纯字符串。

解决方案：

```ts
exampleArgs: [
  "--model qwen3-max --yes",
  "--model qwen3-max,qwen-turbo --yes",
  "--all --yes",
  "--on --model qwen3-max --yes",
  "--off --model qwen3-max --yes",
  "--off --all --yes",
],
```

## 确认没问题的部分

- `--dry-run` 在任何远程请求前返回，符合 `risk` 的契约（`packages/core/src/types/command.ts:342`）。
- 追加 hint 不改动原 message 和 exitCode；`mapApiError` 恒为 GENERAL，不会误入 AUTH 分支追加 API Key 指引。
- `insufficient_quota`（TPS / TPM 限流）有反例测试，不会误触发。
- `kscli` 等没有 usage 命令的产品被能力门控挡住。
- `usage.md` 中的 Risk 行、`--yes` flag 行、“Only after explicit user confirmation”注释与 `tools/generate-reference.ts` 的生成规则一致。
- e2e 第 3 条用例（`--yes` 后落到 AUTH）依赖隔离 HOME，不会读到宿主机的 console 凭证。

## 建议的合并前清单

1. R1：hint 去掉 `--yes`，改 3 处测试断言，SKILL.md 示例回退。
2. R2：定方案 A 或 B；选 A 则改风险文案、补 CHANGELOG、重新生成参考文档。
3. R3：补去重逻辑和 1 条单测（或删文档中的那句）。
4. S2、S3 顺手带上；S1 单独提 PR。
5. 跑一遍 `vp check` 与 `vp test`（本次审查未能运行）。
