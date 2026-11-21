# PR #306 审查：成功路径主动退出（Issue #294 进程滞留约 50 秒）

> 日期：2026-11-21 · 审查对象：`modelstudioai/cli` PR #306（head `38e4d91f`，1 个提交，基于 `2c29049`；当前 main 为 `50e66d2`，v2.10.4）· 关联 Issue #294 · 状态：讨论稿

## 结论与范围

结论：方向正确，但**不建议原样合并**。PR 能消除 Windows release 二进制退出前约 50 秒的滞留，但在 Bun 下用来"刷新输出"的写法不起作用。stdout 是管道时，超过 64 KB 的输出会被截断，退出码却是 0。这正好命中 Issue 描述的使用场景：别的程序 spawn `bl`，从管道读 JSON。此外，Issue 里还有两件事 PR 没有处理。

| # | 级别 | 问题 | 来源 | 建议 |
| --- | --- | --- | --- | --- |
| P1 | **阻塞** | Bun 下 `write("", cb)` 不等待之前的数据，`process.exit` 截断管道输出（实测 64 KB / 128 KB） | 本 PR 引入 | 跟踪每次真实写入的回调，全部完成后再退出，并加超时兜底 |
| P2 | 需要处理（中） | 更新检查失败时不写 `lastChecked`，每次运行都额外等 3～6 秒 | 历史遗留，Issue 中 npm 版 4.3 秒的主要来源 | 失败也记录检查时间（短退避），缩短超时 |
| P3 | 需要处理（中） | 更新检查写死 `registry.npmjs.org`，忽略 `npm_config_registry` / `.npmrc` | 历史遗留，Issue 明确提出 | 按 npm 规则解析 registry；PR 不应写 `Fixes #294` 关闭整个 Issue |
| S1 | 建议 | 刷新没有超时，回调不触发时进程会永久挂起 | 本 PR 引入 | 与 P1 一起：`Promise.race` 加超时 |
| S2 | 建议 | 单元测试 mock 了 `process.exit`，e2e 跑在 Node + tsx 下，Bun 上的行为没有覆盖 | 本 PR 引入 | 在现有 `http-bun.e2e.test.ts` 的编译二进制框架里加子进程测试 |
| S3 | 建议 | 根因（Bun `fetch` 连接池持有 keep-alive socket）未处理，只靠强制退出掩盖 | 本 PR 未覆盖 | 可选：Bun 分支下关闭连接复用，作为第二层保障（需 Windows 验证） |
| S4 | 建议 | stdout 已被 destroy 时，`write("")` 触发 `ERR_STREAM_DESTROYED`，`handleStdoutPipeError` 会把它重新抛出 | 本 PR 引入（极少触发） | P1 方案不再做空写入，问题随之消失 |
| S5 | 提示 | Bun 中一旦访问过 `process.stdout`，`console.log` 也变成异步且没有回调，强制退出会截断它 | 平台行为 | 约束：输出只走 `process.stdout.write`；加 lint 规则禁止 `console.log` |
| S6 | 提示 | 出错路径 `handleError` 写完 stderr 立即 `process.exit`，Bun 下理论上同样会截断大于 64 KB 的 stderr | 历史遗留 | 出错路径也改走 P1 的 `flushOutput()` |

审查方法与边界：

- 读了 Issue、PR 描述和 diff，并阅读了上下游代码：`create-cli.ts`、`middleware.ts`、`error-handler.ts`、`output/reporter.ts`、`utils/update-checker.ts`、`core/client/transport.ts`，以及所有长时间运行的命令（`config ui`、`auth login --console`、managed-agent workbench）。
- 在 Linux 容器中用 **Node 22.23**、**Bun 1.3.14**（与 CI 和 publish 工作流固定的版本一致）和 Bun 1.4.4 写了最小脚本做实测。P1 的截断现象、P1 的修复原型和 S5 都已实测。
- **没有在 Windows 上测试**。在 Linux 上（本地 HTTP keep-alive 和经代理的 HTTPS 都试过），Bun 和 Node 都没有复现 50 秒滞留，所以 PR 对 Windows 滞留的实际效果没有亲眼确认。
- 没有安装依赖，也没有跑仓库自带的测试。没有修改 `modelstudioai/cli` 的任何内容，也没有发表评论。下文代码是建议写法，其中 P1 原型已用独立脚本验证，并入仓库后需要编译和测试确认。

## 背景

### Issue #294

- 环境：CLI 2.7.2，Windows 11，PowerShell 7，Node v26.13，region cn。每次都能复现。
- 命令：`bl usage token-plan --output json`，退出码 0，JSON 正确。
- 现象：JSON 约 0.8～1.2 秒输出，进程约 50～56 秒后才退出。
- 对比：release 二进制约 54.7 秒退出；npm 版约 4.3 秒；npm 版加 `NO_UPDATE_NOTIFIER=1` 约 0.9 秒。
- reporter 的判断：release 二进制（Bun 编译）的 `globalThis.fetch` 保持着连接池里的 keep-alive socket；npm 版的 undici 连接池约 4 秒超时。建议在 `run()` 结束后主动退出。
- 附带请求：更新检查写死了 `https://registry.npmjs.org`，忽略 `.npmrc` 和 `npm_config_registry`。

### PR #306 的改动

5 个文件，+86 / −7：

- `packages/runtime/src/create-cli.ts`：新增 `runToExit(cli, argv?)`。它先 `await cli.run(argv)`，再对 stdout 和 stderr 各调用一次 `stream.write("", cb)` 并等待回调，最后 `process.exit(process.exitCode ?? 0)`。
- `packages/cli/src/main.ts`、`packages/kscli/src/main.ts`：入口从 `void createCli(...).run()` 改为 `void runToExit(createCli(...))`。
- `packages/runtime/src/index.ts`：导出 `runToExit`。
- `packages/runtime/tests/run-to-exit.test.ts`：3 个单元测试，mock 了 `process.exit` 和 `run`。

### 已确认没问题的部分

- **出错路径**：`run()` 内部捕获所有错误，交给 `handleError` → `process.exit`，行为不变。
- **后台任务**：更新检查（`versionCheckStage` 里 `await pending`）和 telemetry（`await flushTelemetry(1000)`）都在 `run()` 内部等待完成，强制退出不会丢掉它们。
- **长时间运行的命令**都在 `run()` 内部等到结束，提前退出不会打断它们：
  - `config ui`：`await new Promise(... server.once("close"))`（`commands/config/ui.ts:801`）。
  - `auth login --console`：`await completion`，`close` 事件在所有连接结束后才触发，所以浏览器能收到 `OK` 响应（`commands/auth/login-console.ts:582`）。
  - managed-agent workbench：`await waitForChildExit(child)`（`managed-agent/_engine/playground-launcher.ts:144`）。
  - 浏览器打开（`execFile`）和 detached 子进程（`.unref()`）在父进程退出后照常运行。
- **有一处行为变化，可以接受**：`config ui` 里发起的控制台登录（`auth/console-ui.ts:73`）不会被 await。以前 UI 关闭后进程会继续挂着，等这个登录完成（最长 15 分钟）；现在 UI 一关就退出。这更符合直觉，但建议在 PR 描述里写明。
- **与 main 的差异**：PR 基于 `2c29049`。之后 main 改过 `create-cli.ts` 和 `reporter.ts`（新增 recovery config 相关逻辑），但没有动退出路径，rebase 预计只有 import 行的小冲突。

## 需要处理

### P1 Bun 下 `write("", cb)` 不能保证数据写完，强制退出截断管道输出

**位置**：PR 新增的 `runToExit`（`packages/runtime/src/create-cli.ts`）：

```ts
const flush = (stream: NodeJS.WriteStream) =>
  new Promise<void>((done) => {
    stream.write("", () => done());
  });
await flush(process.stdout);
await flush(process.stderr);
process.exit(code);
```

**原理**：Node 的可写流按顺序处理写入队列，空写入的回调会排在之前所有写入之后，所以这个技巧在 Node 下成立。Bun 的 `process.stdout` 实现不同：空写入的回调在之前的数据真正写进管道之前就触发了。Bun 也不报告积压量（`writableLength` 恒为 0，`writableNeedDrain` 恒为 false），所以等 `drain` 事件也没用。只有**承载数据的那次 `write(data, cb)` 的回调**才能可靠地表示"这段数据写完了"。

**实测**（Linux x64，写 8,000,001 字节到 stdout）：

| 写法 | Node 22 管道 | Bun 1.3.14 管道 | Bun 1.3.14 `--compile` 管道 | Bun 1.4.4 管道 | Bun 重定向到文件 |
| --- | --- | --- | --- | --- | --- |
| 不调用 exit（PR 合并前的行为） | 完整 | 完整 | — | 完整 | 完整 |
| `write(data)` 后直接 `process.exit` | 完整 | — | — | 131,072 | — |
| **PR 写法**：`write("", cb)` 刷新后 exit | 完整 | **65,536** | **65,536** | **131,072** | 完整 |
| 等 `drain` 后 exit | 完整 | 65,536 | — | — | — |
| `write(data, cb)`，在 cb 里 exit | 完整 | 完整 | — | 完整 | — |
| **建议方案**：跟踪所有写入回调后 exit | 完整 | 完整 | 完整 | — | — |

**影响**：

- 截断只发生在 stdout 是**管道**时，终端和重定向到文件不受影响。所以手工在终端里测不出来，但凡是被别的程序 spawn 的场景（Agent、脚本、`| jq`）都会中招。这也正是 Issue 关心的场景。
- 阈值在 64 KB 左右（与 Linux 管道缓冲区大小一致）。大列表、`--introspect` 的命令树 schema、日志查询之类的 `--output json` 结果都可能超过。
- 被截断时退出码仍然是 0，调用方只会看到一个 JSON 解析错误，很难定位。
- 合并前不会截断，所以这是一个**新引入的回归**：用 50 秒的延迟换来了静默的数据丢失。

**解决方案**：在 runtime 里集中跟踪 stdout 和 stderr 的每一次写入回调，`runToExit` 等全部完成（并有超时兜底）后再退出。用包装 `stream.write` 的方式，而不是只改 reporter，是因为还有不少地方直接调用 `process.stdout.write` / `process.stderr.write`：`create-cli.ts` 里的 introspect 和 version，help 输出，登录 URL 提示等。

新增 `packages/runtime/src/output/flush.ts`：

```ts
/**
 * Track every stdout/stderr write so the process can exit only after the data
 * reached the OS. Bun does not order an empty write's callback after earlier
 * writes and does not report writableLength, so the callback of each real write
 * is the only reliable completion signal on both Node and Bun.
 */
const pending = new Set<Promise<void>>();
const tracked = new WeakSet<NodeJS.WriteStream>();

type WriteCallback = (error?: Error | null) => void;

export function trackStreamWrites(stream: NodeJS.WriteStream): void {
  if (tracked.has(stream)) return;
  tracked.add(stream);
  const original = stream.write.bind(stream) as (
    chunk: unknown,
    encoding?: BufferEncoding,
    callback?: WriteCallback,
  ) => boolean;
  stream.write = ((
    chunk: unknown,
    encodingOrCallback?: BufferEncoding | WriteCallback,
    maybeCallback?: WriteCallback,
  ) => {
    const callback = typeof encodingOrCallback === "function" ? encodingOrCallback : maybeCallback;
    const encoding = typeof encodingOrCallback === "function" ? undefined : encodingOrCallback;
    let settle!: () => void;
    const done = new Promise<void>((resolve) => (settle = resolve));
    pending.add(done);
    void done.then(() => pending.delete(done));
    return original(chunk, encoding, (error) => {
      settle(); // settle on error too: an EPIPE must not keep the process alive
      callback?.(error);
    });
  }) as typeof stream.write;
}

/** Wait until every tracked write has completed, or `timeoutMs` has passed. */
export async function flushOutput(timeoutMs = 5000): Promise<void> {
  if (pending.size === 0) return;
  let timer: ReturnType<typeof setTimeout> | undefined;
  await Promise.race([
    Promise.all([...pending]),
    new Promise<void>((resolve) => {
      timer = setTimeout(resolve, timeoutMs);
      timer.unref?.();
    }),
  ]);
  clearTimeout(timer);
}
```

`runToExit` 改为：

```ts
import { flushOutput, trackStreamWrites } from "./output/flush.ts";

export async function runToExit(cli: Cli, argv?: string[]): Promise<void> {
  // Install before run() so every write of the invocation is tracked.
  trackStreamWrites(process.stdout);
  trackStreamWrites(process.stderr);
  await cli.run(argv);
  await flushOutput();
  process.exit(process.exitCode ?? 0);
}
```

出错路径（S6）也改为先刷新再退出。`handleError` 目前是同步的、返回 `never`，可以把退出交回给 `run()` 的 catch 分支：

```ts
// create-cli.ts, run() 的 catch 分支
} catch (error) {
  for (const warning of warnings.splice(0)) reporter.output.warning(warning);
  await flushTelemetry(1000);
  const code = reporter.error(error); // 只渲染，不退出
  await flushOutput();
  process.exit(code);
}
```

如果不想改 `handleError` 的签名，可以只在 `runToExit` 这一层处理成功路径，出错路径留作后续，因为错误输出通常远小于 64 KB。

**原型验证**（Bun 1.3.14，Linux x64，stdout 为管道）：

| 场景 | 期望字节数 | 实测 |
| --- | --- | --- |
| 单次写 8 MB 后 `flushOutput()` + exit | 8,000,001 | 8,000,001 |
| 400 次各写 20,001 字节 | 8,000,400 | 8,000,400 |
| 同一脚本 `bun build --compile` 后的二进制 | 8,000,001 | 8,000,001 |
| Node 22 | 8,000,001 | 8,000,001 |
| 用 `console.log` 写 8 MB（见 S5） | 8,000,001 | **65,536** |

### P2 更新检查失败不记录时间，每次运行额外等 3～6 秒

**位置**：`packages/runtime/src/utils/update-checker.ts`

```ts
const FETCH_TIMEOUT_MS = 3000;
// checkForUpdate():
if (method === "binary") {
  latest = await fetchBinaryChannelVersion("latest", FETCH_TIMEOUT_MS); // 最多 3 秒
  ...
  if (!latest) latest = await fetchLatestVersion(FETCH_TIMEOUT_MS, npmPackage); // 再 3 秒
} else {
  latest = await fetchLatestVersion(FETCH_TIMEOUT_MS, npmPackage);
}
if (!latest) return;          // ← 失败时直接返回，不写 state
writeState({ lastChecked: now, latestVersion: latest });
```

`versionCheckStage`（`middleware.ts:295`）会在命令执行完后 `await pending`，所以这段时间算在 `run()` 内部，PR 的强制退出管不到。

**影响**：

- 在 cn 区域，访问 `registry.npmjs.org` 很慢或不通时，每次拉取都会超时失败，24 小时去抖永远不生效。npm 版每次运行多等约 3 秒，二进制版最多 6 秒（binary channel 和 npm 各 3 秒）。
- 这和 Issue 的数据吻合：npm 版 4.3 秒 ≈ 0.9 秒的命令本身 + 约 3 秒超时，加了 `NO_UPDATE_NOTIFIER=1` 就降到 0.9 秒。
- PR 合并后，二进制版从约 54 秒降到"命令耗时 + 最多 6 秒"，仍然不是 Issue 期望的"输出完立即退出"。

**解决方案**：

1. 失败时也记录本次检查，用较短的退避间隔，避免每次都重试：

   ```ts
   const FAILED_CHECK_RETRY_MS = 60 * 60 * 1000; // 失败后 1 小时内不再重试

   if (!latest) {
     // Keep the last known version; only push the next attempt out.
     writeState({
       lastChecked: now - CHECK_INTERVAL_MS + FAILED_CHECK_RETRY_MS,
       latestVersion: state?.latestVersion ?? currentVersion,
     });
     return;
   }
   ```

   这里借用 `lastChecked` 来表示下次重试的时间，不用改 state 的格式。如果愿意改格式，更清楚的写法是新增 `nextCheckAt` 字段。

2. 缩短超时：版本检查是锦上添花的功能，建议 `FETCH_TIMEOUT_MS` 降到 1000～1500 ms。二进制版的两次请求改为并发（`Promise.any`），或者共享同一个总超时预算，把最坏情况从 6 秒压到 1.5 秒左右。

3. 可选：对非交互调用（stdout 不是 TTY，或输出格式为 JSON）跳过更新检查，或者只读缓存、不发请求。Agent 和脚本场景本来就看不到更新提示（提示走 stderr），却要付出延迟成本。

### P3 registry 写死，忽略 npm 配置

**位置**：`update-checker.ts:15`，`export const NPM_REGISTRY = "https://registry.npmjs.org";`，`fetchLatestVersion` 直接拼接这个地址。

Issue 明确提出了这一点，PR 没有处理，但 PR 描述写了 "Fixes #294"，合并时会自动关闭整个 Issue，这部分请求就会悄悄丢掉。它和 P2 是相关的：国内用户本来可以配置镜像（如 `https://registry.npmmirror.com`）来避开超时。

**解决方案**：按 npm 的优先级解析 registry，并对 scoped 包支持 `@scope:registry`：

```ts
import { existsSync, readFileSync } from "node:fs";
import { homedir } from "node:os";
import { join } from "node:path";

const DEFAULT_REGISTRY = "https://registry.npmjs.org";

function readNpmrc(path: string): Record<string, string> {
  if (!existsSync(path)) return {};
  const out: Record<string, string> = {};
  for (const raw of readFileSync(path, "utf8").split(/\r?\n/)) {
    const line = raw.trim();
    if (!line || line.startsWith("#") || line.startsWith(";")) continue;
    const eq = line.indexOf("=");
    if (eq > 0) out[line.slice(0, eq).trim()] = line.slice(eq + 1).trim();
  }
  return out;
}

/** npm precedence: env > user ~/.npmrc (> global npmrc, optional). */
export function resolveNpmRegistry(npmPackage: string, env = process.env): string {
  const scope = npmPackage.startsWith("@") ? npmPackage.split("/")[0] : undefined;
  const userrc = readNpmrc(env.npm_config_userconfig || join(homedir(), ".npmrc"));
  const pick = (key: string) =>
    env[`npm_config_${key}`] ?? env[`NPM_CONFIG_${key.toUpperCase()}`] ?? userrc[key];
  const value = (scope && pick(`${scope}:registry`)) || pick("registry") || DEFAULT_REGISTRY;
  return value.replace(/\/+$/, "");
}
```

几点说明：

- 有意**不读项目级 `.npmrc`**（当前目录），因为 `bl` 是全局工具，不应该随工作目录改变更新源。
- `.npmrc` 里的值可能包含 `${ENV}` 插值，可以先不支持，遇到时回退到默认地址。
- 二进制版的 binary channel 是自有服务，不受影响。只有回退到 npm 的那一步用这个函数。
- 建议同时提供 `BAILIAN_UPDATE_REGISTRY` 环境变量作为显式覆盖，方便企业内网使用。
- `bl update` 实际执行 `npm install -g` 时，npm 自己会读配置，不需要改。

如果这部分不打算放进 #306，建议把 PR 描述改成 "Refs #294" 或 "Partially fixes #294"，并把 P2、P3 拆成独立 Issue。

## 建议

### S1 刷新没有超时

PR 的 `flush` 没有超时。如果某个运行时或某种流类型的写回调一直不触发（例如 stdout 被重定向到一个不读取的管道，对端也不关闭），进程会永久挂起，比原来的 50 秒还糟。P1 方案里的 `flushOutput(timeoutMs = 5000)` 已经包含兜底，计时器用了 `unref()`，不会自己撑住事件循环。

### S2 测试没有覆盖 Bun 和真实子进程

- PR 的 3 个单元测试都 mock 了 `process.exit` 和 `run`，只验证了调用顺序，验证不了"数据写完了"和"进程真的退出了"。
- 现有 e2e（`packages/e2e/src/run-subprocess.ts`）用 `node --import tsx` 跑 `main.ts`，不经过 Bun，测不出 P1。
- 仓库已经有一套跑在 Node、原生 Bun 和编译二进制上的测试框架：`packages/runtime/tests/http-bun.e2e.test.ts`。它会编译 probe 二进制，可以用 `BAILIAN_TEST_BUN` 指定 Bun。建议在这里加用例（见下文"测试方法"的 T3）。

### S3 可选：从根上避免 keep-alive socket 撑住进程

强制退出是兜底手段。根因是 Bun 的 `fetch` 连接池在请求结束后仍持有 socket。所有请求都经过 `packages/core/src/client/transport.ts` 的 `httpTransport.fetch`，可以在 Bun 分支统一关掉连接复用：

```ts
if (process.versions.bun) {
  // A CLI invocation makes few requests; connection reuse is not worth a socket
  // that keeps the process alive after the answer is printed (#294).
  return globalThis.fetch(input, { ...init, keepalive: false } as RequestInit);
}
```

注意事项：

- Bun 的 `fetch` 用 `keepalive: false` 关闭连接复用（Bun 的扩展语义；在浏览器语义里 `keepalive` 表示另一件事）。这一点需要在 Windows 上确认确实消除了滞留，我在 Linux 上复现不了滞留，所以没法验证。
- `packages/core/src/telemetry/sink.ts` 依赖 `keepalive: true`，telemetry 不走这个分支时不受影响；如果走，要单独保留。
- 关掉复用后，一次调用里有多个请求时（分页列表、轮询）会多几次 TLS 握手。可以只对更新检查这类一次性请求关闭，或者干脆只依赖 P1 的强制退出。

建议的优先级是：P1（强制退出加可靠刷新）必须做；S3 作为第二层保障，等 Windows 验证后再决定。

### S4 stdout 已 destroy 时空写入会重新抛错

stdout 已被 destroy 时，`write("")` 会发出 `ERR_STREAM_DESTROYED` 错误事件。此时 `run()` 已经结束，`cleanup()` 已经清掉了 `activeProcessOutputs`；`handleStdoutPipeError` 发现这不是 EPIPE、也没有活跃调用，就会在只有自己一个监听器时 `throw error`，变成未捕获异常。实际很难触发：EPIPE 时已经先 `exit(0)` 了。P1 方案不再做空写入，这个问题自然消失。

### S5 Bun 下 `console.log` 同样会被截断，且无法跟踪

实测 Bun 1.3.14：

- 没碰过 `process.stdout` 时，`console.log` 写 8 MB 后直接 `process.exit`：输出完整（同步写入）。
- 只要访问过 `process.stdout`（哪怕只是 `void process.stdout`），后续的 `console.log` 就变成异步写入：直接 exit 只剩 65,536 字节，空写入刷新后也只有 65,538 字节。
- `console.log` 没有回调，P1 的跟踪方案管不到它。

目前 `packages/*/src` 里没有 `console.log`，所以现在不受影响。建议加一条 lint 规则（`no-console`，限定 `src/`），防止以后有人引入。

### S6 出错路径也会截断

`handleError` 的实现是 `process.exit(reporter.error(error))`：写完 stderr 立即退出。Bun 下 stderr 同样是异步管道，超过 64 KB 的错误输出理论上会被截断。这是历史遗留问题，错误载荷通常很小，优先级低，P1 方案里已经给了改法。

## 测试方法

### 环境矩阵

| 编号 | 环境 | 用途 | 能复现什么 |
| --- | --- | --- | --- |
| E1 | Linux/macOS + Bun 1.3.14 | 最小脚本验证 P1 截断与修复原型 | P1、S5（已实测） |
| E2 | Linux/macOS + 从源码编译的 `bl` 二进制（`tools/release/lib/binary-compile.mjs`，Bun 1.3.14） | 用真实 CLI 验证 P1 | P1 |
| E3 | **Windows 11 + PowerShell 7 + release 二进制**（v2.7.2 或合并前的最新版） | 复现 Issue 的 50 秒滞留，验证修复效果 | Issue 主问题、P1、P2 |
| E4 | Windows 或 Linux + npm 版（Node 22 / 26）+ cn 网络 | 验证 4.3 秒延迟来自更新检查 | P2 |
| E5 | 任意 OS + `npm_config_registry` 或 `~/.npmrc` 指向镜像 | 验证 registry 配置 | P3 |

Bun 1.3.14 可以不覆盖本机 Bun，直接从 npm 拿：

```bash
npm pack @oven/bun-linux-x64@1.3.14
```

```bash
mkdir -p bun-1.3.14 && tar xzf oven-bun-linux-x64-1.3.14.tgz -C bun-1.3.14
```

```bash
export BAILIAN_TEST_BUN=$PWD/bun-1.3.14/package/bin/bun
```

macOS 用 `@oven/bun-darwin-arm64@1.3.14`，Windows 用 `@oven/bun-windows-x64@1.3.14`。

### T1 最小脚本复现 P1（E1，无需仓库依赖）

`pr-style.mjs`（PR 的写法）：

```js
process.stdout.write("x".repeat(8_000_000) + "\n");
const flush = (s) => new Promise((d) => s.write("", () => d()));
await flush(process.stdout);
await flush(process.stderr);
process.exit(0);
```

```bash
$BAILIAN_TEST_BUN pr-style.mjs | wc -c
```

期望 `8000001`，Bun 1.3.14 实测 `65536`。对照：

```bash
node pr-style.mjs | wc -c
```

```bash
$BAILIAN_TEST_BUN pr-style.mjs > out.txt && wc -c out.txt
```

这两条都输出完整，说明问题只在 Bun 加管道时出现。编译二进制同样会出现：

```bash
$BAILIAN_TEST_BUN build --compile pr-style.mjs --outfile pr-style-bin && ./pr-style-bin | wc -c
```

修复原型：把上文 P1 的 `trackStreamWrites` / `flushOutput` 粘到脚本顶部，替换掉空写入刷新，再跑同样的命令，期望得到 `8000001`。

### T2 用真实 `bl` 复现 P1（E2 / E3）

1. 检出 PR 分支并编译二进制：

   ```bash
   git fetch origin pull/306/head:pr306 && git checkout pr306
   ```

   ```bash
   pnpm install --frozen-lockfile && pnpm -r --filter "./packages/*" build
   ```

   然后按 `tools/release/lib/binary-compile.mjs` 的方式（或 `publish.yml` 里的步骤）用 Bun 1.3.14 编译出 `bl` 二进制。

2. 找一个输出超过 64 KB 的命令。不需要凭证的首选是命令树 schema：

   ```bash
   ./bl --introspect > full.json && wc -c full.json
   ```

   确认大于 65,536 字节。如果不够大，可以换成需要凭证的大列表命令（如模型列表的 `--output json`）。

3. 对比管道和文件：

   ```bash
   ./bl --introspect | wc -c
   ```

   修复前（PR 原样）：管道结果小于文件大小，约为 65,536。修复后两者相等。

4. 结构化校验（模拟 Agent 调用）：

   ```bash
   ./bl --introspect | node -e 'let s="";process.stdin.on("data",d=>s+=d).on("end",()=>{JSON.parse(s);console.log("ok",s.length)})'
   ```

   修复前会报 JSON 解析错误，修复后输出 `ok <长度>`。

Windows PowerShell 下（E3），管道到外部程序同样会触发截断：

```powershell
.\bl.exe --introspect | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>console.log(s.length))"
```

```powershell
.\bl.exe --introspect > full.json; (Get-Item full.json).Length
```

### T3 自动化测试（建议加入仓库，S2）

在 `packages/runtime/tests/http-bun.e2e.test.ts` 的同一套框架里（Node、原生 Bun、编译二进制三种运行方式）增加 probe 脚本：

```ts
// probe: 写大输出，经 runToExit 退出；同时建一个本地 keep-alive 连接
import { runToExit } from "../src/index.ts";

await runToExit({
  async run() {
    const res = await fetch(process.env.KEEPALIVE_URL!); // 本地 keep-alive 服务器
    await res.text();
    process.stdout.write(JSON.stringify({ data: "x".repeat(2_000_000) }) + "\n");
  },
});
```

断言：

- stdout 用管道（`execFile` 默认就是）接收，`JSON.parse(stdout)` 成功，`data.length === 2_000_000`。
- 退出码为 0；设置 `process.exitCode = 3` 的变体中退出码为 3。
- 进程在 5 秒内结束（本地服务器设置较长的 keep-alive，如 `server.keepAliveTimeout = 60_000`，确保"不强制退出就会滞留"的对照组确实会滞留）。
- 三种运行方式都要跑；CI 已经安装 Bun 1.3.14，不需要额外配置。

另外给 `flushOutput` 补单元测试：

- 写入回调延迟触发时，`flushOutput` 要等它。
- 回调永不触发时，`flushOutput` 在超时后返回。
- 回调带 error（EPIPE）时也算完成。

### T4 复现 Issue 主问题并验证修复（E3，Windows）

```powershell
$env:NO_UPDATE_NOTIFIER = "1"   # 先排除 P2 的干扰
Measure-Command { .\bl.exe usage token-plan --output json | Out-Host }
```

再测一次管道场景，更接近 Agent 调用：

```powershell
Measure-Command { .\bl.exe usage token-plan --output json | Out-String }
```

| 版本 | 期望总耗时 |
| --- | --- |
| 修复前 release 二进制 | 约 50～56 秒（复现 Issue） |
| PR 原样 | 约 1 秒，但大输出会被截断（T2） |
| PR + P1 方案 | 约 1 秒，输出完整 |
| 仅 S3（`keepalive: false`），不强制退出 | 用来验证根因判断；如果也约 1 秒，说明根因确实是 keep-alive socket |

需要 token-plan 账号和凭证。没有的话，可以换任何会发 HTTPS 请求的命令，比如用一个无效 API key 调模型列表（出错路径本来就会 exit，不适合作对照），或者用 `bl auth status` 这类读远端状态的命令。对照的要点是：命令要发过 HTTPS 请求，并且走成功路径。

辅助定位：滞留期间在另一个窗口运行下面的命令，应能看到指向服务端 443 端口、状态为 Established 的连接；进程退出时这些连接随之消失。

```powershell
Get-NetTCPConnection -OwningProcess (Get-Process bl).Id
```

### T5 验证 P2 更新检查延迟（E4）

```powershell
Remove-Item "$env:USERPROFILE\.bailian\update-state.json" -ErrorAction SilentlyContinue
```

update-state 文件位于 `getConfigDir()/update-state.json`，默认是 `~/.bailian/update-state.json`；设置了 `BAILIAN_CONFIG_DIR` 时位于该目录下。然后连续运行三次，记录每次耗时：

```powershell
1..3 | % { (Measure-Command { bl usage token-plan --output json | Out-Null }).TotalSeconds }
```

- 能正常访问 npmjs 时：第 1 次多约 0.x 秒，第 2、3 次没有额外延迟（命中 24 小时缓存）。
- 访问 npmjs 超时时（cn 网络，或用防火墙规则、`HTTPS_PROXY` 指向一个不通的地址来模拟）：
  - 修复前：**每次**都多约 3 秒（二进制版最多 6 秒）。
  - P2 修复后：只有第 1 次多 1～1.5 秒，之后 1 小时内没有额外延迟。
- 对照：`NO_UPDATE_NOTIFIER=1` 时三次都没有额外延迟。

Linux/macOS 上可以用一个不可达的代理模拟超时：

```bash
HTTPS_PROXY=http://10.255.255.1:9 bl --version
```

注意 `--version` 不经过中间件，所以要换成会进入 `versionCheckStage` 的命令，如 `bl auth status`。

### T6 验证 P3 registry 配置（E5）

修复后，在本地起一个假 registry 观察请求是否到达：

```bash
node -e 'require("http").createServer((q,s)=>{console.error("HIT",q.url);s.end(JSON.stringify({version:"99.0.0"}))}).listen(4873)'
```

清掉 update-state 后：

```bash
npm_config_registry=http://127.0.0.1:4873 bl auth status
```

期望：假 registry 打印 `HIT /bailian-cli/latest`，stderr 出现 `Update available: … → 99.0.0`。

再测 `.npmrc`：把 `registry=http://127.0.0.1:4873/` 写进 `~/.npmrc`，不设环境变量，结果应该相同；环境变量与 `.npmrc` 同时存在时，环境变量优先。修复前，假 registry 收不到任何请求。

单元测试：`resolveNpmRegistry` 覆盖以下情况：环境变量的大小写两种写法、`~/.npmrc`、`@scope:registry`、末尾斜杠、注释行、文件不存在时回退默认值。

## 合并建议

1. **必须**：用 P1 方案替换空写入刷新，补 T3 的 Bun 和编译二进制子进程测试；按 T2 确认大输出在管道下完整。
2. **必须**：PR 描述改为 "Refs #294"，或者在同一个 PR 里完成 P2、P3；否则 Issue 会被自动关闭，剩下两个问题无人跟进。
3. **建议**：P2 失败退避与缩短超时（单独 PR，改动小、收益明显）。
4. **建议**：在 Windows 上按 T4 验证；视结果决定是否加 S3 作为第二层保障。
5. **建议**：加 `no-console` lint 规则（S5）；出错路径改走 `flushOutput`（S6）。
