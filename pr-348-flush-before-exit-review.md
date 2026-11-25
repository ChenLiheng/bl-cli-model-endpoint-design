# PR #348 审查：退出前刷新 stdout/stderr（runToExit）

> 日期：2026-11-25 · 审查对象：`modelstudioai/cli` PR #348（head `2db845f`，1 个提交，基于 main `2561c8c`）· 关联：PR #306（成功路径主动退出，见 [pr-306-exit-after-run-review.md](pr-306-exit-after-run-review.md)）· 状态：讨论稿（已实测）

## 结论与范围

结论：PR **解决了它要解决的两个问题**：Bun/Node 下直接 `exit` 截断管道输出，以及 keep-alive socket 让进程滞留。两者都已实测确认。现有测试和 `vp check` 都通过。

但实测发现一个**新的回归（N1）**：stdout 的读取方超过 5 秒没有读完（例如 `bl … | less`）时，成功路径会把输出截断到 64KB，并且仍然以 0 退出。改动前输出是完整的。建议合并前修复 N1。

初版审查中列为"高"的 P1（退出码被 `exit(0)` 覆盖）**在实测中没有复现**，已降级为防御性建议。其余问题都已实测确认。

| # | 级别 | 问题 | 实测 | 位置 | 建议 |
| --- | --- | --- | --- | --- | --- |
| N1 | 需要处理（高） | 成功路径的 flush 硬超时 5 秒。读取方慢于 5 秒时，输出被截断到 64KB，退出码仍为 0；改动前输出完整 | **已复现**（Node、Bun、真实 `less`） | `runtime/src/create-cli.ts:449-454`、`output/flush.ts:14` | 只要写入还在推进就继续等；或者大幅延长超时，并在超时后设非 0 退出码 |
| P2 | 需要处理（中） | 公开导出的 `handleError` 从同步 `never` 改为 `Promise<never>`。不加 `await` 时类型检查仍能通过，但后面的代码会继续执行 | **已确认** | `runtime/src/error-handler.ts:15`、`runtime/src/index.ts:36` | changelog 标注 breaking；或保留同步版本，另加一个异步版本 |
| M1 | 小问题 | flush 超时后输出一段中英双语纯文本，破坏 `--output json` 下"stderr 为单行 JSON"的约定 | **已复现** | `runtime/src/output/flush.ts:79` | JSON 模式下不输出，或通过 reporter 输出结构化 warning |
| M2 | 小问题 | "第二次 Ctrl-C 立即退出"的测试测不出这个功能 | **已确认**（删掉该逻辑后测试仍通过） | `runtime/tests/interrupt.test.ts:70-94` | 断言第二次 `emit` 后 `exit` 被同步调用 |
| M3 | 小问题 | stdout 出现非 EPIPE 错误时，`flushOutput()` 白等 5 秒 | **已复现**（约 5.06 秒） | `runtime/src/create-cli.ts:174` | 改用短超时 `INTERRUPT_FLUSH_TIMEOUT_MS` |
| M4 | 小问题（原有） | `cli.run()` reject 时跳过 flush，以"未处理的 rejection"退出码 1 退出，打印原始堆栈 | **已复现**（与改动前相同） | `runtime/src/create-cli.ts:449-454` | 用 `try/catch/finally` 统一处理 |
| P1 | 防御性建议（降级） | Ctrl-C 和 stdout 出错的 handler 改成异步 flush 后再退出，理论上 `runToExit` 的 `exit(0)` 可能抢先 | **未复现**（120 次，除信号未送达的情况外全部为 130） | `runtime/src/create-cli.ts:161`、`:174` | 仍建议先写 `process.exitCode = 130`/`code`，成本极低 |

## 测试环境与总体结果

- 环境：Linux 容器，Node v22.23.5，Bun 1.4.4，pnpm 10.33.2（`pnpm install --frozen-lockfile`）。
- **没有**在 Windows、macOS，也没有对 Bun 编译后的 `bl` 发布二进制做实测。Windows 上约 50 秒滞留的问题，是在 Linux 上用本地 keep-alive 服务模拟的。
- 只在本地克隆中读代码和运行测试。为验证 M2 临时改过一处源码，验证后已用 `git checkout --` 恢复，工作区干净。没有修改 `modelstudioai/cli` 远端仓库，也没有在 PR 上评论。

| 检查 | 结果 |
| --- | --- |
| `vp check`（全仓库 lint + 类型） | 0 error，9 warning；warning 都不在本 PR 改动的文件中 |
| `packages/runtime` 测试 | 44 个文件，601 个用例全部通过 |
| 全仓库 `vp test run` | 10026 通过 / 5 失败 / 274 跳过。5 个失败都在 `commands/tests/e2e/skill.e2e.test.ts`，原因是容器访问不了 `bailian-wiki.oss-cn-hangzhou.aliyuncs.com`；在 main `2561c8c` 上同样是这 5 个失败，**与本 PR 无关** |
| `run-to-exit.e2e.test.ts` | Node、Bun 各 2 个用例全部通过（本机有 Bun，Bun 分支没有被跳过） |

## PR 是否解决了问题（前后对比）

实验脚本放在 scratchpad，没有提交，按 PR 自带的 e2e harness 改写。三种退出方式：

- `plain`：只 `await cli.run()`，即 main 上 `bl` 入口的行为；
- `naive`：`run()` 之后直接 `process.exit(0)`，即 PR #306 的做法；
- `runToExit`：本 PR 的做法。

### 大输出写到管道（2,000,012 字节，读取方正常读）

| 运行时 | plain | naive | runToExit |
| --- | --- | --- | --- |
| Node | 2,000,012 ✅ | **65,536** ❌ | 2,000,012 ✅ |
| Bun | 2,000,012 ✅ | **131,072** ❌ | 2,000,012 ✅ |

结论：直接 `exit` 确实会截断输出（Node 截在 64KB，Bun 截在 128KB）；`runToExit` 能完整输出。

### 残留 keep-alive 连接（服务端 `keepAliveTimeout` 设为 20 秒）

| 运行时 | plain | naive | runToExit |
| --- | --- | --- | --- |
| Node | **20.69s** | 0.80s | 0.65s |
| Bun | **20.21s** | 0.16s | 0.19s |

结论：不主动退出时，进程会一直挂到 keep-alive 超时；`runToExit` 能立即退出。

**两个目标问题都解决了。**

## N1：读取方慢于 5 秒时，成功路径的输出被截断（新回归）

`runToExit` 在成功路径用 `flushOutput()`，默认 5 秒超时。超时后直接 `process.exit(process.exitCode ?? 0)`，没有写完的数据就丢了。

实测：2MB 输出，读取方先 `sleep 7` 再读：

| 运行时 | plain（改动前） | runToExit（本 PR） |
| --- | --- | --- |
| Node | 2,000,012，exit 0 | **65,536**，exit 0，stderr 有超时警告 |
| Bun | 2,000,012，exit 0 | **65,536**，exit 0，stderr 有超时警告 |

用真实的 `less` 复现：在 pty 里运行 `bun … | less`，停留 8 秒（相当于用户在看第一页），然后按 `G` 跳到末尾：

- 改动前：能看到最后一行 `END_OF_OUTPUT`；
- 本 PR：**看不到最后一行**，stderr 有 `Warning: timed out waiting for stdout/stderr to flush`。

会触发的场景：

- `bl … | less`、`| more`：用户阅读第一页超过 5 秒；
- 管道下游处理慢，例如 `| jq` 处理大 JSON、`| python slow.py`、`| ssh remote 'cat > file'`；
- Agent 或 CI 读取子进程 stdout 不够及时。

后果是**数据被静默截断**：退出码仍是 0，脚本无法区分。PR 注释也写明"Does not change process.exitCode"。

修复建议（任选其一，第一种最好）：

1. **只要还在推进就继续等**：每完成一次写入就重置计时器，只有"连续 N 秒没有任何写入完成"才算超时。读取方慢但在读，就一直等；真卡死了才放弃。
2. 成功路径不设超时，只在 Ctrl-C 路径保留 500ms 短超时。keep-alive 的问题靠"写完即 `exit`"就能解决，不依赖超时。
3. 至少在超时时把退出码设为非 0（例如 `process.exitCode ||= 1`），让调用方知道输出不完整。

建议补一个 e2e 用例：读取方 `sleep 7` 后再读，断言输出完整。

## P2：`handleError` 的签名改动是公开 API 的破坏性变更（已确认）

- `bailian-cli-runtime` 是会发布的包（`package.json` 中 version 2.11.2，没有 `private`），`index.ts:36` 导出了 `handleError`。
- 实测：按旧方式调用 `handleError(new Error("x"), reporter);`，不加 `await`，**下一行代码照样会执行**，之后才以 reporter 返回的退出码退出。
- 仓库内只有 `create-cli.ts:432` 一处调用，已改为 `await`；`vp check` 也没有报 floating promise，说明 lint 规则管不到外部调用方。

建议二选一：在 changelog 中标明 breaking 并提升相应版本号；或者保留同步的 `handleError`，另外导出 `handleErrorAsync` 给 runtime 内部使用。

## 小问题

### M1：超时警告污染 JSON 输出（已复现）

`--output json`，命令先输出 2MB 再抛错，读取方 7 秒后才读。stderr 内容：

```
{"error":{"code":1,"message":"boom","cause":{"message":"boom"}}}
Warning: timed out waiting for stdout/stderr to flush; output may be truncated.
警告：等待标准输出刷新超时，输出可能被截断。
```

按"stderr 为单行 JSON"解析会失败（`error-handler.test.ts` 也断言 stderr 只有一行）。建议 JSON 模式下不输出，或交给 reporter 输出结构化 warning（例如 `code: "OUTPUT_FLUSH_TIMEOUT"`）。

### M2：第二次 Ctrl-C 的测试没有区分度（已确认）

在本地临时删掉 `handleInterrupt` 中的这段代码：

```ts
if (interruptExiting) {
  process.exit(130);
  return;
}
```

然后运行 `packages/runtime/tests/interrupt.test.ts`，**两个用例仍然全部通过**。原因：第二次 SIGINT 会再走一遍完整的链，同样产生两次 `exit(130)`。建议改为：

```ts
process.emit("SIGINT");
expect(exit).not.toHaveBeenCalled();
process.emit("SIGINT");
expect(exit).toHaveBeenCalledTimes(1);   // 同步、立即
expect(exit).toHaveBeenCalledWith(130);
```

### M3：非 EPIPE 的 stdout 错误白等 5 秒（已复现）

命令先输出 2MB，读取方暂不读，50ms 后让 stdout 触发 `EIO` 错误：Node、Bun 都是约 **5.06 秒**后才退出，stderr 还会带上 M1 的超时警告。这条路径上 stdout 已经不可用，建议用 `INTERRUPT_FLUSH_TIMEOUT_MS`（500ms）。

### M4：`run()` reject 时以原始堆栈退出（已复现，原有问题）

初版审查说"不会退出"，这一点不准确。实测：`run()` reject 时，Node 和 Bun 都会因为"未处理的 rejection"以退出码 1 退出，并打印原始堆栈；只是跳过了 flush，也没有按 CLI 的错误格式输出。改动前也是这样，不算回归。`runToExit` 的职责是"保证退出"，建议顺手统一处理：

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

## P1：退出码被 `exit(0)` 覆盖（未复现，降级）

初版的推理是：Ctrl-C handler 改成"先异步 flush 再 `exit(130)`"之后，命令如果在这段时间跑完，`runToExit` 的 `exit(0)` 可能抢先。

实测：用真实的 `createCli` + fixture 命令，命令内部给自己发 SIGINT，然后在 0/5/50/300ms 后正常返回。分别测了三种情况：没有遥测；stdout 积压 2MB 且读取方 2 秒后才读；用一个不响应的 fetch 模拟遥测请求一直挂着（Bun 上确认有 1–2 个请求在途）。三组共 120 次（无遥测 80 次、stdout 积压 4 次、遥测挂起 36 次），覆盖 Node、Bun、`plain`、`runToExit` 的组合，退出码**全部是 130**。唯一的例外是 Node 上延迟 0ms 的那组：`plain` 和 `runToExit` 都是 0，因为信号还没送达命令就结束了，改动前后一样，与本 PR 无关。

为什么没有发生：

- `run()` 成功路径本身就会 `await flushTelemetry(1000)`，等的是同一批在途请求，所以不会比中断链更早结束；
- 中断链的 flush 等的是更早的写入快照，超时也更短（500ms），总是先完成。

因此这在实际中不是回归。不过在 handler 里先写 `process.exitCode = 130`（stdout 出错路径写 `= code`）只要两行，能消除对时序的依赖，仍然建议加上。

## 已检查、没有发现问题的点

- **长时间运行的命令不会被提前杀掉**：`config ui`、`auth login --console`、managed-agent playground 都在 `run()` 内部 await 到结束（`config/ui.ts:801`、`auth/login-console.ts:627`、`playground-launcher.ts:144`）。
- **playground 打开浏览器**用 `detached: true` + `unref()` 启动子进程（`playground-launcher.ts:461-465`），父进程退出不影响它。
- **更新检查**在 `versionCheckStage` 中被 await（`middleware.ts:314-320`）；远程遥测在 `run()` 内通过 `flushTelemetry(1000)` 刷新。
- **in-process 测试 harness 不受影响**：写入追踪只在 `runToExit` 中安装，没有追踪时 `flushOutput` 直接返回。
- **write 包装**：处理了两种重载、回调报错和同步抛异常；`WeakSet` 防止重复包装；写入完成后从 `pending` 中删除，不会泄漏内存。
- **非 0 的 `process.exitCode`** 能被保留（PR 的 e2e 用例在 Node、Bun 上都通过）。

## 建议的合并条件

1. 修复 N1（推荐"只要还在推进就继续等"），并补一个"读取方慢于 5 秒"的 e2e 用例。
2. 处理 P2：changelog 标注 breaking，或保留同步 API。
3. M1–M4 和 P1 的防御性改动可以在本 PR 中顺手修，也可以另开 issue 跟进。
