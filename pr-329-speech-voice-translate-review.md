# PR #329 审查：Qwen-MT 翻译命令、语音生成与音色管理

> 日期：2026-11-21 · 审查对象：`modelstudioai/cli` PR #329（head `bbe7edb9`，4 个提交，基于 main `50e66d2b`）· 状态：讨论稿

## 结论与范围

结论：PR 可以合并，没有阻塞性 bug。合并前建议处理 P1、P2；P3 属于健壮性风险，建议同一轮一起处理。

| # | 级别 | 问题 | 位置 | 建议 |
| --- | --- | --- | --- | --- |
| P1 | 需要处理（中） | `speech generate` 的 dry-run 请求与实际请求不一致：不传 `--audio` 时多出 `"references": []` | `speech/generate.ts:526` | 空数组时不输出 `references` |
| P2 | 需要处理（中） | `speech generate` 的区域判断是死代码；说明写"北京/新加坡"，默认 endpoint 实际只有北京 | `speech/generate.ts:182`、`:273`、`:413` | 删除死代码并改说明，或补上按区域选择 MaaS host |
| P3 | 需要处理（低） | qwen-mt-plus 流式累计内容不是前缀扩展时直接抛错，用户看到半截译文加报错 | `shared/chat-completion-stream.ts:114` | 改为降级：不再增量输出，结束时给出完整结果 |
| S1 | 建议 | `image translate` 同语言校验只比较字符串，`zh` 与 `Chinese` 不被识别为相同 | `image/translate.ts:215` | 先把语言名称和代码统一再比较 |
| S2 | 建议 | dry-run 中 `endpoint` 格式不统一：translate 输出相对路径，speech/voice 输出完整 URL | `image/translate.ts:331`、`omni/translate.ts:372` | 统一用 `ctx.client.url(path)` |
| S3 | 建议 | 三个 translate 命令的辅助函数重复；voice clone/design 的请求体构建了两次 | 多处 | 抽成共享模块，dry-run 与实际请求共用同一个请求体 |
| S4 | 建议 | `QwenMtUniRequest` 定义了 `glossary`、`sensitives`、`config`，`omni translate` 没有对应参数 | `core/src/types/api.ts:948` | 补参数，或先从类型中删除 |
| S5 | 建议 | `speech generate`、`speech voice` 的校验报错是纯英文，没走 `localize` | `speech/generate.ts`、`speech/voice/shared.ts` | 统一用 `LocalizedText` |
| S6 | 建议 | 用 `video task get` 查询图片/文档翻译任务，命名空间不直观 | `video/task-get.ts:45` | 后续增加通用的 `task get`，`video task get` 保留为别名 |
| S7 | 建议 | `omni` 既是命令又是命令组，`bl omni --help` 不列出 `omni translate` | `cli/src/commands.ts:401-402` | 在同时是命令和命令组的节点的帮助里追加子命令列表 |

PR 改动概览（50 个文件，+7619 / −190）：

- 新增 `text translate`（Qwen-MT 文本，Chat Completions `translation_options`）、`image translate`（qwen-mt-image 异步任务）、`omni translate`（qwen-mt-uni 文档/图片/音频文件翻译）。
- 新增 `speech generate`（qwen-audio-*-tts-next，播客/对话类音频生成，支持最多 3 条参考音频）。
- 新增 `speech voice clone / design / list / get / update / delete / preview`，core 新增 `speech/voice.ts` 封装 `/api/v1/services/audio/tts/customization`。
- `text chat` 的 Chat Completions 执行逻辑抽到 `text/run-chat.ts`；流式累加器新增 `contentMode: "delta" | "snapshot"`，供 qwen-mt-plus 使用。
- `video task get` 透传 `image_url` 以及 qwen-mt-uni 的 `Success / Code / Message / RequestId / Data`。
- registry smoke 的 `deriveGroupPaths` 跳过同时注册为命令的前缀（`omni` + `omni translate`）。

审查方法：

- 本地检出 PR head，阅读全部源码改动（测试与文档只抽查）。
- `vp check`：0 error，3 个 warning（都在 `tools/release`，与本 PR 无关）。
- 相关单测（`commands/tests/shared`、`core/tests/speech-voice.test.ts`、`core/tests/image-input.test.ts`、`e2e/tests`）：7 个文件、45 个用例全部通过。
- 用 `--dry-run --output json` 实际运行 `speech generate`、`image translate`、`text translate`、`omni --help`，复现 P1、P2、S1、S7。
- live e2e 没有运行（需要真实 API Key 和 workspace）。

## P1：`speech generate` 的 dry-run 与实际请求不一致

### 现象

```bash
bl speech generate --text-prompt hi --workspace-id ws1 --dry-run --output json
```

输出：

```json
{
  "endpoint": "https://ws1.cn-beijing.maas.aliyuncs.com/api/v1/services/audio/tts/SpeechSynthesizer",
  "request": {
    "model": "qwen-audio-3.1-tts-next",
    "input": { "text_prompt": "hi", "format": "wav", "references": [] }
  }
}
```

实际请求只有在 `references.length > 0` 时才设置 `input.references`，所以真正发出的请求里没有 `references` 字段。dry-run 的用途是让用户（以及 Agent）确认将要发出的请求，多一个字段就失去了这个保证；如果服务端对空 `references` 有不同处理，用户也无法从 dry-run 看出来。

### 原因

```ts
// speech/generate.ts:523
const dryInput = {
  ...input,
  references: redactReferences(references), // references 是 []，返回 []
};
```

`redactReferences` 只在入参为 `undefined` 时返回 `undefined`，而这里传入的始终是数组。

### 解决方案

dry-run 从同一个 `input` 派生，只替换已有的 `references`：

```ts
const dryInput =
  input.references !== undefined
    ? { ...input, references: redactReferences(references) }
    : input;
```

测试：在 `speech-generate` 的 dry-run 用例里断言，不传 `--audio` 时 `request.input` 不含 `references` 键（`expect("references" in input).toBe(false)`）。

## P2：`speech generate` 的区域判断是死代码

### 现象

- 说明写的是 "Default endpoint needs a workspace ID (Beijing/Singapore)"（`speech/generate.ts:413`）。
- 实际上，不配置 base URL 时，endpoint 永远是 `https://<ws>.cn-beijing.maas.aliyuncs.com`。
- 传 `--base-url https://dashscope-intl.aliyuncs.com` 时，endpoint 变成 `https://dashscope-intl.aliyuncs.com/api/v1/services/audio/tts/SpeechSynthesizer`，`--workspace-id` 被忽略，也不会拼出新加坡的 MaaS host。

### 原因

`Client.url(path, defaultBaseUrl)` 只有在 `baseUrlIsDefault` 为 true 时才调用回调：

```ts
// core/src/client/client.ts:139
url(path: string, defaultBaseUrl?: () => string): string {
  const baseUrl = this.deps.baseUrlIsDefault && defaultBaseUrl ? defaultBaseUrl() : this.baseUrl;
  return baseUrl + path;
}
```

而 `baseUrlIsDefault` 为 true 意味着没有任何来源配置过 base URL，此时 `ctx.client.baseUrl` 一定是 `REGIONS.cn`。所以回调里的

```ts
const cloudRegion = cloudRegionFromBaseUrl(ctx.client.baseUrl) ?? "cn-beijing"; // 恒为 cn-beijing
if (!SUPPORTED_CLOUD_REGIONS.has(cloudRegion)) { ... }                          // 永不触发
```

永远得到 `cn-beijing`，`SUPPORTED_CLOUD_REGIONS` 检查永远不会触发。`cloudRegionFromBaseUrl` 里对 intl、MaaS host 的解析在这条路径上都用不到。

`assertSupportedAudioEndpoint` 对显式 `--base-url` 的 us 区域拦截是有效的（e2e 也覆盖了），不受影响。

### 解决方案

两种做法，取决于产品是否需要新加坡默认 endpoint。

**方案 A（推荐，改动小）：承认默认只有北京。**

- 回调里删除 `cloudRegionFromBaseUrl` 和 `SUPPORTED_CLOUD_REGIONS` 检查，直接 `return workspaceMaaSBaseUrl(workspaceId, "cn-beijing")`。
- `cloudRegionFromBaseUrl` 如无其他调用方，一并删除。
- notes 改为："默认 endpoint 为北京 MaaS（需要 workspace ID）。新加坡或其他 endpoint 请通过 `--base-url` / `DASHSCOPE_BASE_URL` / Profile `base_url` 显式指定（例如 `https://<ws>.ap-southeast-1.maas.aliyuncs.com`）。"
- 保留 `assertSupportedAudioEndpoint`。

**方案 B：支持按区域选择默认 MaaS host。**

- 新增 `--maas-region <cn-beijing|ap-southeast-1>` 参数（或读取已有 workspace 配置中的区域，如果有），默认值为 `cn-beijing`。
- 回调中使用该值：`workspaceMaaSBaseUrl(workspaceId, flags.maasRegion ?? "cn-beijing")`，并保留 `SUPPORTED_CLOUD_REGIONS` 校验（这时才有意义）。
- 不要从 `ctx.client.baseUrl` 推断区域：回调被调用时它恒为北京。

测试：

- dry-run 不配置 base URL：endpoint 为 `ws1.cn-beijing.maas...`。
- 方案 B：`--maas-region ap-southeast-1` 时 endpoint 为 `ws1.ap-southeast-1.maas...`；传 `us-east-1` 返回 USAGE 错误。

## P3：qwen-mt-plus 流式累计内容异常时直接抛错

### 现象

`contentMode: "snapshot"` 时，每个片段被当作"截至目前的全文"，只输出新增的后缀：

```ts
// shared/chat-completion-stream.ts:111
if (fragment === content) return "";
if (!fragment.startsWith(content)) throw snapshotProtocolError();
```

如果服务端在后续片段里修改了已经输出过的前文（例如修正标点或词序），就会抛出 "Stream protocol error"。这时前面的译文已经写到 stdout，用户得到半截译文、一个非 0 退出码，而且没有完整结果。

另外，"plus 系列流式返回全文、flash/lite 返回增量"这一判断（`text/translate.ts:19`）没有标注文档出处，模型行为变化时不容易发现。

### 解决方案

前缀不匹配时不报错，改为降级：

```ts
let snapshotDiverged = false;

const applyContentFragment = (fragment: string): string => {
  if (!fragment) return "";
  if (contentMode === "delta") {
    content += fragment;
    return fragment;
  }
  if (fragment === content) return "";
  if (!snapshotDiverged && fragment.startsWith(content)) {
    const suffix = fragment.slice(content.length);
    content = fragment;
    return suffix;
  }
  // 前文被改写：停止增量输出，只记录最新全文，由 finish() 给出完整结果
  snapshotDiverged = true;
  content = fragment;
  return "";
};
```

`finish()` 返回的结果里增加 `content_revised: true`（仅在降级发生时）。`runChatCompletions` 在 text 模式下检测到这个标记时，向 stderr 输出一行提示，然后把完整 `content` 换行后重新输出到 stdout，保证用户最终拿到完整译文。JSON 模式下 `content` 本来就是最终全文，不需要额外处理。

如果团队倾向于保留"严格模式"，至少要把已收到的完整 `fragment` 放进错误的 `details`，避免译文丢失。

同时在 `QWEN_MT_SNAPSHOT_MODEL` 旁边注释写明对应的官方文档链接，并在 `tests/shared/chat-completion-stream.test.ts` 增加用例：snapshot 序列 `"A" → "AB" → "Ab C"`，断言不抛错，并且 `finish().content === "Ab C"`。

## 建议项

### S1：`image translate` 同语言校验只比较字符串

`--source-lang zh --target-lang Chinese` 能通过 `assertLanguagePair`，请求会发到服务端后才失败（或者得到无意义的结果）。

解决方案：复用 `isChineseOrEnglish` 已有的别名表，抽成 `normalizeLang`：

```ts
const LANG_ALIASES: Record<string, string> = {
  zh: "zh", "zh-cn": "zh", chinese: "zh",
  en: "en", "en-us": "en", "en-gb": "en", english: "en",
};
const normalizeLang = (lang: string) => LANG_ALIASES[lang.trim().toLowerCase()] ?? lang.trim().toLowerCase();
```

同语言比较和 `isChineseOrEnglish` 都基于 `normalizeLang`。`zh-tw` 不应映射为 `zh`（繁体与简体互译是合法需求），保持原值。

### S2：dry-run 中 `endpoint` 格式不统一

- `image translate`、`omni translate` 输出的是 `/api/v1/services/...`（相对路径）。
- `speech voice *`、`speech generate` 输出的是完整 URL。

解决方案：translate 两个命令改为 `ctx.output.result({ endpoint: ctx.client.url(endpoint), request })`，与 speech 系列一致。dry-run 测试同步改为断言完整 URL（或断言 `endsWith(path)`）。

### S3：重复代码

重复的内容：

- `isHttpUrl`、`hasNonAscii`：`image/translate.ts`、`omni/translate.ts` 各一份。
- `source=target` 术语对解析：`text/translate.ts`（输出 `source/target`）、`image/translate.ts`（输出 `src/tgt`）各一份。
- 异步任务轮询中对 `FAILED / CANCELED / UNKNOWN` 的处理：image、omni 两份，结构几乎相同。
- voice clone/design：dry-run 用 `buildVoiceRequest(input)`，实际调用又把同样的字段重新传给 `createCloneVoice` / `createDesignVoice`。新增字段时只改一处，dry-run 与实际请求就会不一致（与 P1 是同一类问题）。

解决方案：

1. 新建 `commands/src/commands/shared/translation.ts`，导出 `isHttpUrl`、`hasNonAscii`、`parseTermPairs(terms, localize)`（返回 `{ source, target }[]`，image 侧再映射为 `{ src, tgt }`）。
2. 新建 `shared/dashscope-task.ts`，导出 `pollDashScopeTask(ctx, taskId, { pollInterval, extraCheck? })`；omni 的 `Success !== true` 判断通过 `extraCheck` 传入。
3. voice：core 增加 `postVoiceRequest(client, request: VoiceRequest)`；clone/design 先构建好 `request`，dry-run 输出脱敏后的 `request`，实际调用直接发送同一个 `request`。`createCloneVoice` / `createDesignVoice` 可以保留，内部改为调用 `buildVoiceRequest` + `postVoiceRequest`，供其他调用方使用。

### S4：`QwenMtUniRequest` 有字段但命令没有对应参数

类型里有 `ext.glossary`、`ext.sensitives`、`ext.config.imageSegment`，但 `omni translate` 只暴露了 `--format-hint` 和 `--domain-hint`；另外两个 translate 都支持 `--term`。

解决方案（二选一）：

- 补参数：`--term <source=target>`（映射为 `glossary`，复用 S3 的 `parseTermPairs`）、`--sensitive <text>`（可重复）、`--image-segment <bool>`；同步更新 `skills/bailian-gen/reference/omni.md`。
- 先不支持：从类型中删除这三个字段，避免误导后续维护者。

### S5：报错文案没有本地化

`speech generate` 的 `validate` 返回值以及 `buildReference`、`speech/voice/shared.ts` 中的 `BailianError` 都是纯英文字符串；同一个 PR 里的 translate 命令都使用了 `localize`。

解决方案：`validate` 返回 `LocalizedText`（框架已支持，translate 命令的 `--poll-interval` 校验就是这样写的）；`shared.ts` 中会抛错的函数增加 `localize` 参数。这是纯机械改动，可以放在后续 PR 中。

### S6：用 `video task get` 查询翻译任务

`image translate`、`omni translate` 的 notes 都建议用户通过 `video task get --task-id` 查询异步任务，而 `video task get` 为此透传了 `Success / Code / Message / RequestId / Data` 等 qwen-mt-uni 专有字段。

解决方案：后续增加通用的 `task get --task-id`（DashScope 任务查询本来就与模态无关），把透传逻辑移过去；`video task get` 保留为别名，避免破坏现有用法。本 PR 可以不改，但 notes 里的指引在新命令上线后要同步更新。

### S7：`omni` 既是命令又是命令组

路由按最长前缀匹配，`bl omni ...` 与 `bl omni translate ...` 都能正确分发，registry smoke 也已在 `deriveGroupPaths` 中处理。问题在于可发现性：`bl omni --help` 只渲染 omni 对话命令自己的帮助，只能靠 notes 里一句话找到 `omni translate`。

解决方案：在 runtime 渲染命令帮助时，如果该节点还有子命令，在帮助末尾追加一节 "Subcommands:"，列出子命令及其描述（可复用 `locate()` 中生成 "Available commands" 的逻辑）。这对以后其他同时是命令和命令组的节点同样适用。

## 做得好的地方

- `text chat` 重构没有改变原有行为：dry-run、`--prime`、`--batch` 三种输出结构都保留，Responses API 路径未受影响。
- 下载文件名使用随机 hex，不信任远端 URL 路径；`--out-prefix` 做了防目录穿越校验。
- voice design 保存预览音频失败、或响应缺少预览数据时，提示音色已经创建，不要重复创建。
- `voice update`、`voice delete` 标注了 `risk: high`。
- qwen-mt-uni 任务 `SUCCEEDED` 后仍检查 `Success === true`，否则按业务失败处理。
- `redactDataUri` 支持带参数的 Data URI（如 `;codecs=...`），dry-run 不会回显音频内容。
- `isSpeechGenerateE2EReady` 在缺少 workspace 时跳过测试，而不是空跑通过。

## 建议的处理顺序

1. 本 PR 内：P1（一行改动加一个断言）、P2 方案 A（删除死代码并改说明）、P3（降级逻辑加一个单测）。
2. 本 PR 或紧随其后：S1、S2（改动小，并且影响 dry-run 输出，越早定下越好）。
3. 后续 PR：S3 重构、S4、S5、S6、S7。
