# PR #349 审查：AEM 遥测按国内 / 国际站分流

> 日期：2026-11-25 · 审查对象：`modelstudioai/cli` PR #349（head `eeffcc5`，1 个提交，分支 `feat/telemetry-region-endpoint`）· 基线：拉分支时的 main `a382549`；当前 main `cd88751`（release 2.12.0）· 状态：讨论稿

## 结论与范围

结论：方向和主体实现没问题。合并前要做两件事：

1. 先解决与当前 main 的合并冲突（共 5 个文件，见[合并冲突与解决方案](#合并冲突与解决方案)）。解冲突时还要决定 `first_install` 事件报到哪个站。
2. 修复 P1。不修的话，`siteReason` 的漏判统计会失真，而文档里写的监控口径正是用这个统计来衡量漏判。

| # | 级别 | 问题 | 位置 | 建议 |
| --- | --- | --- | --- | --- |
| C1 | 阻塞 | 与当前 main 有 5 个文件的内容冲突（首装统计、`--biz-from` 改了同样的代码） | `telemetry/{sink,tracker,event,index}.ts`、`docs/agents/telemetry-change.md` | 按下文逐文件合并 |
| C2 | 阻塞（解冲突时引入） | `remoteSink` 的入参变成了 `TrackingEvent \| FirstInstallEvent`，`event.site` 类型检查不通过；`first_install` 的站点规则没有定义 | `telemetry/sink.ts` `remoteSink` | `first_install` 显式报国内站（方案 A），或在安装阶段按配置判定（方案 B） |
| C3 | 阻塞（解冲突时引入） | main 的 `reportFirstInstall` 依赖 `remoteSink` 返回 `boolean` 来记录上报次数；PR 版本返回 `void` | `telemetry/sink.ts` | 合并后保留 `Promise<boolean>`，`initRemoteForSite` 失败时返回 `false` |
| P1 | 需要处理（高） | 没配置 baseUrl 时会用默认值 `https://dashscope.aliyuncs.com`，结果被记成 `positive-domestic`。`openapi`、`none`、未配置的 `apiKey` 命令都属于这种情况，兜底事件被统计成"确定国内" | `telemetry/site.ts` `classifyFromModelOrigin`、`runtime/src/middleware.ts` `telemetryStage` | 把 `baseUrlIsDefault` 传进判定；默认值且没有 `lastResolvedOrigin` 时返回 `default-fallback` |
| M1 | 小问题 | `auth: "console"` 且 `consoleSite === "domestic"` 是显式配置，却返回 `default-fallback`，和未配置分不开 | `telemetry/site.ts` `resolveTelemetrySite` | 显式 `domestic` 返回 `positive-domestic` |
| M2 | 小问题 | 原来的 `remoteInitialized` 只初始化一次；现在 `initRemoteForSite` 失败后不缓存，每条事件都会重跑 `ensureUid` 和 `new Tracker` | `telemetry/sink.ts` `initRemoteForSite` | 记录失败的站点，或在注释里说明 |
| M3 | 小问题 | `lastResolvedOrigin` 在 `url()` 拼 URL 时就记录，不代表真的发了请求（dry-run 也会记录） | `core/src/client/client.ts` `url()` | 在 getter 注释里写明语义；目前只有 `video generate` 传了 `defaultBaseUrl`，影响有限 |
| M4 | 小问题 | `hostnameOf` 处理不带协议头的 host 时会保留端口，`dashscope.aliyuncs.com:443` 会被判成 `unknown` | `config/model-endpoints.ts` `hostnameOf` | 补上协议头后统一用 `new URL` 解析 |
| M5 | 小问题 | `ApiKeyEndpointKind` 在 core 里以 `ModelEndpointKind` 名字导出，`login-api-key.ts` 里还有一个本地同名类型；`INTERNATIONAL_WORKSPACE_REGIONS` 用 `.filter` 生成，丢了字面量类型 | `config/model-endpoints.ts`、`commands/src/commands/auth/login-api-key.ts:37` | 统一成一个类型；国际 region 列表直接写成字面量元组 |

审查方法：

- 克隆 `modelstudioai/cli`，`git fetch origin pull/349/head`，阅读全部 16 个文件的改动，并对照 `runtime/src/middleware.ts` 的 `authStage`、`core/src/client/client.ts`（`url()`、`uploadFile`）、vendored `lib/remote-telemetry/tracker.cjs` 核对。
- 在本地临时目录用 `git merge-tree --write-tree origin/main pr349` 和 `git merge --no-commit` 试合，确认冲突文件和每一处冲突块（试合后已 `merge --abort`，没有推送）。
- **没有运行测试**：安装依赖的那一步被本次会话的权限策略拦下了。以上结论都来自读代码。
- 没有在 PR 上发评论，也没有修改 `modelstudioai/cli` 仓库。

## PR 改了什么

- 新增 `core/src/config/model-endpoints.ts`：把官方模型 host、workspace region 收到一处，提供 `classifyModelHost()`（返回 domestic / international / unknown，`cn-hongkong` 算国际）。`auth login --api-key` 的候选 URL 也改用这里的 `KNOWN_MODEL_BASE_URLS`。
- 新增 `core/src/telemetry/site.ts`：
  - `resolveTelemetrySite()`：`console` 看 `consoleSite`；`apiKey` / `openapi` 优先看 `lastResolvedOrigin`，否则看 `client.baseUrl`；`none` 取两者的 OR。
  - `telemetryRemoteTarget()`：国内 `gm.mmstat.com` + `bailian-cli-node`，国际 `sg.mmstat.com` + `bailian-cli-node-intl`。
- `Client` 新增 `lastResolvedOrigin`，在 `url()`（含 `uploadFile` 内部调用的 `url("", defaultBaseUrl)`）里记录。
- `sink.ts` 从单例 Tracker 改成按站点存在 `Map` 里，同一进程可以同时有国内和国际两个 Tracker；`flushTelemetry` 会遍历所有 Tracker 调 `_sendAll`。
- `tracker.ts` 在 `finally` 里懒调用 `deps.resolveSite()`，失败时退回 `domestic + default-fallback`；事件新增 `site`、`siteReason`，同时写入本地 jsonl 和 AEM `ext`。
- `telemetryStage` 传入 `resolveSite` 闭包。闭包在 `finally` 时才读 `ctx.client`，能拿到 `authStage` 替换后的 Client。

核对通过的点：

- vendored Tracker 把 `endpoint` 当 host 用（`"https://" + endpoint + "/aes.1.1"`），所以传 `sg.mmstat.com` 就能得到正确的国际地址，不需要 patch。
- `uploadFile` 和 `resolveImageInput` 都会经过 `this.url(...)`，PR 描述里"upload 路径也记录 origin"成立。
- 站点判定有自己的 try/catch，不会盖掉命令本身的错误，也不会阻塞上报。
- `uid` / `device_id` 两站共用，与文档描述一致。

## 合并冲突与解决方案

### 冲突原因

PR 基于 `a382549`。之后 main 合入了 50 个提交，其中 5 个改了遥测：

| 提交 | 内容 |
| --- | --- |
| `1010b21f` | feat(install): add first-install statistics（新增 `first_install` 事件、`reportFirstInstallNow`） |
| `e1ad22ad` | feat(telemetry): add global `--biz-source` |
| `04966499` | refactor(telemetry): rename global source flag to `--biz-from`（AEM `c6`） |
| `2d94fa9a` | merge main into `feat/first-install-statistics` |
| `2f4f4a77` | fix(install): honor telemetry and proxy settings |

`git merge-tree --write-tree origin/main pr349` 的结果：

```text
CONFLICT (content): Merge conflict in docs/agents/telemetry-change.md
CONFLICT (content): Merge conflict in packages/core/src/telemetry/event.ts
CONFLICT (content): Merge conflict in packages/core/src/telemetry/index.ts
CONFLICT (content): Merge conflict in packages/core/src/telemetry/sink.ts
CONFLICT (content): Merge conflict in packages/core/src/telemetry/tracker.ts
Auto-merging packages/core/tests/telemetry-tracker.test.ts   # 自动合并成功
```

文本冲突都是"两边在同一位置各加了字段 / 逻辑"，原则上**两边都保留**。真正需要决定的只有 `sink.ts` 的 `remoteSink`（C2、C3）。

### 操作步骤（PR 作者执行）

在 PR 分支上合入最新 main：

```bash
git fetch origin main
```

```bash
git checkout feat/telemetry-region-endpoint
```

```bash
git merge origin/main
```

也可以用 `git rebase origin/main`。PR 只有一个提交，两种方式解冲突的工作量一样；rebase 后需要 force push 自己的 PR 分支。

按下面逐文件解决，然后：

```bash
vp check
```

```bash
vp test packages/core/tests/telemetry-uid.test.ts packages/core/tests/telemetry-sink.test.ts packages/core/tests/telemetry-tracker.test.ts packages/core/tests/telemetry-send.test.ts packages/core/tests/telemetry-site.test.ts packages/core/tests/client-last-resolved-origin.test.ts packages/core/tests/install-record.test.ts packages/runtime/tests/telemetry-stage.test.ts packages/cli/tests/postinstall-install-record.test.ts packages/commands/tests/e2e/auth.e2e.test.ts
```

这条命令在 PR 文档原有的测试集合上，加了 main 新增的 `install-record`、`postinstall-install-record` 两组首装测试。

### `packages/core/src/telemetry/event.ts`

三处冲突，都是两边各加字段，全部保留。

`TrackingEvent`：

```ts
  skillVersion?: string;
  /** Channel from install.json. The install day is a separate event. */
  installChannel?: InstallAttribution["installChannel"];
  /** Caller-declared call source (--biz-from), normalized; reported as AEM c6. */
  bizFrom?: string;
  /** AEM reporting site (domestic vs international). */
  site: TelemetrySite;
  /** Coarse site resolution reason; never includes raw hosts. */
  siteReason?: TelemetrySiteReason;
}
```

`createTrackingEvent` 的 `opts`：

```ts
  caller: CallerAttribution;
  /** Already-read install record. Callers must not create the file to fill this in. */
  install?: InstallAttribution;
  bizFrom?: string;
  site?: TelemetrySite;
  siteReason?: TelemetrySiteReason;
}): TrackingEvent {
```

函数体末尾：

```ts
  if (opts.install) {
    event.installChannel = opts.install.installChannel;
  }

  if (opts.siteReason) {
    event.siteReason = opts.siteReason;
  }

  return event;
```

两边的 import（main 的 `InstallAttribution`、PR 的 `TelemetrySite` / `TelemetrySiteReason`）已经自动合并，检查一下两条都在就行。

如果选方案 B（见 `sink.ts` 一节），`FirstInstallEvent` 和 `buildFirstInstallAemOptions` 还要加 `site` / `siteReason`；方案 A 不用改。

### `packages/core/src/telemetry/index.ts`

两边都改了导出行，合并成：

```ts
export { localSink, remoteSink, flushTelemetry, resetRemoteTelemetryForTests } from "./sink.ts";
export { reportFirstInstallNow, trackCommandExecution } from "./tracker.ts";
export { FIRST_INSTALL_REPORT_ARGUMENT, isFirstInstallReportLaunch } from "./first-install.ts";
```

PR 新增的 `resolveTelemetrySite` 等导出块不在冲突区，保持不变。

### `packages/core/src/telemetry/tracker.ts`

**import 冲突**：取两边并集。

```ts
import { normalizeBizFrom, type CallerAttribution } from "./caller.ts";
import { createFirstInstallEvent, createTrackingEvent } from "./event.ts";
import {
  FIRST_INSTALL_REPORT_ATTEMPTS,
  noteFirstInstallReportQueued,
  readFirstInstallReportAttempts,
} from "./first-install.ts";
import { flushTelemetry, localSink, remoteSink } from "./sink.ts";
import type { TelemetrySiteResolution } from "./site.ts";
```

**`finally` 开头**：两段逻辑互不依赖，都保留。

```ts
  } finally {
    const durationMs = Math.round(performance.now() - start);
    const install = readCommandInstallAttribution(deps.identity);
    let siteResolution: TelemetrySiteResolution = {
      site: "domestic",
      siteReason: "default-fallback",
    };
    try {
      if (deps.resolveSite) siteResolution = deps.resolveSite();
    } catch {
      // Site resolution must never mask command errors or block telemetry.
    }
```

**`createTrackingEvent` 参数**：

```ts
      caller: deps.caller,
      install,
      bizFrom: normalizeBizFrom(deps.settings.bizFrom),
      site: siteResolution.site,
      siteReason: siteResolution.siteReason,
    });
```

`TrackingDeps` 里 PR 新增的 `resolveSite` 不在冲突区，已自动合入。

### `packages/core/src/telemetry/sink.ts`（需要决策）

冲突块：

```ts
<<<<<<< HEAD (main)
    initRemote();
    if (!remoteSendEvent) return false;
    if (isFirstInstallEvent(event)) {
      remoteSendEvent(FIRST_INSTALL_EVENT_NAME, buildFirstInstallAemOptions(event));
    } else {
      remoteSendEvent(event.command, buildRemoteAemOptions(event));
    }
    return true;
=======
    const site: TelemetrySite = event.site ?? "domestic";
    const entry = initRemoteForSite(site);
    if (!entry) return;
    entry.sendEvent(event.command, buildRemoteAemOptions(event));
>>>>>>> pr349
```

不能直接选任何一边：

- 选 main：PR 的 `initRemote` / `remoteSendEvent` 已经删掉了，编译不过，分流也就没了。
- 选 PR：`event` 现在是联合类型，`FirstInstallEvent` 上没有 `site`，类型检查报错（C2）；`return;` 和 `Promise<boolean>` 不兼容，`reportFirstInstall` 永远记不到"已上报"，每次安装都会重试到次数上限（C3）；`first_install` 还会被当成命令事件，用 `event.command`（`undefined`）作事件名发出去。

**关于 `first_install` 报到哪个站**：首装事件在 `reportFirstInstallNow(identity)` 里发出，这时还没有命令、没有 auth、没有 Client，只能读到 `buildSettings(buildSources({}))` 得到的配置。新用户首次安装时，配置基本是空的。

- **方案 A（推荐，本 PR 采用）**：`first_install` 固定报国内站。新用户没有任何站点信号，按配置判定的结果也是国内兜底；规则简单，而且和 PR 文档"best-effort、未知走国内"的口径一致。在 `telemetry-change.md` 的"已知残留"里加一条"`first_install` 固定国内站"。
- **方案 B（后续可选）**：在 `reportFirstInstallNow` 里用读到的 settings 调 `resolveTelemetrySite({ auth: "none", consoleSite, baseUrl, baseUrlIsDefault })`，把结果写进 `FirstInstallEvent.site` / `siteReason`，`buildFirstInstallAemOptions` 的 `ext` 也带上这两个字段。只对"已有国际站配置的机器重装"这种情况有用；要改的地方更多，可以单独开 PR。

方案 A 的合并结果：

```ts
/** Returns whether the event was queued. Callers must not await before this function enqueues. */
export async function remoteSink(event: TrackingEvent | FirstInstallEvent): Promise<boolean> {
  try {
    // first_install has no command/auth context; it always reports domestic (best-effort).
    const site: TelemetrySite = isFirstInstallEvent(event) ? "domestic" : event.site ?? "domestic";
    const entry = initRemoteForSite(site);
    if (!entry) return false;
    if (isFirstInstallEvent(event)) {
      entry.sendEvent(FIRST_INSTALL_EVENT_NAME, buildFirstInstallAemOptions(event));
    } else {
      entry.sendEvent(event.command, buildRemoteAemOptions(event));
    }
    return true;
  } catch {
    // Telemetry must never affect the CLI main flow
    return false;
  }
}
```

要点：

- 保留 main 的返回值约定：入队成功返回 `true`，Tracker 构造失败或抛错返回 `false`。
- 保留 main 的"同步入队"约定（注释 "Callers must not await before this function enqueues"）：`initRemoteForSite` 是同步的，满足要求。
- `sink.ts` 顶部 import 已自动合并：`isFirstInstallEvent`、`FIRST_INSTALL_EVENT_NAME`、`buildFirstInstallAemOptions`、`FirstInstallEvent`（来自 main）和 `telemetryRemoteTarget`、`TelemetrySite`（来自 PR）都要在。
- `localSink` 的签名改动（main）不在冲突区，已自动合入。

### 测试的连带改动

- main 的 `telemetry-sink.test.ts` 有首装相关用例。原来它依赖单例 Tracker 在用例间的状态；改成按站点的 `Map` 后，用例间需要调用 PR 新增的 `resetRemoteTelemetryForTests()`，否则 Tracker 构造次数的断言可能串到别的用例。
- 建议补一条：`remoteSink(createFirstInstallEvent(...))` 只构造国内 Tracker（`pid=bailian-cli-node`、`endpoint=gm.mmstat.com`），事件名是 `first_install`，返回 `true`。
- 建议补一条：Tracker 构造抛错时，`remoteSink` 返回 `false`，`reportFirstInstall` 不调用 `noteFirstInstallReportQueued`。

### `docs/agents/telemetry-change.md`

冲突在"AEM 映射"表：main 在 `ext` 行加了 `installChannel`，并新增 `c6` 行（`bizFrom`）；PR 在 `ext` 行加了 `site` / `siteReason`，并在表后加了 `siteReason` 的说明段落。合并成：

```md
| AEM 字段   | 内容 |
| ---------- | ---- |
| event name | 命令路径 |
| `et`       | `EXP` |
| `ext`      | 除 `command`、`params` 外的结构化事件字段，含 `commandPrefix`、`callerType`、`site`、可选 `siteReason`、失败时的 `exitCode`，以及读到的 `installChannel`。安装日不在命令事件里（main 原文的后半句保留） |
| `c1`       | allowlist 参数 |
| `c2`       | `success` / `failure` |
| `c3`       | HTTP status |
| `c4`       | 错误文案，最多 500 字符 |
| `c5`       | request ID |
| `c6`       | `bizFrom` 业务来源（全局 `--biz-from`，归一化标量；专用扁平列，直接筛选/分组） |

`siteReason` 为粗粒度枚举……（PR 的说明段落原样保留）
```

另外：

- 在"国内 / 国际 AEM 分流"一节的"已知残留"里加一条："`first_install` 固定报国内站（安装阶段没有 auth / Client 上下文）"。
- main 若在文档里给首装事件写了 "pid=bailian-cli-node" 之类的描述，保持国内 pid，与方案 A 一致。
- 表格列宽交给格式化工具（`vp check` / prettier）重新对齐，不用手工对齐。

## P1：默认 baseUrl 被记成 `positive-domestic`

### 问题

`authStage` 的 `base` 一定会展开 `resolveModelBaseUrlState(sources)`：

```ts
const base = {
  identity: ctx.identity,
  settings,
  ...
  ...resolveModelBaseUrlState(sources),   // { baseUrl, baseUrlIsDefault }
};
```

没配置时，`baseUrl` 是 `REGIONS.cn = "https://dashscope.aliyuncs.com"`，`baseUrlIsDefault = true`。`telemetryStage` 只传 `ctx.client.baseUrl`，不传 `baseUrlIsDefault`，于是 `classifyFromModelOrigin` 总能拿到一个国内 host，返回 `positive-domestic`。

受影响的场景：

| 命令 | 结果 | 应为 |
| --- | --- | --- |
| `auth: "openapi"`（不走模型域，`baseUrl` 基本都是默认值） | `domestic / positive-domestic`，国际站 AK 用户全部被当成"确定国内" | `domestic / default-fallback` |
| `auth: "none"` 且没有 `consoleSite`（如首次 `auth login`） | `domestic / positive-domestic` | `domestic / default-fallback` |
| `auth: "apiKey"`、没配 baseUrl、没走 `defaultBaseUrl` 回调 | `domestic / positive-domestic`（落国内是对的，但这是兜底） | `domestic / default-fallback` |

PR 文档写的监控口径是"漏判量化用国内站 `siteReason` 兜底类占比"。上面这些事件被标成 positive，兜底占比会被系统性低估，指标就失去了意义。

### 复现（单测）

```ts
// packages/core/tests/telemetry-site.test.ts
test("openapi with default model baseUrl is a fallback, not a positive domestic", () => {
  expect(
    resolveTelemetrySite({
      auth: "openapi",
      baseUrl: "https://dashscope.aliyuncs.com",
      baseUrlIsDefault: true,
    }),
  ).toEqual({ site: "domestic", siteReason: "default-fallback" });
});
```

按当前 PR 代码，这条会得到 `positive-domestic`（`baseUrlIsDefault` 字段还不存在，需要先按下面的方案加上）。

### 修复方案

`core/src/client/client.ts`：暴露 `baseUrlIsDefault`。

```ts
  /** True when the model base URL came from the built-in default (nothing configured). */
  get baseUrlIsDefault(): boolean {
    return this.deps.baseUrlIsDefault === true;
  }
```

`ClientDeps` 的注释写着"有 apiCred 时两者一致"，也就是 `apiCred.baseUrl` 与 `deps.baseUrl` 来自同一条解析链，所以只看 `deps.baseUrlIsDefault` 就够了。

`core/src/telemetry/site.ts`：

```ts
export interface ResolveTelemetrySiteInput {
  auth: AuthRequirement;
  consoleSite?: "domestic" | "international";
  lastResolvedOrigin?: string;
  baseUrl?: string;
  /** baseUrl is the built-in default, not a user signal. */
  baseUrlIsDefault?: boolean;
}

function classifyFromModelOrigin(input: ResolveTelemetrySiteInput): TelemetrySiteResolution {
  // lastResolvedOrigin is a real per-command choice (service default callback) and wins.
  const host =
    hostnameOf(input.lastResolvedOrigin) ??
    (input.baseUrlIsDefault ? undefined : hostnameOf(input.baseUrl));
  if (!host) {
    return { site: "domestic", siteReason: "default-fallback" };
  }
  // ...后面不变
}
```

`resolveTelemetrySite` 里的两处调用改成 `classifyFromModelOrigin(input)`。

`runtime/src/middleware.ts` 的 `telemetryStage`：

```ts
      resolveSite: () =>
        resolveTelemetrySite({
          auth: ctx.command.auth,
          consoleSite: ctx.settings.consoleSite,
          lastResolvedOrigin: ctx.client.lastResolvedOrigin,
          baseUrl: ctx.client.baseUrl,
          baseUrlIsDefault: ctx.client.baseUrlIsDefault,
        }),
```

补充测试：

- `openapi` + 默认 baseUrl → `default-fallback`。
- `openapi` + 用户配置的 `https://dashscope-intl.aliyuncs.com`（`baseUrlIsDefault: false`）→ `international / positive-international`。
- `apiKey` + 默认 baseUrl + `lastResolvedOrigin = https://dashscope-intl.aliyuncs.com` → `international`（`lastResolvedOrigin` 优先）。
- `telemetry-stage.test.ts`：stub client 的 `baseUrlIsDefault: true` 时，断言 `resolveSite()` 返回 `default-fallback`。

这条修复与冲突解决互不影响，可以在合并 main 之后单独提交。

## M1：console 显式国内站记成 `default-fallback`

```ts
  if (input.auth === "console") {
    if (input.consoleSite === "international") {
      return { site: "international", siteReason: "positive-international" };
    }
    if (input.consoleSite === "domestic") {
      return { site: "domestic", siteReason: "positive-domestic" };
    }
    return { site: "domestic", siteReason: "default-fallback" };
  }
```

`auth: "none"` 分支同理：`consoleSite === "domestic"` 且模型侧没有信号时，也可以返回 `positive-domestic`。`telemetry-site.test.ts` 第 37 行的 console 用例需要同步更新断言。

## M2：Tracker 初始化失败不再只尝试一次

原来：

```ts
function initRemote(): void {
  if (remoteInitialized) return;
  remoteInitialized = true;
  ...
}
```

现在 `initRemoteForSite` 抛错后不写 `Map`，下一条事件会重新执行 `ensureUid()`（可能写盘）和 `new Tracker()`。单进程通常只有一条命令事件，加上 `first_install` 最多两次，影响很小。建议：

```ts
const failedSites = new Set<TelemetrySite>();

function initRemoteForSite(site: TelemetrySite): RemoteSiteClient | undefined {
  const existing = remoteBySite.get(site);
  if (existing) return existing;
  if (failedSites.has(site)) return undefined;
  try {
    ...
  } catch {
    failedSites.add(site);
    return undefined;
  }
}
```

`resetRemoteTelemetryForTests()` 里同时 `failedSites.clear()`。

## M3：`lastResolvedOrigin` 的语义

`url()` 只要被调用就会记录，dry-run 或只拼 URL 不发请求的路径也算。目前只有 `commands/src/commands/video/generate.ts:299` 传了 `defaultBaseUrl`，实际影响很小。建议把 getter 注释改成：

```ts
  /**
   * Origin last chosen by {@link url} (including upload paths). It records URL
   * resolution, not a completed request; dry-run also records it. Telemetry only.
   */
```

## M4：`hostnameOf` 保留端口

```ts
export function hostnameOf(input: string | undefined): string | undefined {
  const trimmed = input?.trim();
  if (!trimmed) return undefined;
  try {
    const withScheme = /^https?:\/\//i.test(trimmed) ? trimmed : `https://${trimmed}`;
    return new URL(withScheme).hostname.toLowerCase() || undefined;
  } catch {
    return undefined;
  }
}
```

补一条 `hostnameOf("dashscope.aliyuncs.com:443") === "dashscope.aliyuncs.com"` 的测试。

## M5：类型统一

- 删除 `login-api-key.ts:37` 的本地 `type ApiKeyEndpointKind = Exclude<ApiKeyLoginKind, "unknown">`，改为从 core 导入 `ModelEndpointKind`。或者反过来，core 里定义成 `Exclude<ApiKeyLoginKind, "unknown">`，保证与 `ApiKeyLoginKind` 不会漂移。
- `INTERNATIONAL_WORKSPACE_REGIONS` 改成字面量元组，并加一个编译期检查，保证国内加国际正好覆盖 `ORDINARY_WORKSPACE_REGIONS`：

```ts
export const INTERNATIONAL_WORKSPACE_REGIONS = [
  "ap-southeast-1",
  "cn-hongkong",
  "ap-northeast-1",
  "eu-central-1",
  "us-east-1",
] as const;

type _RegionsCovered =
  Exclude<OrdinaryWorkspaceRegion, DomesticWorkspaceRegion | (typeof INTERNATIONAL_WORKSPACE_REGIONS)[number]> extends never
    ? true
    : never;
const _regionsCovered: _RegionsCovered = true;
```

## 已知限制（PR 文档已列出，同意暂不处理）

- `auth login` 那条事件读的是登录前的 settings，首次登录国际站的那条事件会记到国内。
- 反代或自定义 host 一律落国内（`unknown-host`）。
- managed-agent 在外部拼 host，不经过 `Client.url()`。
- `ext` 里新增了 `site` / `siteReason`，AEM 看板需要确认字段已经配好；国际 pid `bailian-cli-node-intl` 需要在 AEM 侧提前建好。

## 建议的合并顺序

1. PR 作者在分支上合入 main，按[合并冲突与解决方案](#合并冲突与解决方案)解冲突（`first_install` 用方案 A），跑 `vp check` 和上面的测试集合。
2. 同一个 PR 里追加 P1 的修复和测试；M1 改动很小，可以一起做。
3. M2 到 M5 可以放在这个 PR，也可以放到后续 PR。
4. 合并后观察国际 pid 的事件量，以及国内 pid 里 `default-fallback` / `unknown-host` 的占比。
