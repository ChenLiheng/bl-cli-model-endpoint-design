# PR #321 审查：speech synthesize 超时覆盖音频下载（Issue #309）

> 日期：2026-11-24 · 审查对象：`modelstudioai/cli` PR #321（head `3e5a7400`，2 个提交，基于 main `eef8336f`）· 关联：Issue #309 · 状态：讨论稿

## 结论与范围

结论：PR 修复了 Issue #309 报告的路径（REST 非流式 + `--out`），实现质量不错。但它**没有完全修复 #309**：`--timeout` 大于 300 秒时，下载仍会被传输层默认超时截断（P1）；`--model auto`（MCP）下载和 `--stream` 模式也还有同类问题。建议把 PR 描述里的 `Closes #309` 改为 `Refs #309`，或者在本 PR 中补上 P1、P2。

| # | 级别 | 问题 | 位置 | 建议 |
| --- | --- | --- | --- | --- |
| P1 | 需要处理（高） | 下载请求没有关闭 undici/Bun 默认的 300s 超时，`--timeout` 大于 300 时下载仍在 300s 被截断，报 GENERAL 而不是 TIMEOUT | `runtime/src/utils/download.ts:17` | 复用 core `http.ts` 的 `withoutTransportTimeouts` dispatcher 和 Bun 的 `timeout: false` |
| P2 | 需要处理（中） | `--model auto`（MCP 路径）的下载没有传 signal，卡住时依旧不超时，半截文件留在 `--out` | `speech/synthesize.ts:601` | 与 REST 路径共用同一个 deadline 和 `track` |
| P3 | 需要处理（中） | `--stream` 模式只有空闲超时、没有总超时；直接写 `--out`，失败时留下半截文件，并且仍打印 `Saved:` | `speech/synthesize.ts:746`、`:790` | 增加总 deadline；写临时文件后再 rename；只在成功时打印 `Saved:` |
| P4 | 需要处理（中） | 返回 200 但 body 为空或不是音频时，同样会覆盖原有的好文件 | `runtime/src/utils/download.ts:104-113` | 校验 `received === content-length`，拒绝 0 字节 |
| P5 | 需要处理（低） | `--timeout` 没有上限，超过约 24.8 天时 `setTimeout` 溢出，1ms 后就超时 | `core/src/config/validation.ts:10`、`speech/synthesize.ts:629` | 在 `validateTimeout` 中设上限 |
| B1 | 边界 | `--out` 是软链接时，软链接被普通文件替换，原目标文件没有更新 | `download.ts:113` | 先 `realpath` 再在目标所在目录暂存，或在文档中说明 |
| B2 | 边界 | 并发部分成功：已 rename 的文件留在 `speech/`，但 stdout 为空，错误里也没提 | `speech/synthesize.ts:702-706` | 失败时清理本批次已落盘的文件，或在错误中列出 |
| B3 | 边界 | 父目录不可写、只有目标文件可写时，新路径失败（旧路径成功），抛出原始的 `EACCES` | `download.ts:91` | 捕获后转成 `BailianError`；必要时回退到直接写入 |
| B4 | 边界 | `finally` 中 `rm`、`stat` 抛出的错误可能覆盖原始的 TIMEOUT 错误 | `download.ts:107`、`:117` | `rm` 吞掉错误；`stat` 的非 ENOENT 错误转成 `BailianError` |
| B5 | 边界 | `--out` 指向已存在的目录时，`rename` 报 `EISDIR`/`ENOTEMPTY`，是原始 Node 错误，退出码不对 | `download.ts:111-113` | 预先检查目录，报 USAGE 错误 |
| B6 | 边界 | 进程被 SIGKILL 或 Ctrl+C 时，`.bailian-audio-*` 临时目录残留；Windows 下目标文件被占用时 `rename` 报 EPERM | `download.ts:91`、`:113` | 启动时清理过期的残留目录；处理 SIGINT |
| S1 | 建议 | `--timeout` 语义从"每个请求"变成"合成 + 全部下载的总时长"，与 image/video 命令不一致 | `speech/synthesize.ts:619-630` | 在 `--help` 和 changelog 中写明 |
| S2 | 建议 | 只有带 `signal` 的调用才走"暂存 + rename"，image/video/workflow export/pipeline 仍用旧逻辑，下载也没有超时 | `download.ts:27-30` | 后续统一成一条路径 |
| S3 | 建议 | 测试依赖 300ms–1.5s 的定时器，慢 CI 上可能不稳定；缺少 MCP 下载卡住的用例 | `speech-synthesize-output.test.ts` | 放宽阈值或使用 fake timers；补 P2 的用例 |

审查方法：

- 克隆 `modelstudioai/cli`，`git fetch origin pull/321/head`，阅读全部源码与测试改动，以及相关的 core 代码（`client/http.ts`、`client/transport.ts`、`config/validation.ts`）和 runtime 代码（`utils/concurrent.ts`、`proxy.ts`）。
- 在 PR head 上运行 `packages/commands/tests/output/speech-synthesize-output.test.ts`，53 个用例全部通过（Node，Linux）。
- 写了一个临时测试，直接调用带 `signal` 的 `downloadFile`，复现了 P4、P5、B1。测试跑完就删了，没有提交。B3 因为容器以 root 运行，权限检查不生效，没能复现。
- 没有对真实 DashScope、分发的 `bl` 二进制（Bun）验证，也没有验证服务端取消。

## Issue #309 回顾

- **命令**：`bl speech synthesize --model cosyvoice-v3-flash --format wav --sample-rate 24000 --enable-ssml --text-file … --out x.wav --output json --timeout 180`。没有 `--stream`，所以走 `handleNonStreamMode`。
- **现象**：偶发卡住，一次至少 5 分 43 秒，另一次被外部 240s deadline 杀掉。WAV 停在 540,995 字节，RIFF/data 长度还是占位值；没有任何错误，也没有 request ID。
- **期望**：在 deadline 内完成，或者返回结构化的超时错误，并清楚标明不完整的 WAV。
- **根因**：非流式模式先 `requestJson` 拿到音频 URL（这一步遵守 `--timeout`），再调 `downloadFile` 下载。`downloadFile` 直接调 `httpTransport.fetch(url)`，不带 signal 和 deadline，并且直接写入 `--out`，所以卡住或被 kill 时会留下半截文件。
- **5 分 43 秒**：与 undici 默认 300s 的 `headersTimeout`/`bodyTimeout` 加上前面的耗时相符（推测，未验证）。这一点也是 P1 的来源。

## PR 做了什么

1. `handleNonStreamMode` 创建一个 `AbortController`，用 `settings.timeout` 作为总 deadline，同时覆盖合成请求（`requestJson({ signal })`）和全部下载（`downloadFile(..., { signal })`）。超时时 reason 是 `BailianError(ExitCode.TIMEOUT)`，最终以退出码 5 和结构化 JSON 错误结束。
2. 任一操作失败时，abort 其余操作，用 `Promise.allSettled(pending)` 等它们结束，抛出的仍是最初的那个错误。
3. 新增 `downloadWithDeadline`：先在目标目录下 `mkdtemp(".bailian-audio-")`，用 `pipeline(Readable.fromWeb(body), meter, createWriteStream(partial), { signal })` 写入临时文件，完成后把原文件的 `mode & 0o777` 复制过来，再 `rename` 覆盖目标；`finally` 中删除临时目录。
4. 测试覆盖：卡在响应头、卡在 body、合成和下载共用预算、超时后保留原文件、保留 0600/0664 权限、并发时一个失败会取消另一个、真实 CLI 的 `--timeout` 退出码。

这几项做得是对的，下面只讨论还没覆盖到的部分。

## P1：下载仍受传输层默认 300s 超时限制

### 现象

core 的 `http.ts` 专门关闭了传输层超时，注释写的是 "Core owns the request deadline"：

```ts
// packages/core/src/client/http.ts:14
const withoutTransportTimeouts = (dispatch) => (options, handler) =>
  dispatch({ ...options, headersTimeout: 0, bodyTimeout: 0 }, handler);
// :103
...(process.versions.bun ? { timeout: false } : { dispatcher: compose(getGlobalDispatcher(), withoutTransportTimeouts) })
```

而 PR 里的下载请求没有带这些参数：

```ts
// packages/runtime/src/utils/download.ts:17
await httpTransport.fetch(url, { signal: opts.signal })
```

全局 dispatcher 是 undici 默认的 Agent，或者 `proxy.ts` 设置的 `EnvHttpProxyAgent`，默认 `headersTimeout`/`bodyTimeout` 都是 300s；Bun 的 fetch 也有类似的默认超时。因此：

- `--timeout 600` 时，如果服务端 300s 内没有返回响应头，或者两个数据块之间间隔超过 300s，下载会在 300s 被传输层截断。
- 报出的是 `UND_ERR_HEADERS_TIMEOUT`、`UND_ERR_BODY_TIMEOUT` 或 `terminated` 这类原始错误，走 GENERAL 退出码，不是 5。PR 的测试只用了 0.3s/1s 的超时，覆盖不到这个情况。

这是 #309 中"5 分多钟"最可能的来源。PR 修好了 `--timeout` 小于 300 的情况，`--timeout` 大于 300 时仍然不会按用户的设置生效。

### 建议

把 core 的请求选项抽成一个可复用的导出函数，下载时也用上：

```ts
// core/src/client/http.ts
export function transportInit(): Partial<RequestInit> {
  return process.versions.bun
    ? { timeout: false as const }
    : { dispatcher: Dispatcher.prototype.compose.call(getGlobalDispatcher(), withoutTransportTimeouts) };
}

// runtime/src/utils/download.ts
const res = await httpTransport.fetch(url, opts?.signal ? { signal: opts.signal, ...transportInit() } : undefined);
```

注意：不带 signal 的旧调用方不能这样改，否则会失去唯一的超时保护（见 S2）。

补一条测试：把 undici 全局 dispatcher 的 `headersTimeout` 设为 200ms，`--timeout 1`，服务端 500ms 后才返回响应头，预期下载成功。

## P2：`--model auto`（MCP 路径）的下载没有超时

`handleMcpMode` 最后仍然是旧写法：

```ts
// speech/synthesize.ts:601
const saved = await downloadParallel(items, downloadFile, { quiet: ... });
```

这条路径上的 MCP `tools/call` 有自己的超时（`mcp-sdk.ts:469`），但之后的音频下载没有 signal，卡住时和 #309 的行为完全一样，并且会把半截文件写到 `--out`。

建议：把 `handleNonStreamMode` 里创建 deadline、`track`、在 catch 中 abort 再 allSettled 的那段逻辑抽成 `withSynthesisDeadline(settings, async (signal, track) => …)`，REST 和 MCP 两条路径共用。另外补一条用例：MCP 下载卡在 body 时，以退出码 5 结束，并且不留下文件。

## P3：`--stream` 模式没有总超时，失败时留下半截文件

`handleStreamMode` 用的是 `client.request({ stream: true })`，`protectResponseBody` 给它的是滚动的空闲窗口：每收到一个数据块就重新计时。所以：

- **不会超时**：只要服务端每隔不到 `--timeout` 秒吐一点数据，命令就会一直跑下去。#309 的标题是 "does not terminate stalled audio streaming"，Suspected area 里也提到了 streaming read loop。
- **半截文件**：`createWriteStream(outPath)` 直接写目标文件（`:746`）。出错时 `finally` 只是 `end()`，半截文件会留下，也没有任何不完整的标记。
- **误导性输出**：`finally` 里无条件执行 `commandOutput.info("Saved: …")`（`:790`），失败时也会打印"已保存"。

建议：

1. 增加一个总 deadline，与非流式模式一致，传给 `client.request({ signal })`。
2. 写 `--out` 时也采用"暂存 + rename"，失败时删除临时文件。如果确实需要保留半截文件，就改名为 `x.wav.partial`，并在错误的 hint 中说明。
3. 把 `Saved:` 移出 `finally`，只在成功时打印。
4. 写到 stdout 的流式数据无法撤回，在错误输出里说明已经写出了多少字节即可。

如果不打算在本 PR 处理，建议另开 issue 跟进，并在 PR 描述里说明不在本次范围内。

## P4：200 但内容为空或不是音频时，照样覆盖原文件

### 复现

用临时测试直接调用 PR 版本的 `downloadFile(url, dest, { signal })`，`dest` 里原有内容 `GOOD`：

| 服务端响应 | 结果 |
| --- | --- |
| `200`，`content-length: 0`，空 body | 返回 `{"size":0}`，`dest` 变成空文件 |
| `200`，`text/html`，body 为 `<html>err</html>` | `dest` 被这段 HTML 覆盖 |

`downloadWithDeadline` 只要流读到 EOF 就认为成功并执行 rename，没有核对长度，也不拒绝空响应。旧代码也有这个问题，但这个 PR 的卖点之一就是"只有完整下载成功才替换原文件"，那就应该把"成功"定义清楚。

### 建议

```ts
await pipeline(...);
if (received === 0) throw new BailianError("Download returned an empty body.", ExitCode.GENERAL);
if (contentLength > 0 && received !== contentLength) {
  throw new BailianError(`Download incomplete: ${received}/${contentLength} bytes.`, ExitCode.GENERAL);
}
```

对 `content-type` 可以只做宽松检查，比如拒绝 `text/html`，避免 OSS 返回的错误页被当成音频保存。

## P5：`--timeout` 没有上限，过大时立即超时

`validateTimeout` 只检查是否为有限数值且大于 0（`validation.ts:10`）。`setTimeout` 能接受的最大延迟是 2³¹−1 毫秒（约 24.8 天），超过后 Node 会给出 `TimeoutOverflowWarning`，并把延迟改为 1ms。

实测：`setTimeout(fn, 3_000_000 * 1000)` 在 1ms 后就触发了。也就是说 `bl speech synthesize --timeout 3000000 …` 会马上以 TIMEOUT 失败。core 的 `createRequestSignal` 本来就有同样的问题，这个 PR 又加了一个同类的定时器。

建议：在 `validateTimeout` 中加上上限（例如 `timeout * 1000 <= 2_147_483_647`），超出时报 USAGE 错误；或者把超大值当作"不设超时"处理。

## 边界问题

### B1：`--out` 是软链接

实测：`link.wav -> real.wav`，下载后 `link.wav` 变成普通文件（`isSymbolicLink() === false`），`real.wav` 仍然是旧内容。旧代码是顺着软链接写进 `real.wav` 的。

`stat(destPath)` 会跟随软链接，所以复制的是目标文件的权限，但 `rename` 替换掉的是软链接本身。硬链接同理会断开。另外文件属主、ACL、xattr 都不会保留。

建议：先用 `realpath(destPath)` 求出真实路径（目标不存在时退回原路径），在真实路径所在的目录暂存并 rename；或者在文档中写明这个行为。

### B2：并发部分成功，已落盘的文件没有报告

用 `--concurrent 2` 时，第一个文件已经 rename 到 `speech/tts_…_001.wav`，第二个才超时或失败。此时 stdout 为空，错误里只有失败原因，调用方（尤其是 Agent）不知道已经生成了一个文件，也可能已经为它付费。PR 的 `concurrent-peer` 测试只检查了失败的那个没有残留。

建议：二选一，保持一致即可。

- 失败时删除本批次已经 rename 的文件，做到"全部成功或全部不落盘"；
- 在错误的 details 中列出已保存的文件。

### B3：父目录不可写

旧代码只需要目标文件可写。新代码需要在父目录中执行 `mkdtemp` 和 `rename`，所以目录必须可写，否则抛出原始的 `EACCES`（不是 `BailianError`，退出码不对）。容器以 root 运行，没能复现。

建议：把 `mkdtemp` 的错误转成带 hint 的 `BailianError`；如果要兼容这种情况，就在 `EACCES` 时回退到直接写入目标文件。

### B4：`finally` 中的错误覆盖原始错误

- `finally { await rm(directory, { recursive: true, force: true }) }`：`force` 只会忽略 ENOENT。Windows 上如果文件句柄还没释放导致 EBUSY 或 EPERM，`rm` 抛出的错误会盖掉原本的 TIMEOUT 错误。
- `stat(destPath)` 遇到 ENOENT 以外的错误（`EACCES`、`ENOTDIR`）会直接抛出原始错误。

建议：`rm` 改为 `.catch(() => {})`，失败时最多输出一条 debug 日志；`stat` 的其他错误转成 `BailianError`。

### B5：`--out` 指向已存在的目录

`stat` 返回目录，不会执行 chmod，接着 `rename(partial, dir)` 报 `EISDIR` 或 `ENOTEMPTY`，抛出的是原始错误。建议在下载之前就检查，报 USAGE 错误，这样还能避免白白发起一次合成请求。

### B6：临时目录残留与 Windows 下的 rename

- 进程被 SIGKILL 杀掉（#309 的批处理脚本就是在外部 deadline 到了以后 kill 子进程）或用户按 Ctrl+C 时，`finally` 来不及执行，目标目录下会残留 `.bailian-audio-*`，批量任务中会越积越多。建议启动时清理同目录下超过一定时长的残留目录，或者注册 SIGINT/SIGTERM 处理，在退出前清理。
- Windows 上如果目标文件正被播放器打开，`rename` 会报 EPERM，而旧代码直接写入有可能成功。建议把这个错误转成带 hint 的 `BailianError`。

## 建议项

### S1：`--timeout` 语义变化

以前 `--timeout` 是每个 HTTP 请求各自计时；现在 `speech synthesize` 非流式模式下，它是合成请求加上全部下载的总时长。`--concurrent N` 或文件较大时更容易触发超时。这个语义是合理的，issue 期望的也是这样，但 image/video 等命令仍然是每个请求各自计时，同一个参数在不同命令里含义不一样。建议在 `--help` 和 changelog 中写明，长期把所有命令统一成"命令总时长"。

### S2：两条下载路径分叉

`downloadFile` 现在有两套逻辑：带 `signal` 时走"暂存 + rename + deadline"，不带时走旧的直接写入、没有超时的逻辑。image/video/workflow export/pipeline 仍然走旧逻辑，它们的下载同样可能卡住。建议后续统一：所有调用方都传入 deadline，删掉旧逻辑。

### S3：测试

- 定时器阈值偏小（0.3s 的超时、600ms 的延迟、1.5s 的 watchdog），在负载高的 CI 机器上可能出现偶发失败，建议放宽阈值或改用 fake timers。
- 缺少的用例：P1（`--timeout` 大于传输层默认超时）、P2（MCP 下载卡住）、P4（空 body、长度不符）、B1（软链接）。

## 建议的处理顺序

1. **必须修**：P1。不修的话，`--timeout` 大于 300 时依然不起作用，#309 没有真正闭环。
2. **建议在本 PR 里一起修**：P2（改动很小）、P4、P5。
3. **可以记 follow-up**：P3（单独开 issue），B1–B6，S1–S3。
