# PR #348 审查：退出前刷新 stdout/stderr（runToExit）

> 日期：2026-11-25 · 审查对象：`modelstudioai/cli` PR #348（head `2db845f`，1 个提交，基于 main `2561c8c`）· 关联：PR #306（成功路径主动退出，见 [pr-306-exit-after-run-review.md](pr-306-exit-after-run-review.md)）· 状态：讨论稿

## 结论与范围

结论：思路正确，实现总体干净。合并前建议修复 P1（改两行即可），P2 至少在 changelog 中注明是 breaking change。其余是小问题。

| # | 级别 | 问题 | 位置 | 建议 |
| --- | --- | --- | --- | --- |
| P1 | 需要处理（高） | Ctrl-C 和 stdout 出错的 handler 改成先异步 flush 再退出。这段时间里命令如果跑完，`runToExit` 会以 `exit(0)` 退出，覆盖 130 或错误码 | `runtime/src/create-cli.ts:161`、`:174`、`:454` | 开始异步 flush 前，先写 `process.exitCode = 130`（或 `= code`） |
| P2 | 需要处理（中） | 公开导出的 `handleError` 从同步 `never` 改为 `Promise<never>`。外部调用方不加 `await` 时类型检查仍能通过，但后面的代码会继续执行 | `runtime/src/error-handler.ts:15`、`runtime/src/index.ts:36` | changelog 标注 breaking；或保留同步版本，另加一个异步版本 |
| M1 | 小问题 | flush 超时后用 `writeSync(2, …)` 输出一段中英双语纯文本，会破坏 `--output json` 下"stderr 为单行 JSON"的约定 | `runtime/src/output/flush.ts:79` | JSON 模式下不输出，或通过 reporter 输出结构化 warning |
| M2 | 小问题 | "第二次 Ctrl-C 立即退出"的测试测不出这个功能：去掉 `interruptExiting` 后测试仍然通过 | `runtime/tests/interrupt.test.ts:70-94` | 断言第二次 `emit` 后 `exit` 被同步调用（在任何 await 之前检查） |
| M3 | 小问题 | stdout 出现非 EPIPE 错误时，排队中写入的回调可能不会触发，`flushOutput()` 会白等默认的 5 秒 | `runtime/src/create-cli.ts:174` | 改用短超时，例如 `INTERRUPT_FLUSH_TIMEOUT_MS` |
| M4 | 小问题（原有） | `cli.run()` reject（如首次安装上报路径抛错）时会跳过 flush 和 exit，变成未处理的 rejection | `runtime/src/create-cli.ts:449-454` | 用 `try/finally` 保证 flush 和退出 |

审查方法：

- 克隆 `modelstudioai/cli`，`git fetch origin pull/348/head`，阅读全部 13 个文件的改动，以及相关的 runtime、core、commands 代码（middleware、telemetry sink、local-server、config ui、login-console、playground-launcher）。
- **没有运行测试**（需要先安装依赖），以上结论都来自读代码。没有对 Bun 分发的二进制或 Windows 做实测。
- 没有在 PR 上发评论，也没有修改 `modelstudioai/cli` 仓库。

## PR 改了什么

- 新增 `runtime/src/output/flush.ts`：
  - `trackStreamWrites(stream)` 包装 `stream.write`，每次写入生成一个 promise，在写入回调触发时 settle。出错和同步抛异常时也会 settle。
  - `flushOutput(timeoutMs)` 等待所有未完成的写入，或者等到超时。超时返回 `true`，并向 stderr 输出一条警告。
- 新增 `runToExit(cli)`：先安装 stdout/stderr 的写入追踪，再 `await cli.run()` 和 `flushOutput()`，最后 `process.exit(process.exitCode ?? 0)`。
  - 目的一：Bun 下 stdout 是管道时，直接 `exit` 会截断输出。
  - 目的二：Bun keep-alive socket 让 Windows 上的进程多挂约 50 秒。
- `bl`、`kscli` 的入口从 `createCli(...).run()` 改为 `runToExit(createCli(...))`。
- `handleError`、`handleInterrupt`、`handleStdoutPipeError` 都改成先 flush 再退出。Ctrl-C 增加了"第二次立即退出"。
- 测试：新增 flush 单测、`runToExit` 单测、Node/Bun 的 2MB 管道输出 e2e、第二次 Ctrl-C 用例；`error-handler`、`preparation` 测试改为 async。

## P1：退出码可能被 `runToExit` 的 `exit(0)` 覆盖

改动前：

```ts
void flushTelemetry(500).finally(() => process.exit(130));   // handleInterrupt
if (active) process.exit(active.getReporter().error(error));  // handleStdoutPipeError
```

改动后，两者都要先等异步 flush 完才退出：

```ts
void flushTelemetry(500)
  .catch(() => undefined)
  .then(() => flushOutput(INTERRUPT_FLUSH_TIMEOUT_MS))  // 合计最长约 1s
  .finally(() => process.exit(130));
```

问题场景（Ctrl-C）：

1. 命令没有注册 `registerInterruptHandler`。SIGINT 只会打印 "Interrupted" 并启动上面这条链，命令本身不会中止。
2. 命令在这约 1 秒内跑完，`run()` resolve。
3. `runToExit` 调 `flushOutput()`，没有待完成的写入时立即返回，然后执行 `process.exit(process.exitCode ?? 0)`，即 `exit(0)`。
4. 结果：终端显示 "Interrupted"，退出码却是 **0**。

改动前，链上待执行的 timer 会让进程一直活到 `exit(130)`，所以这是本 PR 引入的回归。`handleStdoutPipeError` 的非 EPIPE 分支也有同样的竞争，只是概率更低。

修复建议：

```ts
// handleInterrupt
interruptExiting = true;
process.exitCode = 130;
// ...

// handleStdoutPipeError
const code = active.getReporter().error(error);
process.exitCode = code;
void flushOutput(INTERRUPT_FLUSH_TIMEOUT_MS).finally(() => process.exit(code));  // 顺带处理 M3
```

`runToExit` 本来就读 `process.exitCode`，所以无论哪一方先调用 `exit`，退出码都正确。

建议补一个用例：命令在 SIGINT 之后 100ms 内正常返回，断言最终退出码是 130。

## P2：`handleError` 的签名改动是公开 API 的破坏性变更

- `bailian-cli-runtime` 是会发布的包（`package.json` 中 version 2.11.2，没有 `private`），`index.ts:36` 导出了 `handleError`。
- 签名从 `(…) => never` 变成 `(…) => Promise<never>`。旧的调用方式 `handleError(e, r); /* 不可达 */` 在类型上仍然能通过，但运行时 `handleError` 之后的代码会继续执行，直到 flush 完才退出。
- 仓库内只有 `create-cli.ts:432` 一处调用，已改为 `await`，内部没有问题。

建议二选一：

- 在 changelog / release note 里标明 breaking，并提升相应的版本号；
- 或保留同步的 `handleError`，另外导出 `handleErrorAsync`（或 `handleErrorAndFlush`）给 runtime 内部使用。

## 小问题

### M1：超时警告污染 JSON 输出

`flushOutput` 超时后会直接向 fd 2 写：

```
Warning: timed out waiting for stdout/stderr to flush; output may be truncated.
警告：等待标准输出刷新超时，输出可能被截断。
```

`--output json` 时，Agent 和脚本按 JSON 解析 stderr（`error-handler.test.ts` 也断言 stderr 只有一行 JSON），这两行纯文本会导致解析失败。建议 JSON 模式下不输出，或交给 reporter 输出结构化 warning（例如 `code: "OUTPUT_FLUSH_TIMEOUT"`）。

### M2：第二次 Ctrl-C 的测试没有区分度

测试只断言 `exit` 至少被调用 2 次、参数都是 130。去掉 `interruptExiting` 后，第二次 SIGINT 会再走一遍完整的链，同样产生两次 `exit(130)`，测试依然通过。建议改为：

```ts
process.emit("SIGINT");
expect(exit).not.toHaveBeenCalled();
process.emit("SIGINT");
expect(exit).toHaveBeenCalledTimes(1);   // 同步、立即
expect(exit).toHaveBeenCalledWith(130);
```

### M3：非 EPIPE 的 stdout 错误白等 5 秒

stdout 已出错时，排队中写入的回调不一定会触发，`flushOutput()` 会等满默认的 5 秒（`DEFAULT_FLUSH_TIMEOUT_MS`）。这条路径上 stdout 已经不可用，建议用 `INTERRUPT_FLUSH_TIMEOUT_MS`（500ms）。

### M4：`run()` reject 时不退出（原有问题）

```ts
export async function runToExit(cli: Cli, argv?: string[]): Promise<void> {
  trackStreamWrites(process.stdout);
  trackStreamWrites(process.stderr);
  await cli.run(argv);           // reject 时下面两行都不执行
  await flushOutput();
  process.exit(process.exitCode ?? 0);
}
```

`run()` 的主路径会在 `handleError` 中退出，不会 reject。但首次安装上报分支（`reportFirstInstallNow`）在 try/catch 之外，抛错时 `void runToExit(...)` 会变成未处理的 rejection，也不会主动退出。改动前也是这样，不算回归。不过 `runToExit` 的职责就是"保证退出"，建议：

```ts
let failed = false;
try {
  await cli.run(argv);
} catch (error) {
  failed = true;
  console.error(error);
} finally {
  await flushOutput();
  process.exit(process.exitCode ?? (failed ? 1 : 0));
}
```

## 已检查、没有发现问题的点

- **长时间运行的命令不会被提前杀掉**：`config ui`、`auth login --console`、managed-agent playground 都是在 `run()` 内部 await 到结束（`config/ui.ts:801`、`auth/login-console.ts:627`、`playground-launcher.ts:144`），`runToExit` 的强制退出发生在它们结束之后。
- **playground 打开浏览器**用 `detached: true` + `unref()` 启动子进程（`playground-launcher.ts:461-465`），父进程退出不影响它。
- **更新检查**在 `versionCheckStage` 中被 await（`middleware.ts:314-320`）；远程遥测在 `run()` 内通过 `flushTelemetry(1000)` 刷新。`localSink` 是 fire-and-forget，但只有一个 `mkdir` 的 await，之后同步 append，被截断的风险很低。
- **in-process 测试 harness 不受影响**：写入追踪只在 `runToExit` 中安装；没有追踪时 `flushOutput` 发现没有待完成的写入，直接返回。
- **write 包装**：处理了 `write(chunk, cb)`、`write(chunk, encoding, cb)` 两种重载，以及回调报错、同步抛异常；`WeakSet` 防止重复包装；写入完成后从 `pending` 删除，不会泄漏内存。
- **flush 超时后**会把未完成的 promise 从 `pending` 中移除，不会拖住之后的 flush。
- **e2e 用例**同时覆盖 Node 和 Bun（Bun 不存在时跳过）：2MB 管道输出完整、带 keep-alive 时 10 秒内退出、非 0 的 `process.exitCode` 被保留。场景与 PR #306 审查中发现的截断问题一致。

## 建议的合并条件

1. 修复 P1，并补一个"SIGINT 后命令很快正常返回，退出码仍为 130"的用例。
2. 处理 P2：changelog 标注 breaking，或保留同步 API。
3. M1–M4 可以在本 PR 中顺手修，也可以另开 issue 跟进。
