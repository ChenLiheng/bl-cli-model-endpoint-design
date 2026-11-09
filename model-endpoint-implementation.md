# bl CLI 模型端点统一改造 · 落地实现方案

> 日期：2026-11-09 · 状态：讨论稿

## 概述

本文是「bl CLI 模型端点统一改造方案」的实施稿：按 13 个 Step 顺序编码，每步给出文件、接口与代码骨架；未拍板的点给出多个实现方案，标记为【待取舍 X】，由负责人最终选择。

- 代码基线：`modelstudioai/cli` main `cadc301d`（v2.8.4）。文中行号均指这个版本。
- 前提：方案文档的 P0 验证已完成，D1–D10 已拍板。未拍板时，按每处标注的「推荐」先实现。
- 约定：代码为骨架级别，类型与函数签名以本文为准；用户可见文案以 Step 11 错误文案表为准。
- 每个 Step 结束运行 `vp check` 与 `vp test <相关目录>`，全部完成后运行 `pnpm ready`。

| Step | 层 | 内容 | 依赖 |
| --- | --- | --- | --- |
| 1 | core | `config/model-endpoint.ts` | — |
| 2 | core | `region` 字段、`BAILIAN_REGION`、`--region` | 1 |
| 3 | core | resolver、`ApiKeyCredential`、`Client` | 1, 2 |
| 4 | core | `client/endpoints.ts` workspace 端点 | 3 |
| 5 | core | upload、refresh-token、batch、security、公网例外 | 3 |
| 6 | runtime | middleware、create-cli、error-handler | 3 |
| 7 | commands | 登录 | 3, 6 |
| 8 | commands | config 与业务命令 | 4, 5, 6 |
| 9 | core + commands | 旧配置迁移 | 3, 7 |
| 10 | kscli、dsh | 入口与插件 | 4 |
| 11 | 全局 | 错误文案 | — |
| 12 | 全局 | 测试 | 各步 |
| 13 | 全局 | 文档、skill、发布 | 全部 |

建议先做 PoC：Step 1–3 + Step 6，再仅接入 `text chat`，跑通后再铺开 Step 4–8。

## Step 1 core：新增 model-endpoint.ts

新增 `packages/core/src/config/model-endpoint.ts`，是域名规则的唯一出处；从 `config/index.ts` 与包入口导出。只依赖 `model-base-url.ts`、`errors/*`，不依赖 runtime。

```ts
// packages/core/src/config/model-endpoint.ts
import { normalizeModelBaseUrl } from "./model-base-url.ts";
import type { CredentialSource } from "../auth/types.ts";

export const DEFAULT_MODEL_REGION = "cn-beijing";

/** 已知 workspace 地域；仅用于登录探测与提示，不限制用户输入。来源：login-api-key.ts ORDINARY_WORKSPACE_REGIONS */
export const KNOWN_MODEL_REGIONS = [
  "cn-beijing", "ap-southeast-1", "cn-hongkong",
  "ap-northeast-1", "eu-central-1", "us-east-1",
] as const;

const WORKSPACE_ID_RE = /^[a-zA-Z0-9_-]+$/;          // 与 endpoints.ts:16 保持一致
// region 校验规则见下方【待取舍 S1】
const MAAS_HOST_RE = /^([a-z0-9_-]+)\.([a-z0-9-]+)\.(maas|pre-maas)\.aliyuncs\.com$/i;

export type ModelEndpointKind = "workspace" | "token-plan" | "custom";

export interface ModelEndpoint {
  origin: string;                 // 永远是规范化 origin，无尾斜杠
  kind: ModelEndpointKind;
  workspaceId?: string;           // kind=workspace 必有
  region?: string;                // kind=workspace / token-plan 必有
  preRelease?: boolean;           // pre-maas 域名
  source: CredentialSource;       // 赢家来源（flag / env / config）
}

export function isValidWorkspaceId(id: string): boolean { return WORKSPACE_ID_RE.test(id); }
export function isValidRegion(r: string): boolean { /* 见下方【待取舍 S1】 */ }

/** 拼接专属 Host；非法输入抛 UsageError（文案见 Step 11 E1/E2）。 */
export function buildWorkspaceOrigin(workspaceId: string, region: string, opts?: { preRelease?: boolean }): string {
  // 校验 workspaceId / region → `https://${workspaceId}.${region}.${opts?.preRelease ? "pre-maas" : "maas"}.aliyuncs.com`
}

/** 将任意 URL 分类；不抛错，由调用方决定如何处理 dashscope。 */
export type ParsedOrigin =
  | { kind: "workspace"; origin: string; workspaceId: string; region: string; preRelease: boolean }
  | { kind: "token-plan"; origin: string; region: string }
  | { kind: "dashscope"; origin: string; legacyRegion?: string }
  | { kind: "custom"; origin: string };

export function classifyModelOrigin(url: string): ParsedOrigin {
  const origin = normalizeModelBaseUrl(url);
  const host = new URL(origin).hostname.toLowerCase();
  // 1. LEGACY_DASHSCOPE_HOSTS 命中 → dashscope（带 legacyRegion 映射）
  // 2. token-plan.{region}.maas.aliyuncs.com → token-plan（必须在 workspace 判断之前）
  // 3. MAAS_HOST_RE 命中 → workspace
  // 4. 其他 → custom
}

/** 旧域名 → region；映射待后端确认（方案文档 R4）。 */
export const LEGACY_DASHSCOPE_HOSTS: Readonly<Record<string, string>> = {
  "dashscope.aliyuncs.com": "cn-beijing",
  "dashscope-intl.aliyuncs.com": "ap-southeast-1",
  "dashscope-us.aliyuncs.com": "us-east-1",
  "cn-hongkong.dashscope.aliyuncs.com": "cn-hongkong",
};
```

【待取舍 S1】region 校验强度：

- **S1-A 宽松正则**（推荐）：`/^[a-z]{2,}(-[a-z0-9]+)+$/`，允许未来新地域；未知地域只在登录探测失败时提示。
- **S1-B 白名单**：只接受 `KNOWN_MODEL_REGIONS`；报错最友好，但新地域必须发版。

注意：`token-plan` 的判断必须先于 workspace 正则，否则 `token-plan` 会被当成 workspace ID（方案文档 R7）。`pre-maas` 只在显式 base_url 中出现，拼接时默认不生成。

单测：新增 `packages/core/tests/model-endpoint.test.ts`，用例见 Step 12 T1–T8。

## Step 2 core：配置字段 region、环境变量、全局 flag

新增一个配置字段、一个环境变量、一个 flag；region 与 `console_region` 是两个字段，前者管模型域，后者管控制台网关。

| 文件 | 改动 |
| --- | --- |
| `core/src/config/schema.ts` | `ConfigFile` 加 `region?: string`；`CONFIG_FILE_KEYS` 加 `"region"`；`parseConfigFile` 用 `isValidRegion` 过滤；`Settings` 加 `region?: string` |
| `core/src/config/schema.ts` | `REGIONS` 保留但加 `@deprecated` 注释，仅供旧域名识别；新代码改用 `LEGACY_DASHSCOPE_HOSTS` |
| `core/src/config/loader.ts:319` | `buildSettings` 加 `region: flags.region \|\| env.BAILIAN_REGION?.trim() \|\| file.region \|\| undefined` |
| `core/src/config/loader.ts:221` | `selectApiKeyResolutionSources` 的「显式覆盖」判断加入 `flags.workspaceId`、`flags.region`、`env.BAILIAN_REGION` |
| `core/src/config/validation.ts` | `config set region` 的校验文案 |
| `core/src/auth/store.ts:28` | `AuthPersistPatch` 加 `"region"`；`LOGOUT_KEYS.all` 是否含 `workspace_id` / `region` 见【待取舍 S2】 |
| `core/src/types/command.ts:150` | `MODEL_AUTH_FLAGS` 加 `workspaceId` 与 `region` |

```ts
// packages/core/src/types/command.ts
export const MODEL_AUTH_FLAGS = {
  apiKey:  { /* 不变 */ },
  baseUrl: { /* 不变，description 改为「覆盖模型服务 origin（预发 / 代理）」 */ },
  workspaceId: {
    type: "string", valueHint: "<id>",
    description: { "en-US": "Workspace ID for the model endpoint (env: BAILIAN_WORKSPACE_ID)",
                   "zh-CN": "模型端点使用的 Workspace ID（环境变量：BAILIAN_WORKSPACE_ID）" },
  },
  region: {
    type: "string", valueHint: "<region>",
    description: { "en-US": "Model service region, e.g. cn-beijing (env: BAILIAN_REGION)",
                   "zh-CN": "模型服务地域，如 cn-beijing（环境变量：BAILIAN_REGION）" },
  },
} satisfies FlagsDef;
```

硬约束：`runtime/src/registry.ts:122-131` 在命令重复声明保留 flag（GLOBAL / 确认 / 凭证域 flag）时直接 `throw`。因此把 `workspaceId` / `region` 放进 `MODEL_AUTH_FLAGS` 后，所有 `auth: "apiKey"` 且自带同名 flag 的命令必须删掉自己的声明：`PRIME_FLAGS.workspaceId`（chat / video）、`WORKSPACE_FLAG`（知识库 / 记忆 / sandbox / security / parsex 中用 apiKey 的部分）、`decision predict --region`、`knowledge doc-import-oss --region`（若为 apiKey 命令）。实施前先跑 `grep -rn "workspaceId:\|region:" packages/commands/src` 核对完整清单。

【待取舍 S3】`--region` 与已有同名 flag 的冲突：

- **S3-A（推荐）** 全局 `--workspace-id` + `--region` 进 `MODEL_AUTH_FLAGS`；删除各命令的同名声明；`decision predict` 直接读 `settings.region`（含义一致）；`doc-import-oss --region` 改名 `--oss-region`（与 `collection-create --oss-region` 一致，旧名保留一个版本的隐藏别名若 runtime 支持）。一致性最好，`doc-import-oss` 有破坏性。
- **S3-B** 全局只加 `--model-region`，`workspaceId` 仍由各命令声明；endpoint 解析从 `flags.workspaceId` 读取（各命令声明同名即可生效）。零破坏，但没声明 `--workspace-id` 的命令（如 image / speech）无法用 flag 切空间，只能靠环境变量或配置。
- **S3-C** 放进 `GLOBAL_FLAGS`（所有命令可见，含 console / openapi 命令）。最简单，但 help 会在不相关的命令上也展示这两个 flag，且与 `CONSOLE_AUTH_FLAGS.workspaceId` 冲突，需同时从那里删除。

【待取舍 S2】`auth logout --all` 是否清除 `workspace_id` / `region`：S2-A 不清除（推荐，它们是资源定位不是凭证）；S2-B 清除（与今天清 `base_url` 的行为对等）。

## Step 3 core：resolver / ApiKeyCredential / Client 改造

`resolveModelBaseUrlState` 替换为 `resolveModelEndpoint`，结果注入 Client；模型请求时才报「缺 workspace」，保证 `auth: "none"` 命令与 dry-run 不受影响。

### 3.1 resolver（`core/src/auth/resolver.ts`）

```ts
export type ModelEndpointResolution =
  | { ok: true; endpoint: ModelEndpoint }
  | { ok: false; error: BailianError };   // E3 缺 workspace / E4 dashscope 废弃 / E5 冲突

export function resolveModelEndpoint(s: ResolutionSources): ModelEndpointResolution {
  // 1. 显式 base_url（flag > env DASHSCOPE_BASE_URL > file.base_url）→ classifyModelOrigin
  //    - custom / token-plan：直接返回（workspaceId / region 取各自链，仅作参数）
  //    - dashscope：见【待取舍 S7】
  //    - workspace：反解出的 ws / region 作为候选值，带上 base_url 的来源层级，进入第 2 步
  // 2. workspaceId 候选：flags.workspaceId / env.BAILIAN_WORKSPACE_ID / file.workspace_id / 第 1 步反解值
  //    region 候选：flags.region / env.BAILIAN_REGION / file.region / 第 1 步反解值
  //    → pickByTier()（见【待取舍 S6】）
  // 3. 无 workspaceId → { ok: false, error: E3 }
  // 4. region 缺省 → DEFAULT_MODEL_REGION（或 file.console_region，见【待取舍 S8】）
  // 5. origin = buildWorkspaceOrigin(ws, region, { preRelease })
}

/** 兼容旧调用方；解析失败时抛错。下个大版本删除。 */
/** @deprecated */ export function resolveModelBaseUrl(s: ResolutionSources): string
```

- `resolveModelBaseUrlSource` / `resolveModelConnectionProvenance` / `isModelConnectionSourceMismatch` 保留：provenance 的 `baseUrlSource` 改为 `endpoint.source`。
- `resolveApiKey(s)`：只解析 token，不再计算 baseUrl。返回的 `ApiKeyCredential.baseUrl` 改为可选，或保留为 `endpoint.origin`（见【待取舍 S9】）。

【待取舍 S6】base_url 反解值与 workspace_id / region 不一致时：

- **S6-A 严格**：任意来源之间不一致就报 E5。最安全，但配置里 ws-A、命令行 `--workspace-id ws-B` 也会报错（如果配置里还留有 maas base_url）。
- **S6-B 按层级（推荐）**：反解值与 base_url 同层级参与竞争，flag > env > config 取最高层；只有同一层级内出现两个不同值才报 E5（如同时传 `--base-url ws-A` 和 `--workspace-id ws-B`）。符合直觉，且配置里不再存 maas base_url 后，配置层冲突只会出现在旧配置里。

```ts
function pickByTier<T>(cands: Array<{ value?: T; tier: CredentialSource; from: string }>):
  { value?: T; tier?: CredentialSource } | { conflict: [string, string] } {
  for (const tier of ["flag", "env", "config"] as const) {
    const vals = cands.filter(c => c.tier === tier && c.value !== undefined);
    if (vals.length === 0) continue;
    const distinct = new Set(vals.map(v => v.value));
    if (distinct.size > 1) return { conflict: [vals[0].from, vals[1].from] };
    return { value: vals[0].value, tier };
  }
  return {};
}
```

【待取舍 S8】region 缺省时是否用 `console_region` 兜底：S8-A 直接 `cn-beijing`（推荐，行为可预测）；S8-B 先用 `console_region`，再 `cn-beijing`（国际站用户少配一项，但两个概念会被混用）。

### 3.2 类型（`core/src/auth/types.ts`）

【待取舍 S9】`ApiKeyCredential.baseUrl` 的去留：

- **S9-A（推荐）** 保留 `baseUrl: string`（= `endpoint.origin`），新增 `endpoint: ModelEndpoint`。managed-agent `_engine/credentials.ts` 与 Command Pack host 通过 credential 导出 baseUrl，保留可以不动它们。
- **S9-B** 删除 `baseUrl`，一律读 `endpoint.origin`。干净，但要改 Command Pack 对外契约（`docs/agents/command-pack.md`）。

### 3.3 Client（`core/src/client/client.ts`）

```ts
export interface ClientDeps {
  // 删除：baseUrl: string; baseUrlIsDefault?: boolean;
  modelEndpoint: ModelEndpointResolution;
  /* 其余不变 */
}

class Client {
  /** 解析失败时在这里抛（懒抛）。 */
  get endpoint(): ModelEndpoint { const r = this.deps.modelEndpoint; if (!r.ok) throw r.error; return r.endpoint; }
  /** dry-run / config show 用：不抛错。 */
  get endpointPreview(): ModelEndpointResolution { return this.deps.modelEndpoint; }
  get baseUrl(): string { return this.endpoint.origin; }
  url(path: string): string { return this.endpoint.origin + path; }

  /** workspace 域服务用；可声明支持的地域，不在列表内抛 E6。custom 端点跳过校验。 */
  workspaceUrl(path: string, opts?: { service: string; regions?: readonly string[] }): string

  private toOpts({ path, ...rest }) { /* url: absolute ? path : this.url(path) */ }
}
```

【待取舍 S5】`Client.url(path, defaultBaseUrl?)` 的第二个参数：

- **S5-A（推荐）** 直接删除参数与 `baseUrlIsDefault`；TypeScript 会把所有调用方（prime、decision、sandbox、batch、video generate）标红，逐一改掉，漏改不了。
- **S5-B** 保留参数但忽略（`@deprecated`），分批迁移。改动小，但会残留死代码与误导性的签名。

## Step 4 core：endpoints.ts 的 workspace 端点改造

目标：删除所有写死 `cn-beijing` 的拼接函数，workspace 域服务一律经 `client.workspaceUrl()` 取地址，并在一处声明各服务支持的地域。

### 4.1 删除 / 合并的函数

| 现函数（`client/endpoints.ts`） | 处理 | 调用方 |
| --- | --- | --- |
| `workspaceEndpoint(ws, path)` :15 | 删除 | 仅本文件内部 |
| `ragEndpoint(ws, path)` :452 | 见【待取舍 S10】 | knowledge/\* 约 60 处 |
| `memoryEndpoint(ws, path)` :148 | 同上 | memory/\* |
| `knowledgeSearchEndpoint(ws)` / `knowledgeChatEndpoint(ws)` :218/:224 | 同上 | knowledge search / chat |
| `parsexBaseUrl(ws)` :25 | 删除；`parsexEndpoint(baseUrl, …)` 保留 | parsex/shared.ts:245 |
| `agentStudioHost(ws)` :236 | 删除 | security/shared.ts:36 |
| `sandboxBaseUrl(ws)` / `sandboxEndpoint(ws, path)` :464/:474 | 删除；`sandboxApiPath()` 保留 | sandbox/shared.ts:105、sandbox/file.ts:76 |
| `workspaceMaaSBaseUrl(ws, region)` :213 | 删除 | shared/prime.ts:62、decision/shared.ts:202 |

### 4.2 服务地域声明

```ts
// packages/core/src/client/endpoints.ts
/** 各 workspace 域服务支持的地域；undefined = 不限制。取值由方案文档 P0 矩阵确定，下表为占位。 */
export const WORKSPACE_SERVICE_REGIONS = {
  model: undefined,                       // chat / image / video / speech / tasks
  knowledge: ["cn-beijing"],              // TBD(P0)
  memory: ["cn-beijing"],                 // TBD(P0)
  parsex: ["cn-beijing"],                 // TBD(P0)
  sandbox: ["cn-beijing"],                // 现文案已声明仅 cn-beijing
  agentstudio: ["cn-beijing"],            // endpoints.ts:231 注释：后端固定
} as const satisfies Record<string, readonly string[] | undefined>;
export type WorkspaceService = keyof typeof WORKSPACE_SERVICE_REGIONS;
```

`Client.workspaceUrl(path, { service })`：`endpoint.kind === "custom"` → 不校验直接拼；`kind === "token-plan"` → 抛 E7（Token Plan 不提供 workspace 服务）；`kind === "workspace"` 且 region 不在列表 → 抛 E6。

### 4.3 调用方迁移

【待取舍 S10】约 74 处 `ragEndpoint` / `memoryEndpoint` / `knowledge*Endpoint` 调用怎么改：

- **S10-A（推荐）改为返回相对路径** — `ragPath(path)` / `memoryPath(path)` 只返回 path，调用方写 `ctx.client.workspaceUrl(ragPath(RAG_PATHS.x), { service: "knowledge" })`。地址只有一个来源，且与其他模型域 path builder 形态一致。缺点：74 处都要改，可用 codemod（正则 ` (rag|memory)Endpoint\((\w+),  ` → …）。
- **S10-B 保留签名、第一个参数改为 client** — `ragEndpoint(ctx.client, path)`，内部调用 `client.workspaceUrl(path, { service: "knowledge" })`。改动最机械（每处只换第一个实参），但 endpoints.ts 要依赖 Client 类型（用 `Pick<Client, "workspaceUrl">` 避免循环引用）。
- **S10-C 保留签名，加可选 region 参数** — `ragEndpoint(ws, path, region?)`，调用方传 `settings.region`。改动最小，但 workspace 仍有两个来源（`resolveWorkspaceId` 与 endpoint），不解决「一半打 A 一半打 B」。不推荐。

`resolveWorkspaceId(ctx)`（`commands/shared/workspace.ts:28`）保留给需要在 body / query 里传 workspace ID 的接口，实现改为优先取 `ctx.client.endpointPreview` 中的 `workspaceId`，保证与域名一致。

## Step 5 core：上传、刷新 token、Batch、安全中心、公网例外

这一步把所有「用正则猜域名」的地方换成 `classifyModelOrigin()`，并把仍需公网的 API 集中到一个文件。

### 5.1 临时上传（`core/src/files/upload.ts:36`）

`resolveUploadBaseUrl(baseUrl, apiKey)` 保留签名，内部改为：

```ts
const parsed = classifyModelOrigin(baseUrl);
switch (parsed.kind) {
  case "token-plan": throw unsupportedTemporaryUpload("Token Plan");
  case "workspace":
    if (!SUPPORTED_UPLOAD_REGIONS.includes(parsed.region)) throw unsupportedTemporaryUpload(parsed.region);
    rejectOfficialTokenPlanKey(apiKey); return parsed.origin;
  case "dashscope": /* 弃用期内保留旧逻辑；移除后不会走到这里 */
  case "custom": return parsed.origin;
}
```

`upload.ts:279` 的默认值 `baseUrl = REGIONS.cn` 删除，改为必填；调用方均已从 Client 传入。

### 5.2 刷新 CLI token（`core/src/auth/refresh-token.ts:19`）

`resolveCLIAccessTokenTarget(settings, baseUrl)` 中从 hostname 推导 region 的部分改为：

```ts
const p = classifyModelOrigin(baseUrl);
region = p.kind === "workspace" || p.kind === "token-plan" ? p.region
       : p.kind === "dashscope" ? p.legacyRegion : undefined;
```

仍只接受 `MODEL_STUDIO_HOSTS` 中的 cn-beijing / ap-southeast-1。可选：签名改为直接接收 `ModelEndpoint`，省掉一次解析。

### 5.3 公网例外集中管理

新增 `core/src/client/public-endpoints.ts`，把仍只部署在公网的 API 集中在一处，每项注明依据与移除条件。

```ts
/** 仍只部署在公网的 API。每项在 workspace 域名上线后删除。依据：方案文档 P0 矩阵。 */
export const PUBLIC_ONLY_ENDPOINTS = {
  customModelImport: "https://dashscope.aliyuncs.com/api/v1/custom_models/import", // endpoints.ts:337
  batchChat: "https://batch.dashscope.aliyuncs.com",                              // endpoints.ts:80
  bailianMcp: "https://dashscope.aliyuncs.com",                                  // mcp/connect.ts:107
  cmodel: "https://dashscope.aliyuncs.com",                                      // cmodel/shared.ts
} as const;
```

【待取舍 S11】公网例外对非北京用户怎么处理：

- **S11-A（推荐）固定北京公网** — 与现状一致（上述 API 今天就只走北京），风险最小。
- **S11-B 按 region 映射到对应公网** — 用 `LEGACY_DASHSCOPE_HOSTS` 反查，ap-southeast-1 → `dashscope-intl`。更合理，但需验证每个公网站点是否都有这些 API。
- **S11-C 直接走 workspace 域名** — 仅当 P0 矩阵证明 workspace 域名已支持时采用，对应项从常量中删除。

### 5.4 Batch（`core/src/batch/api.ts:59`）

`resolveBatchChatEndpoint` 改为：endpoint 为 custom → 用它；否则 → `PUBLIC_ONLY_ENDPOINTS.batchChat + chatPath()`；token-plan → 报现有错误。原来拦截 `dashscope(-intl|-us)` 的正则删除（这些域名已不会出现在 endpoint 中）。若 P0 证明 workspace 域名支持同步 Batch，则按 S11-C 处理。

### 5.5 安全中心（`core/src/client/security.ts:15-26`）

删除 `DASHSCOPE_GATEWAY_ORIGINS` 与 `isDashScopeGateway`（及 `client/index.ts:88` 的导出）。「是否显式覆盖」的判断由 `endpoint.kind === "custom"` 代替，由 `client.workspaceUrl(..., { service: "agentstudio" })` 统一处理。

### 5.6 其他

- `core/src/advisor/engine/debug-loop.ts:5`：开发调试脚本，改为从 `BAILIAN_WORKSPACE_ID` + `BAILIAN_REGION` 拼接，缺失则直接退出并提示。
- `core/src/client/headers.ts` 的 `x-dashscope-*` 请求头不动（方案文档 D9）。

## Step 6 runtime：middleware、create-cli、error-handler

runtime 只负责把 `resolveModelEndpoint` 的结果注入 Client，并调整来源提示与网络错误文案。

| 位置 | 现状 | 改为 |
| --- | --- | --- |
| `runtime/src/create-cli.ts:312` | `...resolveModelBaseUrlState(sources)` | `modelEndpoint: resolveModelEndpoint(sources)` |
| `runtime/src/middleware.ts:195` | `base` 展开 `resolveModelBaseUrlState(sources)` | `modelEndpoint: resolveModelEndpoint(sources)` |
| `runtime/src/middleware.ts:224`、`:236` | 用 `apiSources` 再算一次 baseUrl | `modelEndpoint: resolveModelEndpoint(apiSources)`（Profile capability 回落后的来源） |
| `runtime/src/middleware.ts:110-123` | 来源标签只认 `--base-url` / `DASHSCOPE_BASE_URL` | 增加 `endpoint` 字段类型：`--workspace-id` / `BAILIAN_WORKSPACE_ID` / `--region` / `BAILIAN_REGION` |
| `runtime/src/middleware.ts:126-166` | api_key 与 base_url 来源不同时提示 | 比较对象改为 api_key 与 `endpoint.source` |
| `runtime/src/middleware.ts:104` | `credential_fields: ["api_key", "base_url"]` | 加入 `"workspace_id"`、`"region"` |
| `runtime/src/error-handler.ts:93-96` | 「如果 DashScope 主机地址有误，请检查 baseUrl」 | E8：提示检查 workspace / region，并打印实际请求的 origin |

要点：

- authStage 不在这里抛「缺 workspace」，由 `Client.endpoint` 懒抛；例外是 `auth: "apiKey"` 且非 dry-run 的命令：在运行前先访问一次 `client.endpoint`，让报错早于任何本地副作用（如文件写入）。需要确认 `command.prepare` 流程不受影响。
- `maybeShowStatusBar`（:250）如显示地址，改为显示 `workspaceId @ region`。
- E8 的实际 origin 从哪里取：`classifyRuntimeError` 目前只拿到 `commandPrefix`，需要新增可选参数 `endpoint?: ModelEndpointResolution`，由调用方从 ctx 传入。

## Step 7 commands：登录流程

`auth login --api-key` 不再探测 dashscope 公网，必须得到一个 workspace；校验通过后落盘 `workspace_id` + `region`，而不是完整 base_url。

### 7.1 flag（`commands/auth/login.ts:30`）

- 新增 `workspaceId`、`region`（若 Step 2 选 S3-A，则由 `MODEL_AUTH_FLAGS` 提供；`auth login` 的 `auth` 类型需确认是否会自动带上这些 flag，否则在命令内声明）。
- `baseUrl` 保留，描述改为「可粘贴控制台完整 Endpoint，自动拆解 workspace 与 region」。

### 7.2 校验候选（`commands/auth/login-api-key.ts:127`）

```ts
export function apiKeyValidationCandidates(key, options) {
  // 1. explicitBaseUrl：classifyModelOrigin
  //    - workspace → 只探测该 origin
  //    - token-plan / custom → 与现状相同
  //    - dashscope → 抛 E4（不探测）
  // 2. options.workspaceId + options.region → 只探测 buildWorkspaceOrigin(ws, region)
  // 3. options.workspaceId，无 region → 并发探测 KNOWN_MODEL_REGIONS（现 ORDINARY_WORKSPACE_REGIONS 逻辑）
  // 4. token-plan key → KNOWN_BASE_URLS["token-plan"]（不变）
  // 5. 删除 KNOWN_BASE_URLS.ordinary 的 4 个 dashscope 域名
  // 6. 普通 key 且没有 workspace → 抛 E3（不发任何请求）
}
```

【待取舍 S12】普通 Key 登录时 workspace 从哪里来：

- **S12-A 必须显式提供** — `--workspace-id`、`--base-url`、`BAILIAN_WORKSPACE_ID` 或已存配置之一，否则 E3。实现最简单、行为最确定。
- **S12-B 用控制台列表补全** — 已有 console 登录态时，调 `listConsoleWorkspaces()`（`commands/shared/workspace.ts:54`）：TTY 下交互选择，非 TTY 下取 `defaultAgent: true` 的那个；仍无结果则 E3。体验最好，但 apiKey 登录开始依赖 console 凭证。
- **S12-C 用 Key 反查** — 若后端提供「由 Key 查所属 workspace」的接口（方案文档 R3），直接用它。目前未知是否存在。
- 推荐：先实现 S12-A，S12-B 作为增强在同一版本或下一个小版本加入。

### 7.3 落盘（`login-api-key.ts:223` `persistApiKey`）

| 校验结果 kind | 写入 | 删除 |
| --- | --- | --- |
| workspace | `api_key`、`workspace_id`、`region` | `base_url` |
| token-plan | `api_key`、`base_url`（现状）、Profile 预设 | — |
| custom | `api_key`、`base_url`；若传了 `--workspace-id` 一并写入 | — |

实现注意：`authStore.login()`（`core/src/auth/store.ts:95-110`）目前只做写入。需要确认 `undefined` 是跳过还是删除，并新增显式删除语义（如 `null` 表示删除），否则旧的 dashscope `base_url` 会残留并覆盖新的 workspace 端点。

### 7.4 其他登录分支

- `--console`（浏览器回调）：回调会携带 key 和 endpoint（`persistApiKey` 注释）。**未验证**回调的 endpoint 是否仍是 dashscope；如果是，需要控制台侧配合改造，或 CLI 在回调后用 S12-B 补全 workspace。
- `--open-api`：`login.ts:273` 用 `store.resolveBaseUrl()` 推导控制台地域，改为用 `settings.region` 或已存 endpoint；两者都没有时要求 `--console-region`。

## Step 8 commands：config 命令与各业务命令接入

各命令删除自带的「未配置时用 workspace 拼域名」兜底，改为直接使用 `ctx.client.url()` / `ctx.client.workspaceUrl()`。

### 8.1 config 命令

| 文件 | 改动 |
| --- | --- |
| `config/set.ts` | `--key base_url`：值为 maas workspace 域名时拆成 `workspace_id` + `region` 写入并删除 `base_url`，提示一次；值为 dashscope 时报 E4。新增 `--key region`。示例 `:32` 改为 `--key workspace_id --value llm-xxx` |
| `config/show.ts:45-120` | 删除 `:70` 的 `REGIONS.cn` 默认；输出新增 `endpoint`：`{ origin, kind, workspace_id, region, source }`，解析失败时输出 `endpoint_error`；`DASHSCOPE_BASE_URL` 来源标记保留 |
| `config/ui.ts:252` | 初始化表单不再预填 `base_url: REGIONS.cn`，改为 workspace_id + region 两个输入项 |
| `config/agent/index.ts:28-85` | 示例 URL 改为 `https://llm-xxx.cn-beijing.maas.aliyuncs.com/...`；未传 `--base-url` 时用当前 endpoint origin 拼 `/compatible-mode/v1` 或 `/apps/anthropic`（后者待 P0 验证） |
| `config/agent/writers/utils.ts:180-216` | `host.includes("dashscope")` 判断改为 `classifyModelOrigin`；示例文案更新 |

### 8.2 业务命令

| 文件 | 现状 | 改为 |
| --- | --- | --- |
| `shared/prime.ts:44-64` | `client.url(path, () => workspaceMaaSBaseUrl(ws))` | `ctx.client.url(path)`；见【待取舍 S14】 |
| `decision/shared.ts:182-204` | 同上，带 `--region` | `ctx.client.url(decisionPath())`；`DECISION_REGIONS` 约束改为校验 `endpoint.region` |
| `sandbox/shared.ts:105`、`sandbox/file.ts:76` | `sandboxBaseUrl(resolveWorkspaceId(ctx))` | `ctx.client.workspaceUrl(sandboxApiPath(p), { service: "sandbox" })` |
| `security/shared.ts:28-37` | `isDashScopeGateway` + `agentStudioHost` | `ctx.client.workspaceUrl("", { service: "agentstudio" })` 取 origin |
| `parsex/shared.ts:245` | `parsexBaseUrl(workspaceId)` | `parsexEndpoint(ctx.client.workspaceUrl("", { service: "parsex" }), …)` |
| `knowledge/*`、`memory/*` | `ragEndpoint(ws, path)` 等 | 按 Step 4 S10 |
| `video/generate.ts:299/331/445` | `defaultBaseUrl: () => resolvePrimeEndpoint(...)` | 删除 `defaultBaseUrl`，直接用相对路径 |
| `text/chat.ts:335` | prime 分支用 `resolvePrimeEndpoint` | 与普通分支合并 |
| `mcp/connect.ts:107` | `${REGIONS.cn}${path}` | `PUBLIC_ONLY_ENDPOINTS.bailianMcp`（S11） |
| `cmodel/shared.ts`、自定义模型导入 | 写死公网 | `PUBLIC_ONLY_ENDPOINTS`（S11） |
| `managed-agent/_engine/credentials.ts:13` | 注释示例 dashscope | 更新注释；若 S9-B 则改读 `endpoint.origin` |
| `managed-agent/webhook.ts:92`、`sandbox/file.ts:57`、`sandbox/shared.ts:73`、`text/chat.ts:244`、`video/generate.ts:201`、`decision/predict.ts:111`、`security/*.ts:47/125`、`parsex/shared.ts:20` | help 文案描述旧优先级 | 统一替换为新优先级文案（Step 11 H1） |

【待取舍 S14】`--prime` 的去留（方案文档 D8）：

- **S14-A（推荐）保留为无作用开关** — 接受但不改变行为，输出一次弃用警告；`validatePrimeFlags` 中「`--workspace-id` 必须搭配 `--prime`」的限制删除（workspace 已是全局参数）；「`--prime` 必须显式 `--model`」的限制是否保留待确认。
- **S14-B 直接删除** — 脚本带 `--prime` 会报「未知 flag」。
- **S14-C 保留并赋予新含义** — 仅当 Prime 在后端不只是「走 workspace 域名」（例如还代表不同的计费或路由）时才有意义。**拿不准**：需确认 Prime 的业务含义。

## Step 9：旧配置迁移

迁移分两件事：运行时遇到 dashscope 怎么办（S7），以及什么时候、由谁把配置文件改写掉（S15）。

### 9.1 运行时处理

【待取舍 S7】`resolveModelEndpoint` 遇到 dashscope 域名：

- **S7-A 弃用期兼容（推荐用于 2.9.x）** — 若能得到 workspace（flag / env / 配置），用 `legacyRegion` 作为 region 的最低优先级候选，拼出 workspace origin，输出 W1 弃用警告；得不到 workspace 时仍使用 dashscope origin（kind 记为 `custom`）并输出 W1。行为平滑，弃用期内两套逻辑并存。
- **S7-B 忽略并继续解析** — 当作未配置 base_url，走 workspace + region；没有 workspace 则 E3。简单，但老用户升级后立刻不可用。
- **S7-C 直接报错 E4（推荐用于 3.0）** — 提示运行迁移命令或重新登录。最清晰。

选择依赖方案文档 D10 发布节奏：可用一个内部常量 `LEGACY_DASHSCOPE_POLICY: "compat" | "error"` 切换，2.9 取 compat，3.0 取 error。

### 9.2 配置文件改写

【待取舍 S15】何时改写磁盘上的 `~/.bailian/config.json`：

- **S15-A 显式命令（推荐）** — 新增 `bl config migrate [--workspace-id <id>] [--region <r>] [--all-profiles] [--dry-run]`。逐 Profile 按下表处理，先探测再写盘，输出变更清单。可预览、可审计，符合「不静默改用户文件」。
- **S15-B 读配置时自动改写** — 在 `buildSources` 中发现 dashscope base_url + workspace_id 即写回。用户无感，但读操作产生写副作用，并发命令可能争抢文件，且无法探测。
- **S15-C 只在 `auth login` 时顺带改写** — 登录成功后按 7.3 落盘规则覆盖。最少新代码，但不重新登录的用户永远不会被迁移。
- 可组合：S15-A + S15-C。

```ts
// packages/commands/src/commands/config/migrate.ts（S15-A）
for (const profile of targetProfiles) {
  const file = readProfile(profile);
  const plan = planMigration(file, flags);   // 纯函数，放 core，便于单测
  // plan.action: "noop" | "rewrite" | "needs-workspace" | "conflict"
  if (plan.action === "rewrite" && !settings.dryRun) {
    await probe(plan.origin, file.api_key);   // 复用 validateApiKey，失败则不写
    await configStore.patch(profile, { workspace_id, region, base_url: null });
  }
  results.push({ profile, ...plan });
}
```

| 当前 Profile | `planMigration` 结果 |
| --- | --- |
| dashscope base_url + workspace_id | rewrite：region = `--region` → `legacyRegion`；写入前探测，失败则对 `KNOWN_MODEL_REGIONS` 并发探测 |
| dashscope base_url，无 workspace_id，且传了 `--workspace-id` | 同上 |
| dashscope base_url，无 workspace_id | needs-workspace |
| 无 base_url，有 api_key（普通 key） | 有 workspace_id 则 rewrite（只补 region），否则 needs-workspace |
| maas workspace base_url | rewrite：拆为 workspace_id + region；与已有 workspace_id 不同 → conflict |
| token-plan / custom | noop |

## Step 10：kscli 与 dsh 插件

kscli 没有自己的域名代码，跟随 core 即可；两个 dsh 插件按约定不依赖 core，需要各自增加 region 配置项。

### 10.1 kscli（`packages/kscli`）

- `src/` 中没有 dashscope / maas 引用（grep 已确认）。
- 需要做的：确认 kscli 的命令重映射不会因 Step 2 新增的 `--workspace-id` / `--region` 全局 flag 与自身 flag 冲突（`registry.ts:122` 会抛错）；更新 kscli 的 README 与示例。

### 10.2 bailian-kb-dsh

| 位置 | 改动 |
| --- | --- |
| `src/endpoints.ts:22` `kbEndpoint(endpointHost, ws, path)` | 签名不变 |
| `src/index.ts:152`、`:178`、`:258` | 三处写死 `"cn-beijing.maas.aliyuncs.com"` 改为 `` `${region}.maas.aliyuncs.com` ``，region 取自插件配置 |
| 插件配置表单 | 新增 `region` 项，默认 `cn-beijing`，可选值来自方案文档 P0 矩阵中知识库支持的地域 |

### 10.3 bailian-memo-dsh

| 位置 | 改动 |
| --- | --- |
| `src/endpoints.ts:6` `DEFAULT_ENDPOINT_HOST` | 保留为默认值 |
| `memoryEndpoint(ws, path, endpointHost?)` | 调用方传入 `` `${region}.maas.aliyuncs.com` `` |
| `~/.bailian/personal-memory/config.json` | 新增 `region` 字段（可选）；读取时缺省 `cn-beijing` |

【待取舍 S16】dsh 插件是否纳入本次：

- **S16-A** 本次一起改（七包锁步版本，保持一致）。
- **S16-B（推荐）** 本次不改：插件本来就只用 workspace 域名，不受 dashscope 废弃影响；等 P0 确认知识库 / 记忆支持非北京地域后再加 region。

## Step 11：错误文案表

所有新增错误都是 CLI 能权威解释的本地错误，符合 AGENTS.md 第 3 条；服务端 401 / 404 仍原样透传。hint 中的命令用 `formatCommand(identity, …)` 生成，不写死 `bl`。

| ID | 触发 | ExitCode | message（en / zh） | hint（en / zh） |
| --- | --- | --- | --- | --- |
| E1 | workspace ID 含非法字符 | USAGE | Workspace ID must contain only letters, digits, hyphens or underscores. / Workspace ID 只能包含字母、数字、连字符或下划线。 | 沿用现有文案（`endpoints.ts:17`） |
| E2 | region 不合法 | USAGE | Invalid region "{r}". / 地域 "{r}" 无效。 | Use a region id such as cn-beijing or ap-southeast-1. / 请使用 cn-beijing、ap-southeast-1 等地域 ID。 |
| E3 | 模型请求但无 workspace | USAGE | No model endpoint configured: a workspace ID is required. / 未配置模型端点：需要 Workspace ID。 | Run `auth login --api-key <key> --workspace-id <id>`, pass --workspace-id, or set BAILIAN_WORKSPACE_ID. / 执行 `auth login --api-key <key> --workspace-id <id>`，或传 --workspace-id，或设置 BAILIAN_WORKSPACE_ID。 |
| E4 | 出现 dashscope 公网域名（3.0 起） | USAGE | The public DashScope endpoint {origin} is no longer supported. / 已不再支持公共 DashScope 端点 {origin}。 | Run `config migrate`, or use --workspace-id and --region. Source: {source}. / 执行 `config migrate`，或改用 --workspace-id 与 --region。来源：{source}。 |
| E5 | 同层级 workspace / region 不一致 | USAGE | Conflicting workspace: {a} ({fromA}) vs {b} ({fromB}). / Workspace 冲突：{a}（{fromA}）与 {b}（{fromB}）。 | Keep only one of them. / 请只保留其中一个。 |
| E6 | 服务不支持当前地域 | USAGE | {service} is not available in {region}. / {service} 暂不支持地域 {region}。 | Supported: {list}. Use --region or a workspace in a supported region. / 支持的地域：{list}。请改用 --region 或支持地域中的 workspace。 |
| E7 | Token Plan 端点调用 workspace 服务 | USAGE | {service} is not available on the Token Plan endpoint. / Token Plan 端点不支持 {service}。 | Use an ordinary API key profile. / 请切换到普通 API Key 的 Profile。 |
| E8 | 网络错误的地址提示（替换 `error-handler.ts:93`） | — | — | If the endpoint {origin} is wrong, check workspace_id / region (`config show`). / 如果端点 {origin} 有误，请检查 workspace_id / region（`config show`）。 |
| W1 | 2.9 弃用期内使用 dashscope | 警告，不退出 | The public DashScope endpoint is deprecated and will be removed in 3.0. / 公共 DashScope 端点已弃用，将在 3.0 移除。 | Run `config migrate`. / 请执行 `config migrate`。 |
| W2 | 使用 `--prime`（S14-A） | 警告 | --prime is no longer needed; all model requests use the workspace endpoint. / 无需再传 --prime，所有模型请求已使用 workspace 端点。 | — |

警告的输出通道遵循 `docs/agents/output-contract.md`（stderr 诊断，`--quiet` 下是否抑制以该文档为准）。

H1 help 统一文案（替换 Step 8.2 列出的各命令 notes）：

- en-US：Endpoint: `--base-url` > `DASHSCOPE_BASE_URL` > profile `base_url` (custom / pre-release only); otherwise `https://{workspace}.{region}.maas.aliyuncs.com`, with workspace from `--workspace-id` > `BAILIAN_WORKSPACE_ID` > `workspace_id` and region from `--region` > `BAILIAN_REGION` > `region` > cn-beijing.
- zh-CN：请求地址：`--base-url` > `DASHSCOPE_BASE_URL` > Profile `base_url`（仅用于自定义 / 预发）；否则为 `https://{workspace}.{region}.maas.aliyuncs.com`，workspace 按 `--workspace-id` > `BAILIAN_WORKSPACE_ID` > `workspace_id`，region 按 `--region` > `BAILIAN_REGION` > `region` > cn-beijing。

## Step 12：测试用例清单

新增用例集中在解析与迁移两个纯函数上；存量约 48 个测试文件中的 dashscope 断言需逐一更新。测试隔离（HOME、环境变量）按 `docs/agents/test-isolation.md`，尤其要清理 `BAILIAN_WORKSPACE_ID` / `BAILIAN_REGION` / `DASHSCOPE_BASE_URL`。

### 12.1 `packages/core/tests/model-endpoint.test.ts`

| ID | 输入 | 期望 |
| --- | --- | --- |
| T1 | `buildWorkspaceOrigin("llm-xxx", "cn-beijing")` | `https://llm-xxx.cn-beijing.maas.aliyuncs.com` |
| T2 | workspace ID 含 `.` 或 `/` | 抛 E1 |
| T3 | region 为 `CN_BEIJING` / 空串 | 抛 E2 |
| T4 | classify `https://llm-x.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1` | workspace，ws=`llm-x`，region=`ap-southeast-1`，origin 无 path |
| T5 | classify `https://token-plan.cn-beijing.maas.aliyuncs.com` | token-plan（不是 workspace） |
| T6 | classify `https://llm-x.cn-beijing.pre-maas.aliyuncs.com` | workspace，preRelease=true |
| T7 | classify 4 个 dashscope 域名 | dashscope，legacyRegion 按映射表 |
| T8 | classify `http://localhost:8080` | custom |

### 12.2 `packages/core/tests/config-priority.test.ts`（扩展现有锁定表）

| ID | 来源组合 | 期望 endpoint |
| --- | --- | --- |
| T10 | 仅 file.workspace_id=A | A @ cn-beijing，source=config |
| T11 | file A + env BAILIAN_REGION=ap-southeast-1 | A @ ap-southeast-1 |
| T12 | file A + `--workspace-id B` | B（flag 胜出） |
| T13 | `--base-url` maas(A, sg) + file.workspace_id=B | S6-B：A @ sg；S6-A：E5 |
| T14 | `--base-url` maas(A) + `--workspace-id B` | E5（同层级冲突） |
| T15 | `--base-url http://localhost:9000` + file A | custom，origin=localhost，workspaceId=A |
| T16 | 仅 api_key，无任何 workspace | `ok:false`，E3；`config show` 不抛 |
| T17 | file.base_url=dashscope + file A | S7-A：A @ cn-beijing + W1；S7-C：E4 |
| T18 | env DASHSCOPE_BASE_URL=token-plan | token-plan |
| T19 | Profile capability 回落（`selectApiKeyResolutionSources`）且传了 `--workspace-id` | 不回落，按显式覆盖处理 |

### 12.3 其他单测

- `Client`：`url()` / `workspaceUrl()` 在 workspace / custom / token-plan 三种 kind 下的结果；E6、E7；`endpoint` 懒抛。
- `upload.ts`：workspace 在 cn-beijing / ap-southeast-1 通过，在 us-east-1 拒绝；token-plan 拒绝。
- `refresh-token.ts`：从 workspace origin 推导 region。
- `login-api-key.ts`：候选列表不再含 dashscope；普通 key 无 workspace 时不发请求直接 E3；落盘删除旧 base_url。
- `planMigration`：Step 9.2 表中 6 行各一个用例。
- registry：所有命令注册不因新 flag 抛「redeclares reserved flag」（覆盖 bl 与 kscli 两个入口）。

### 12.4 输出契约与 E2E

- `config show --output json` 新增 `endpoint` 字段的快照。
- 各命令 `--dry-run` 输出的 URL 快照由 dashscope 改为 workspace 域名。
- E2E（`packages/e2e`）：`global-setup.ts:35` 的 `DASHSCOPE_BASE_URL=` 改为注入 `BAILIAN_WORKSPACE_ID` / `BAILIAN_REGION`；需要一个真实测试 workspace，凭证走 CI secret。
- 存量更新：先跑 `grep -rln "dashscope" packages --include=*.test.ts` 列清单，按文件逐一修订。

## Step 13：文档、skill 与发布

代码合入前必须同步的非代码资产如下；提交与发版均由负责人确认后再进行。

- [ ] `pnpm run sync:skill-assets`：重新生成各 `skills/<skill>/reference/`（新 flag 与 help 文案会自动进入）。
- [ ] 手改 skill 正文中的 dashscope 示例（`grep -rln "dashscope.aliyuncs" skills`），特别是 `bailian-protocol` 与 `bailian-cli` hub 的登录步骤。
- [ ] README.md / README.zh.md / INSTALL.md：登录与验证流程改为 `auth login --api-key <key> --workspace-id <id>`；按 `docs/agents/install-doc-change.md` 同步线上 install 页。
- [ ] `docs/agents/*`：更新 `config-add.md`、`url-change.md`、`auth-change.md` 中关于 base_url 与 REGIONS 的描述；在 AGENTS.md「关键文件」中加入 `core/src/config/model-endpoint.ts` 与 `core/src/client/public-endpoints.ts`。
- [ ] 新增场景文档（可选）：`docs/agents/model-endpoint-change.md`，记录「新增 workspace 服务 / 新地域 / 公网例外下线」的清单，并登记到 AGENTS.md 场景索引。
- [ ] CHANGELOG.md / CHANGELOG.zh.md：按 `docs/agents/changelog-write.md` 写 Breaking Changes 与迁移指南（`config migrate` 用法、新旧优先级对照）。
- [ ] 版本号：七包锁步，按 D10 选 2.9.0 或 3.0.0；`tools/release/lib/packages.mjs` 无需改动。
- [ ] `pnpm ready`（`vp check` + 全量测试 + 构建）通过。
- [ ] 按 `docs/agents/public-content-review.md` 检查：文档与测试中不出现真实 workspace ID（示例一律用 `llm-xxx`）。

## 待取舍汇总

共 14 个待取舍点（编号 S4、S13 未使用）；标 ★ 的 6 项会改变接口或用户可见行为，建议在编码前定下；其余可按推荐先实现。

| # | 问题 | 推荐 | 备选 | 关联方案文档 | 位置 |
| --- | --- | --- | --- | --- | --- |
| S1 | region 校验强度 | A 宽松正则 | B 白名单 | D7 | Step 1 |
| S2 | logout 是否清 workspace / region | A 不清 | B 清 | — | Step 2 |
| ★ S3 | `--workspace-id` / `--region` 放哪一层 | A 进 `MODEL_AUTH_FLAGS` | B `--model-region`；C 进 GLOBAL | — | Step 2 |
| S5 | `Client.url` 第二参数 | A 直接删 | B 保留忽略 | — | Step 3 |
| ★ S6 | 来源冲突规则 | B 按层级 | A 严格 | — | Step 3 |
| ★ S7 | 运行时遇到 dashscope | 2.9 用 A 兼容，3.0 用 C 报错 | B 忽略 | D6、D10 | Step 9 |
| S8 | region 缺省是否用 console_region | A 固定 cn-beijing | B 先用 console_region | — | Step 3 |
| S9 | `ApiKeyCredential.baseUrl` 去留 | A 保留 | B 删除 | — | Step 3 |
| S10 | 74 处 workspace 端点调用怎么改 | A 改相对路径 + codemod | B 传 client；C 加 region 参数 | — | Step 4 |
| ★ S11 | 公网例外 API 的地域 | A 固定北京 | B 按地域映射；C 走 workspace | D1 | Step 5 |
| ★ S12 | 登录时 workspace 来源 | A 显式提供，后续加 B | B 控制台列表；C Key 反查 | D3 | Step 7 |
| S14 | `--prime` 去留 | A 保留为无作用 | B 删除；C 新含义 | D8 | Step 8 |
| ★ S15 | 配置文件何时改写 | A `config migrate` + C 登录顺带 | B 读时自动改 | D6 | Step 9 |
| S16 | dsh 插件是否本次改 | B 不改 | A 一起改 | — | Step 10 |

附：本文中仍「拿不准」、需要先验证或问后端的实现细节：

- `WORKSPACE_SERVICE_REGIONS` 的取值（Step 4.2）— 全部是占位，等 P0 矩阵。
- 浏览器登录回调的 endpoint 是否仍是 dashscope（Step 7.4）。
- `authStore.login()` 对 `undefined` 的语义，是否需要新增删除语义（Step 7.3）。
- `auth login` 会不会自动带上 `MODEL_AUTH_FLAGS`（Step 7.1）。
- Prime 在后端是否只是「走 workspace 域名」（S14）。
- `/apps/anthropic` 在 workspace 域名是否存在（Step 8.1）。
