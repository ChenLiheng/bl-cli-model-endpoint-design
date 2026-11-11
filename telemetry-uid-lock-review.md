# 遥测 uid 并发发布：mkdir 锁的问题与两种落地方案

> 日期：2026-11-11 · 审查对象：`modelstudioai/cli` PR #314（head `1b757ea`，基于 main `fa8330c2`）中的 `packages/core/src/telemetry/uid.ts` · 状态：讨论稿

## 结论

PR #314 用 mkdir 互斥锁保护 `telemetry-uid` 的首次创建与修复。这把锁有 7 个问题，其中 2 个是 mkdir 锁固有的，打补丁消不掉。

本文给出两种可落地的替代实现，都已写成原型并做过多进程压测：

| | 方案 A：无锁（推荐） | 方案 B：保留锁并修复 |
| --- | --- | --- |
| 思路 | 临时文件 + `link()`，不存在才创建 | 修掉锁的 4 个实现缺陷，发布改为不覆盖 |
| 崩溃残留 | 一个 `.tmp` 文件，不影响任何人 | 锁目录，仍需按时间回收 |
| 阻塞 | 不等待 | 最多同步等 1 秒 |
| 依赖时钟 | 否 | 是（mtime） |
| 固有问题 | 无 | “慢”与“死”无法区分，仍然存在 |
| 代码量（核心） | 约 50 行 | 约 110 行 |
| 改动范围 | 重写 `ensureUid` 及辅助函数，删锁 | 改 4 个函数，保留结构 |

推荐方案 A。方案 B 适合希望这次 PR 改动最小、结构继续对齐 aliyun-cli `EnsureInstanceID` 的情况。

两种方案共用同一个关键改动：**发布从 `rename`（会覆盖）换成 `link`（不覆盖）**。即使只采纳这一点，锁失效时的后果也会从“互相覆盖”降为“后到者采用先到者的值”。

## 现状

`ensureUid()` 的流程（`uid.ts:178-208`）：

1. 无锁读 `telemetry-uid`，有效则直接返回。
2. 缺失或内容无效时，`mkdir telemetry-uid.lock` 抢锁，最多同步等 1 秒（`LOCK_WAIT_MS`）。
3. 锁内再读一次，仍无效则生成 UUID，写临时文件后 `rename` 覆盖发布。
4. 锁目录的 mtime 超过 2 秒（`LOCK_STALE_MS`）视为持有者已崩溃，其他进程把它 rename 走再删除。
5. 解锁时先 rename 锁目录里的 `owner` 文件，再删锁目录。

## mkdir 锁的问题

根因只有一个：mkdir 锁在持有者崩溃后不会自动释放（flock 会）。于是必须靠“按时间判陈旧并回收”兜底，下面的问题都由此派生。

| # | 问题 | 位置 | 性质 |
| --- | --- | --- | --- |
| L1 | 无法区分“慢”和“死” | `uid.ts:37`、`83-84` | 固有 |
| L2 | 回收时 stat 与 rename 之间有竞态 | `uid.ts:83-87` | 实现缺陷 |
| L3 | unlock 的 owner token 校验没有生效 | `uid.ts:124-131` | 实现缺陷 |
| L4 | 写 `owner` 失败时会删掉别人的锁 | `uid.ts:116-122` | 实现缺陷 |
| L5 | 陈旧判定依赖 mtime | `uid.ts:83` | 固有 |
| L6 | 回收失败会变成每条命令 1 秒的永久惩罚 | `uid.ts:139-140` | 实现缺陷 |
| L7 | 等锁同步阻塞主线程 | `uid.ts:72-75`、`140` | 设计取舍 |

### L1 无法区分“慢”和“死”

陈旧阈值是 2 秒。持有者只是被挂起（笔记本合盖、`SIGSTOP`、杀毒软件扫描、网络盘卡顿）超过 2 秒，也会被当成崩溃，锁被抢走，两个进程同时进入临界区。

加大阈值只能降低概率，不能消除：阈值越大，真崩溃后的恢复越慢。

### L2 回收时 stat 与 rename 之间有竞态

1. B 和 C 同时 stat 到陈旧锁 L1。
2. C 先 rename 走 L1 并删除。
3. D 新建锁 L2，进入临界区。
4. B 执行 rename，拿走的是新鲜的 L2，并删除。
5. E 新建锁 L3。D 和 E 同时在临界区内。

PR 的第三个提交把直接 `rm` 换成了 rename，缩小了窗口，但第 4 步仍然成立。

### L3 unlock 的 owner token 校验没有生效

注释写“只在仍持有 owner token 时删除”，但 unlock 从不读取 token：

```ts
renameSync(join(lockPath, "owner"), join(lockPath, `releasing.${token}`));
rmSync(lockPath, { recursive: true, force: true });
```

`rename` 按路径操作，只要该路径下有名为 `owner` 的文件就会成功，无论是谁写的。A 的锁被抢走、C 新建锁之后，A 退出时 rename 的是 C 的 `owner`，随后删掉 C 的锁。

### L4 写 `owner` 失败时会删掉别人的锁

`mkdirSync` 成功后写 `owner` 失败，代码直接 `rmSync(lockPath, { recursive: true })`。如果失败原因是 `EEXIST`（锁被抢后别人已重建并写入 `owner`），删的是别人的锁；并且这个 `EEXIST` 会被外层 catch 当作“锁忙”继续重试。

### L5 陈旧判定依赖 mtime

- 系统时钟回拨：锁永远不陈旧，或立刻陈旧。
- NFS 等网络盘：mtime 由服务端写入，`Date.now()` 取自客户端，两边时钟不一致会系统性误判。

### L6 回收失败会变成永久惩罚

如果陈旧锁目录 rename 不掉（例如 Windows 上目录内有句柄被占用、目录属主不同），`telemetry-uid` 永远建不出来。此后每条命令都会在遥测上报时同步等满 1 秒才放弃，直到有人手动删掉锁目录。

### L7 等锁同步阻塞主线程

`Atomics.wait` 以 5ms 间隔轮询，最多阻塞 1 秒，期间事件循环停转。`ensureUid` 必须同步（否则 `flushTelemetry` 可能在事件入队前返回），所以只要存在“等待”，就只能是同步阻塞。

## 方案 A：无锁发布（推荐）

### 思路

这里不需要互斥，只需要“不存在才创建，并且内容一出现就是完整的”。`link(2)` 正好提供这个语义：目标已存在时报 `EEXIST`，绝不覆盖；链接建立的瞬间，目标文件的内容就是完整的。

### 流程

1. 读 `telemetry-uid`，有效则返回（与现状相同）。
2. 缺失时生成 UUID，写入同目录临时文件，`linkSync(tmp, path)`。
3. link 成功：自己发布成功，返回自己的 ID。
4. link 报 `EEXIST`：别人先发布了，回到第 1 步读别人的值。
5. 无论成败，删除临时文件。

### 边角情况

- **文件系统不支持硬链接**（FAT/exFAT、部分网络盘）：`link` 报 `EPERM` / `ENOTSUP` 等时，退回 `open "wx"` 排他创建。这条路径上内容不是原子出现的，读者可能读到空文件，所以读到无效内容时先等 20ms 再读一次，仍无效才判定为损坏。
- **文件内容损坏**：把坏文件 rename 到 `telemetry-uid.corrupt.<随机>`，再走正常发布。rename 后检查拿走的内容：如果其实是有效 UUID（别人刚修好，被自己误拿），用 `link` 放回去。
- **发布失败**（只读文件系统、磁盘满）：返回 undefined，不上报 `uid`，与现状一致。

### 代码

替换 `packages/core/src/telemetry/uid.ts` 的全部内容。导出的 `UID_FILE_NAME`、`getUidPath`、`loadUid`、`ensureUid` 签名不变；`UID_LOCK_NAME` 和 `getUidLockPath` 删除。

```ts
import { randomBytes, randomUUID } from "crypto";
import {
  closeSync,
  linkSync,
  mkdirSync,
  openSync,
  readFileSync,
  renameSync,
  unlinkSync,
  writeSync,
} from "fs";
import { join } from "path";
import { getConfigDir } from "../config/paths.ts";

/** On-disk name for the anonymous install id reported to AEM as `uid`. */
export const UID_FILE_NAME = "telemetry-uid";

const UUID_V4 = /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/;

/** link(2) error codes that mean "this filesystem has no hard links". */
const LINK_UNSUPPORTED = new Set(["EPERM", "ENOTSUP", "EOPNOTSUPP", "ENOSYS", "EMLINK"]);

/** Losing a publish race costs one retry; a few rounds cover repeated repairs. */
const MAX_ATTEMPTS = 4;

/** Grace period before treating unparsable content as corrupt (no-hard-link fallback). */
const PARTIAL_WRITE_GRACE_MS = 20;

type ReadResult =
  | { kind: "ok"; id: string }
  | { kind: "missing" }
  | { kind: "invalid" }
  | { kind: "unreadable" };

export function getUidPath(): string {
  return join(getConfigDir(), UID_FILE_NAME);
}

function readUidFile(path: string): ReadResult {
  let raw: string;
  try {
    raw = readFileSync(path, "utf8");
  } catch (error) {
    return (error as NodeJS.ErrnoException).code === "ENOENT"
      ? { kind: "missing" }
      : { kind: "unreadable" };
  }
  const id = raw.trim().toLowerCase();
  return UUID_V4.test(id) ? { kind: "ok", id } : { kind: "invalid" };
}

/** Read-only: returns undefined when missing or invalid; never creates the file. */
export function loadUid(): string | undefined {
  const result = readUidFile(getUidPath());
  return result.kind === "ok" ? result.id : undefined;
}

function sleepSync(ms: number): void {
  Atomics.wait(new Int32Array(new SharedArrayBuffer(4)), 0, 0, ms);
}

function removeQuietly(path: string): void {
  try {
    unlinkSync(path);
  } catch {
    /* already gone */
  }
}

function writeExclusive(path: string, text: string): void {
  const fd = openSync(path, "wx", 0o600);
  try {
    writeSync(fd, text);
  } finally {
    closeSync(fd);
  }
}

/**
 * Atomic create-if-absent. Returns true when this process published,
 * false when another process already had. Never overwrites.
 */
function publishIfAbsent(path: string, id: string): boolean {
  const tmp = `${path}.${randomBytes(6).toString("hex")}.tmp`;
  writeExclusive(tmp, `${id}\n`);
  try {
    linkSync(tmp, path);
    return true;
  } catch (error) {
    const code = (error as NodeJS.ErrnoException).code ?? "";
    if (code === "EEXIST") return false;
    if (!LINK_UNSUPPORTED.has(code)) throw error;
    // No hard links here: exclusive create is still no-clobber, though the
    // content is briefly empty. Readers cover that with PARTIAL_WRITE_GRACE_MS.
    try {
      writeExclusive(path, `${id}\n`);
      return true;
    } catch (fallbackError) {
      if ((fallbackError as NodeJS.ErrnoException).code === "EEXIST") return false;
      throw fallbackError;
    }
  } finally {
    removeQuietly(tmp);
  }
}

/**
 * Move a corrupt file out of the way. If the file we grabbed turns out to be
 * valid (another process repaired it first), put it back without overwriting.
 */
function quarantineCorruptFile(path: string): void {
  const aside = `${path}.corrupt.${randomBytes(6).toString("hex")}`;
  try {
    renameSync(path, aside);
  } catch {
    return; // another process already moved it
  }
  if (readUidFile(aside).kind === "ok") {
    try {
      linkSync(aside, path);
    } catch {
      /* someone else published meanwhile, or no hard links */
    }
  }
  removeQuietly(aside);
}

/**
 * Read or create the persistent anonymous install id (AEM `uid`).
 *
 * Must stay synchronous: remoteSink relies on no await between the call and
 * event enqueue, otherwise flushTelemetry may return before the event is queued.
 *
 * Lock-free: creation is a no-clobber link(2), so concurrent first runs
 * converge on whichever process linked first and the rest read its value.
 * Any failure returns undefined so remoteSink omits `uid`.
 */
export function ensureUid(): string | undefined {
  const path = getUidPath();
  try {
    mkdirSync(getConfigDir(), { recursive: true, mode: 0o700 });
    for (let attempt = 0; attempt < MAX_ATTEMPTS; attempt += 1) {
      let current = readUidFile(path);
      if (current.kind === "invalid") {
        sleepSync(PARTIAL_WRITE_GRACE_MS);
        current = readUidFile(path);
      }
      if (current.kind === "ok") return current.id;
      // Do not touch unreadable paths (permission errors, a directory in the way).
      if (current.kind === "unreadable") return undefined;
      if (current.kind === "invalid") quarantineCorruptFile(path);

      const id = randomUUID();
      if (publishIfAbsent(path, id)) return id;
      // Lost the race: loop and read the winner.
    }
  } catch {
    // Read-only FS, disk full, etc.
  }
  return loadUid();
}
```

### 需要同步修改的地方

- `packages/core/tests/telemetry-uid.test.ts`：
  - 删除 `UID_LOCK_NAME`、`getUidLockPath` 的 import 和相关断言。
  - 删除两个 stale-lock 用例（“eight concurrent reclaimers…”、“repeated stale-lock reclaim bursts…”）。
  - “leaves no temporary files or lock dir behind” 改为断言没有 `.tmp` 和 `.corrupt.` 残留。
  - 其余用例（首次创建、0600、无效内容重建、不可写目录、路径被目录占用、8 进程并发创建、8 进程并发修复）保持不变，应全部通过。
- `docs/agents/telemetry-change.md`：把“并发对齐 aliyun-cli `EnsureInstanceID`…”那一段换成无锁描述（临时文件 + `link` 不覆盖发布；不支持硬链接时退回排他创建；损坏文件隔离后重建）。

### 残余风险

- **修复路径仍有极小的分叉窗口。** 三个进程同时修复同一个损坏文件时，可能有一个进程本次上报的 uid 与磁盘最终值不同。磁盘始终收敛到一个值。由于发布是原子的，文件只有被人手改或磁盘出错才会损坏，这条路径几乎不会走到。
- **不支持硬链接的文件系统上，创建不是原子的。** 靠 20ms 宽限期覆盖；极端慢盘上仍可能把正在写入的文件误判为损坏。
- **崩溃可能留下 `.tmp` 文件。** 每个约 37 字节，不影响功能，不会累积到有意义的量；需要的话可在 `ensureUid` 里顺手清理超过一天的 `.tmp`。

## 方案 B：保留 mkdir 锁并修复

### 改动点

| 问题 | 修法 |
| --- | --- |
| L3 token 校验无效 | token 放进文件名：写 `owner.<token>`，unlock 时 rename `owner.<token>`。锁不是自己的就 `ENOENT`，自然跳过 |
| L4 误删别人的锁 | 写 owner 失败时用 `rmdirSync`（只能删空目录），不再递归删除 |
| L2 回收竞态 | rename 抢走后再 stat 一次抢到的目录（rename 不改目录 mtime）；发现是新鲜的就 rename 回去 |
| L1 误判活锁 | `LOCK_STALE_MS` 从 2 秒调到 10 秒。临界区只有几次小文件 IO，正常耗时在毫秒级 |
| 锁失效时互相覆盖 | 发布改用方案 A 的 `publishIfAbsent`（不覆盖）；发布失败说明别人先发布了，读回并采用别人的值 |

最后一条是安全网：前四条只能缩小锁失效的窗口，加上它之后，即使锁失效，创建路径也不会出现两个不同的已发布值。

### 代码

`readUidFile`、`loadUid`、`getUidPath`、`getUidLockPath`、`sleepSync` 保持 PR 现状不变。`removeQuietly`、`writeExclusive`、`publishIfAbsent`、`LINK_UNSUPPORTED` 与方案 A 完全相同，此处不重复。下面是需要替换的部分（原 `publishUid` 删除）：

```ts
const LOCK_WAIT_MS = 1000;

/**
 * The critical section is a handful of tiny file operations, so a lock older
 * than this belongs to a crashed holder. Kept well above LOCK_WAIT_MS on purpose:
 * a waiter gives up and omits `uid` rather than stealing a possibly live lock.
 */
const LOCK_STALE_MS = 10_000;

/**
 * Claim a stale lock by renaming it away, then verify what was actually claimed.
 * rename keeps the directory's mtime, so a fresh mtime means we grabbed a lock
 * that replaced the stale one between our stat and rename: give it back.
 */
function tryReclaimStaleLock(lockPath: string): void {
  try {
    if (Date.now() - statSync(lockPath).mtimeMs <= LOCK_STALE_MS) return;
    const claimed = `${lockPath}.stale.${randomBytes(6).toString("hex")}`;
    try {
      renameSync(lockPath, claimed);
    } catch {
      return; // another process already claimed or removed it
    }
    let fresh = false;
    try {
      fresh = Date.now() - statSync(claimed).mtimeMs <= LOCK_STALE_MS;
    } catch {
      /* treat as stale */
    }
    if (fresh) {
      try {
        renameSync(claimed, lockPath);
        return;
      } catch {
        // A newer lock already exists; the holder we displaced is still safe
        // because publishIfAbsent never overwrites.
      }
    }
    try {
      rmSync(claimed, { recursive: true, force: true });
    } catch {
      /* ignore */
    }
  } catch {
    /* ignore */
  }
}

/**
 * Acquire an exclusive mkdir lock, waiting up to timeoutMs.
 * Returns an unlock function, or undefined when the lock stays busy.
 */
function acquireUidLock(timeoutMs: number): (() => void) | undefined {
  const lockPath = getUidLockPath();
  const deadline = Date.now() + timeoutMs;

  for (;;) {
    let created = false;
    try {
      mkdirSync(lockPath);
      created = true;
    } catch (error) {
      if ((error as NodeJS.ErrnoException).code !== "EEXIST") throw error;
    }

    if (created) {
      // The token is part of the file name, so unlock can only ever match our own lock.
      const token = randomBytes(8).toString("hex");
      const owner = join(lockPath, `owner.${token}`);
      try {
        writeExclusive(owner, "");
      } catch (error) {
        try {
          rmdirSync(lockPath); // removes only an empty dir, never someone else's lock
        } catch {
          /* ignore */
        }
        throw error;
      }
      return () => {
        try {
          // ENOENT unless the lock at this path is still ours.
          renameSync(owner, join(lockPath, `releasing.${token}`));
          rmSync(lockPath, { recursive: true, force: true });
        } catch {
          /* lock was reclaimed or already gone */
        }
      };
    }

    tryReclaimStaleLock(lockPath);
    if (Date.now() >= deadline) return undefined;
    sleepSync(Math.min(5, Math.max(0, deadline - Date.now())));
  }
}

export function ensureUid(): string | undefined {
  const path = getUidPath();
  const current = readUidFile(path);
  if (current.kind === "ok") return current.id;
  // Do not touch unreadable paths (permission errors, a directory in the way).
  if (current.kind === "unreadable") return undefined;

  let unlock: (() => void) | undefined;
  try {
    mkdirSync(getConfigDir(), { recursive: true, mode: 0o700 });
    unlock = acquireUidLock(LOCK_WAIT_MS);
    if (!unlock) return loadUid();

    const again = readUidFile(path);
    if (again.kind === "ok") return again.id;
    if (again.kind === "unreadable") return undefined;
    if (again.kind === "invalid") removeQuietly(path); // repair is serialized by the lock

    const id = randomUUID();
    if (publishIfAbsent(path, id)) return id;
    // The lock was broken and another process published first: adopt its value.
    return loadUid();
  } catch {
    return loadUid();
  } finally {
    unlock?.();
  }
}
```

import 需要相应调整：增加 `linkSync`、`rmdirSync`，去掉 `writeFileSync`。

### 需要同步修改的地方

- `packages/core/tests/telemetry-uid.test.ts`：
  - 伪造陈旧锁的用例里，`utimesSync` 的时间要早于新的 10 秒阈值。现在用的是 `Date.now() - 10_000`，正好落在边界上，改为 `Date.now() - 60_000`。
  - 伪造的 owner 文件名从 `owner` 改为 `owner.dead`（不改也能通过，只是与新约定保持一致）。
  - 建议新增一个用例：预置一个 mtime 新鲜的锁目录，断言 `ensureUid()` 在约 1 秒后返回 undefined，且锁目录仍然存在（不抢活锁）。
- `docs/agents/telemetry-change.md`：更新锁描述（owner 文件名带 token、回收后复查、阈值 10 秒、发布不覆盖）。

### 残余风险

- **L1 仍在。** 持有者被挂起超过 10 秒，锁仍会被回收。此时创建路径靠不覆盖发布兜底，不会分叉；修复路径（先删坏文件再发布）在锁失效时仍可能分叉。
- **L5 仍在。** 时钟回拨或网络盘时钟不一致仍会误判。
- **L6 仍在。** 陈旧锁回收不掉时，每条命令继续同步等 1 秒。可以再加一条缓解：等锁超时后在配置目录写一个标记，后续进程看到标记直接跳过等待，但这会进一步增加复杂度。
- **崩溃后 10 秒内的运行拿不到 uid。** 等锁上限 1 秒小于陈旧阈值 10 秒，这段时间内的命令会省略 `uid` 上报，之后自动恢复。

## 怎么选

- 选 A：如果目标是正确性和可维护性。它没有锁的固有问题，没有阻塞，代码量少一半。代价是不再与 aliyun-cli 的实现结构对应，并且需要重写而不是修补。
- 选 B：如果希望 PR #314 以最小改动合入，并保留与 aliyun-cli `EnsureInstanceID` 对应的结构。代价是留下 L1、L5、L6 三个已知问题。
- 折中：这次先合 B，后续再换 A。B 里已经引入了 `publishIfAbsent`，换到 A 时只需删除锁相关的三个函数并调整 `ensureUid`。

## 验证情况

两种方案都写成了独立的 `.mjs` 原型（逻辑与上文代码一致，路径改为参数传入），用 16 个并发子进程、每种场景 60 轮做了压测。环境：Node v22.23.4，Linux 6.18，本地磁盘。

| 场景 | 方案 A | 方案 B |
| --- | --- | --- |
| 文件缺失，16 进程同时首次运行 | 60/60 收敛 | 60/60 收敛 |
| 文件内容无效，16 进程同时修复 | 60/60 收敛 | 60/60 收敛 |
| 预置陈旧锁（60 秒前） | 不适用 | 60/60 收敛 |
| 预置陈旧锁 + 文件内容无效 | 不适用 | 60/60 收敛 |

“收敛”指所有进程返回的 uid 都等于磁盘最终值，没有进程返回空值，目录里除 `telemetry-uid` 外没有残留文件。

没有验证的部分：

- 上文的 TypeScript 代码是从原型改写的，没有在 `modelstudioai/cli` 仓库里编译，也没有跑该仓库的测试套件（审查环境执行该仓库代码被权限策略拒绝）。
- 只在 Linux 本地磁盘上测过。Windows、macOS、不支持硬链接的文件系统、网络盘都没有实测；`LINK_UNSUPPORTED` 的错误码集合来自文档，未逐个平台确认。
- 压测覆盖不到 L1、L2 这类需要精确时序才能触发的窗口，相关结论来自读代码推演。
- 没有修改 `modelstudioai/cli` 的代码。
