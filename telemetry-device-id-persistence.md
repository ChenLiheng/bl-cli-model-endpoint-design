# 遥测 device_id 本地持久化：方案与改动清单

> 日期：2026-11-10 · 基线：`modelstudioai/cli` main `fa8330c2` · 状态：可实施方案（代码未在仓库内编译运行，见“验证情况”）

## 结论

把遥测 `device_id` 从“每次进程按网卡 MAC 计算的 MD5”改为“首次上报时生成、落盘到 `~/.bailian/telemetry-device-id` 的随机 UUID v4”。

- 改动集中在 `packages/core/src/telemetry/`：新增 1 个文件，修改 `sink.ts`，再补测试和文档。
- 不修改打包进仓库的 `lib/remote-telemetry/tracker.cjs`。它的构造函数会让传入的 `device_id` 覆盖默认值（见“现状”）。
- 遥测关闭（`DO_NOT_TRACK=1` 或 `telemetry: false`）时，不生成、不读取这个文件，也不再构造 tracker、不扫描网卡。

## 现状

### device_id 怎么来的

`packages/core/lib/remote-telemetry/tracker.cjs:1067-1104`：

```js
function E() { /* 遍历 os.networkInterfaces()，返回第一个非全零 MAC；都没有则返回 "00:00:00:00:00:00" */ }
var q = { os, os_version, app_name: "node", app_version, device_id: md5(E()), platform: "node" };
function H(e) { _.call(this, V(V({}, q), e)); ... }   // 传入的 config 覆盖 q
```

- 基类构造函数（`tracker.cjs:689-699`）把这份合并后的配置并进 `this._config`。
- `_getSendConfig()`（`tracker.cjs:750-765`）把 `_config` 里所有非空字段拼进上报的 URL 参数。

所以 `new Tracker({ pid, env, device_id })` 就能替换掉 MAC 哈希，并且会随每条事件上报。

### 问题

| 问题 | 后果 |
| --- | --- |
| 容器、CI 每次分到的 MAC 不同 | 每个容器都算一台新设备，设备数被高估 |
| 网卡枚举顺序会变（VPN、Docker 网桥、虚拟机网卡、Wi-Fi 与有线切换） | 同一台机器的 ID 漂移 |
| 找不到可用网卡时退回固定值 | 所有这类机器共用 `md5("00:00:00:00:00:00")`，被合并成一台 |
| MAC 只有 48 位，厂商前缀公开 | MD5 可被暴力反推，属于准个人信息 |
| 本地 `telemetry.jsonl` 不含 device_id | 本地记录和远端数据对不上 |
| `sink.ts` 在模块加载时就构造 Tracker | 遥测关闭时也会扫描网卡、算 MD5（虽然不上报） |

## 参考实现

| | aliyun-cli（Go） | qwencloud-cli（TS） | arkcli（Go，只有二进制） |
| --- | --- | --- | --- |
| 生成 | `crypto/rand` 生成 UUID v4 | `crypto.randomUUID()` | 对机器 ID 做哈希（`tea.hashToDID(hostinfo.MachineID())`） |
| 落盘 | `~/.aliyun/telemetry-instance-id`，0600 | `~/.qwencloud/device`，0600 | 未发现落盘证据 |
| 并发 | 无锁读；缺失时加文件锁、二次读、原子写 | 无保护 | — |
| 损坏 | 校验格式，不合法就报错并提示 purge | 空内容才重建 | — |
| 用户控制 | `aliyun configure telemetry purge`；`ALIBABA_CLOUD_CLI_TELEMETRY_DISABLE` | 无 | `ARKCLI_OBSERVABILITY_ENABLED` 等 |
| 用途 | 遥测 `instance_id` | OAuth 设备流 `client_id`（无遥测） | 遥测 DID |

来源：

- aliyun-cli：`sysconfig/telemetry/instance_id.go`、`file_io.go`。
- qwencloud-cli：`src/auth/client-id.ts`。
- arkcli：GitHub 仓库 `volcengine/ark-cli` 里没有源码。我对 npm 包 `@volcengine/ark-cli-linux-x64@1.0.48` 的二进制做了 `strings` 静态分析（没有运行），看到 `internal/tea/did.go`、`hostinfo.platformMachineID` 和 `/etc/machine-id`。“未落盘”是推测。

本方案取 aliyun 的思路（随机 UUID + 原子发布 + 格式校验），并做了一处差异化：**文件损坏时静默重建**，不报错。原因是遥测不能影响主流程，也不应要求用户手动修复。

## 目标与非目标

目标：

1. 同一 HOME（或同一 `BAILIAN_CONFIG_DIR`）下，ID 跨进程、跨版本、跨网络环境保持稳定。
2. 多个 `bl` 进程同时首次运行时，最终收敛到同一个 ID。
3. 任何失败（只读文件系统、权限不足、磁盘满）都静默退回原来的 MAC 哈希，不影响命令执行和退出码。
4. 遥测关闭时，不产生任何文件和网卡扫描。

非目标：

- 不追求跨容器稳定。每个新容器、新 CI runner 产生新 ID，是任何本地方案都绕不开的；CI 流量用已有的 `callerType` 区分。
- 不做跨机器去重，也不和账号绑定。
- 这一期不加 purge 命令（见“二期”）。

## 设计

### 文件

| 项 | 值 |
| --- | --- |
| 路径 | `join(getConfigDir(), "telemetry-device-id")`，即默认 `~/.bailian/telemetry-device-id`，并遵循 `BAILIAN_CONFIG_DIR` |
| 内容 | 小写 UUID v4，末尾一个 `\n` |
| 权限 | 文件 0600；目录 0700（与 `ensureConfigDir` 一致；Windows 上忽略） |
| 共享 | `bl`、`kscli`、`aliyun bl` 共用同一配置目录，因此共用同一个 ID。这和现在的 MAC 哈希行为一致，入口用 `commandPrefix` 区分 |

不写进 `config.json`。原因：

- `config.json` 的写入链路有“损坏后清空”“并发丢写”的已知问题（见 [bailian-cli-code-review.md](bailian-cli-code-review.md) 的 B1、B2）。
- `config ui` 和 `config set` 会整文件改写。设备 ID 不应该跟用户配置的生命周期绑在一起。

### 生命周期

```text
命令执行 → trackCommandExecution
  ├─ settings.telemetry=false → 直接执行命令，不触碰遥测（现状，不变）
  └─ 命令结束 finally
       ├─ localSink(event)    （异步）写 telemetry.jsonl，附带 deviceId（只读，不创建）
       └─ remoteSink(event)   （同步执行到入队为止）
            └─ 首次调用：initRemote()
                 ├─ ensureDeviceId()   读文件；缺失或损坏时生成并原子发布
                 └─ new Tracker({ pid, env, device_id })
```

### 为什么必须用同步 IO

`tracker.ts` 在 `finally` 里不等待地调用 `remoteSink(event)`，随后 `create-cli.ts:316/361` 执行 `await flushTelemetry(1000)`。`flushTelemetry` 只会排空**已经入队**的事件。

如果 `remoteSink` 在调用 `remoteSendEvent` 之前 `await` 了任何异步文件操作，事件就可能在 flush 之后才入队，于是被丢掉。所以 `ensureDeviceId` 必须用 `fs` 的同步 API，让 `remoteSink` 从被调用到事件入队都在同一个同步段内完成。读一个 37 字节的文件，开销在亚毫秒级。

### 并发与原子性

| 场景 | 做法 | 结果 |
| --- | --- | --- |
| 文件已存在且合法（稳态） | 只读，不加锁 | 零写入 |
| 文件不存在 | 写临时文件，再用 `linkSync(tmp, path)` 发布。目标存在时硬链接以 `EEXIST` 失败，相当于原子的“不存在才创建” | 先到者胜，后到者读回胜者的 ID |
| 文件系统不支持硬链接（`EPERM`、`ENOTSUP`、`ENOSYS`、`EXDEV`，常见于部分网络盘和 FAT） | 退回 `renameSync(tmp, path)` | 后写者胜；极端并发下，第一条事件可能带一个随后被覆盖的 ID |
| 文件存在但内容不合法 | `renameSync(tmp, path)` 原子覆盖 | 同上 |
| 文件不可读（`EACCES` 等，非 `ENOENT`） | 不覆盖，返回 `undefined` | 退回 MAC 哈希，不破坏用户文件 |
| 任何写入失败 | 吞掉异常，最后再读一次 | 读到就用，读不到就退回 MAC 哈希 |

不用文件锁。遥测场景下“首次并发”窗口极小，`link` 的“不存在才创建”已经能保证收敛，没必要再引入锁文件和超时逻辑。

## 代码改动

### 1. 新增 `packages/core/src/telemetry/device-id.ts`

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

export const DEVICE_ID_FILE_NAME = "telemetry-device-id";

const UUID_V4 = /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/;

/** 硬链接不可用时退回 rename 的错误码。 */
const NO_HARDLINK = new Set(["EPERM", "ENOTSUP", "ENOSYS", "EXDEV", "EOPNOTSUPP"]);

type ReadResult =
  | { kind: "ok"; id: string }
  | { kind: "missing" }
  | { kind: "invalid" }
  | { kind: "unreadable" };

export function getDeviceIdPath(): string {
  return join(getConfigDir(), DEVICE_ID_FILE_NAME);
}

function readDeviceIdFile(path: string): ReadResult {
  let raw: string;
  try {
    raw = readFileSync(path, "utf8");
  } catch (err) {
    return (err as NodeJS.ErrnoException).code === "ENOENT"
      ? { kind: "missing" }
      : { kind: "unreadable" };
  }
  const id = raw.trim().toLowerCase();
  return UUID_V4.test(id) ? { kind: "ok", id } : { kind: "invalid" };
}

/** 只读：不存在或不合法时返回 undefined，从不创建文件。 */
export function loadDeviceId(): string | undefined {
  const result = readDeviceIdFile(getDeviceIdPath());
  return result.kind === "ok" ? result.id : undefined;
}

/**
 * 读取或创建持久化的遥测设备 ID。
 *
 * 必须保持同步：remoteSink 依赖“调用到事件入队”之间没有 await，
 * 否则 flushTelemetry 可能在事件入队前返回。
 * 任何失败都返回 undefined，由 tracker 退回内置的 MAC 哈希。
 */
export function ensureDeviceId(): string | undefined {
  const path = getDeviceIdPath();
  const current = readDeviceIdFile(path);
  if (current.kind === "ok") return current.id;
  // 读不了的文件不覆盖：可能是权限问题，覆盖会破坏用户环境。
  if (current.kind === "unreadable") return undefined;

  const tmp = `${path}.${randomBytes(6).toString("hex")}.tmp`;
  try {
    mkdirSync(getConfigDir(), { recursive: true, mode: 0o700 });
    const fd = openSync(tmp, "wx", 0o600);
    try {
      writeSync(fd, `${randomUUID()}\n`);
    } finally {
      closeSync(fd);
    }
    if (current.kind === "missing") {
      publishIfAbsent(tmp, path);
    } else {
      renameSync(tmp, path); // invalid：原子覆盖
    }
  } catch {
    // 只读文件系统、磁盘满等：静默，下面再读一次。
  } finally {
    try {
      unlinkSync(tmp); // link 成功后 tmp 仍在；rename 后 tmp 已不存在，忽略 ENOENT
    } catch {
      /* ignore */
    }
  }

  return loadDeviceId();
}

/** 原子的“不存在才创建”：并发首次运行时先到者胜。 */
function publishIfAbsent(tmp: string, path: string): void {
  try {
    linkSync(tmp, path);
  } catch (err) {
    const code = (err as NodeJS.ErrnoException).code ?? "";
    if (code === "EEXIST") return; // 别的进程已发布，保留它的
    if (NO_HARDLINK.has(code)) {
      renameSync(tmp, path); // 不支持硬链接：退回后写者胜
      return;
    }
    throw err;
  }
}
```

### 2. 修改 `packages/core/src/telemetry/sink.ts`：延迟构造 Tracker

把模块顶层的 `try { const client = new Tracker(...) ... } catch {}` 整段换成 `initRemote()`，并在 `remoteSink` 里调用它。monkey-patch `send` 的逻辑原样保留，只是挪进函数里。

```ts
import { ensureDeviceId, loadDeviceId } from "./device-id.ts";

let remoteSendEvent: ((name: string, opts: Record<string, unknown>) => void) | undefined;
let remoteClient: any = undefined;
let remoteInitialized = false;

/**
 * 首次远端上报时才构造 tracker：遥测关闭时既不扫描网卡，也不创建设备 ID 文件。
 * 全程同步，见 device-id.ts 中 ensureDeviceId 的说明。
 */
function initRemote(): void {
  if (remoteInitialized) return;
  remoteInitialized = true;
  try {
    const deviceId = ensureDeviceId();
    // env 传给 Tracker 顶层 config 后会进 URL 查询参数, AEM 后台 "环境" 列
    // (原始字段 env) 直接读它, 值域 prod/pre/daily/dev。我们目前只用 prod/dev。
    // device_id 覆盖 tracker 内置的 md5(MAC)；缺失时沿用内置值。
    const client = new Tracker({
      pid: "bailian-cli-node",
      env: detectEnv(),
      ...(deviceId ? { device_id: deviceId } : {}),
    });
    const originalSend = client.send.bind(client);
    client.send = function (payload: string) {
      // ……原有的 AbortController / deadline / inflightSends 逻辑，一字不改……
    };
    remoteClient = client;
    remoteSendEvent = client.use(EventPlugin);
  } catch {
    // 埋点逻辑任何异常都不能影响 CLI 主流程
  }
}

export async function remoteSink(event: TrackingEvent): Promise<void> {
  try {
    initRemote();
    if (!remoteSendEvent) return;
    remoteSendEvent(event.command, buildRemoteAemOptions(event));
  } catch {
    // 埋点逻辑任何异常都不能影响 CLI 主流程
  }
}
```

`flushTelemetry` 不用改：从没上报过时 `remoteClient` 为 `undefined`，`inflightSends` 为空，直接返回。

### 3. 修改 `localSink`：本地日志带上 deviceId

```ts
export async function localSink(event: TrackingEvent): Promise<void> {
  try {
    await ensureConfigDir();
    // ……原有的 5 MB 轮转逻辑不变……
    const deviceId = loadDeviceId(); // 只读；首次运行时由 remoteSink 创建
    appendFileSync(path, JSON.stringify(deviceId ? { ...event, deviceId } : event) + "\n", {
      mode: 0o600,
    });
  } catch {
    // 埋点逻辑任何异常都不能影响 CLI 主流程
  }
}
```

`deviceId` 只加在本地行里，不放进 `TrackingEvent`。原因是 `buildRemoteAemOptions` 会把事件其余字段整体放进 `ext`，如果放进事件，就会和 URL 参数里的 `device_id` 重复上报。

首次运行时，`localSink` 在 `await ensureConfigDir()` 之后才读文件；这时 `remoteSink` 已经同步创建好了文件，所以第一行通常也能带上 ID。这一点不做保证，也不需要保证。

### 4. 修改 `packages/core/lib/remote-telemetry/tracker.d.cts`（只在采用 D2 时需要）

```ts
declare class RemoteTracker {
  constructor(config: { pid: string; debug?: boolean; [key: string]: unknown });
  use(plugin: unknown): (...args: any[]) => void;
  send(gokey: string, options?: { signal?: AbortSignal }): Promise<void>;
  getConfig(key?: string): unknown;
}
```

现有声明的索引签名已经允许传 `device_id`，第 1 至 3 步不需要改它。

### 5. 过渡期双写 legacy_device_id（可选，见决策 D2）

上线后，所有老用户都会换成新 ID，设备数会出现一次断点。如果看板需要把新旧 ID 关联起来，可以在 `initRemote` 里先保留 tracker 内置的 MAC 哈希：

```ts
const client = new Tracker({ pid: "bailian-cli-node", env: detectEnv() });
const config = (client as { _config?: Record<string, unknown> })._config;
if (deviceId && config) {
  config.legacy_device_id = config.device_id; // 内置 md5(MAC)
  config.device_id = deviceId;
}
```

- 这样写依赖 `_config` 私有字段。`sink.ts` 已经在依赖私有的 `_sendAll`，风险等级相同。
- 必须在第一次 `send` 之前完成，因为 `_getSendConfig` 每次发送时读取 `_config`。
- 建议保留 2 至 3 个版本后删除，并在代码里加 `// TODO(remove after vX.Y)`。

### 6. 测试

#### 新增 `packages/core/tests/telemetry-device-id.test.ts`

```ts
import { mkdirSync, readFileSync, statSync, writeFileSync, chmodSync } from "node:fs";
import { join } from "node:path";
import { afterEach, beforeEach, expect, test } from "vite-plus/test";
import { enterIsolatedHome, type IsolatedHomeHandle } from "./helpers/isolation.ts";
import {
  DEVICE_ID_FILE_NAME,
  ensureDeviceId,
  getDeviceIdPath,
  loadDeviceId,
} from "../src/telemetry/device-id.ts";

const UUID_V4 = /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/;
let iso: IsolatedHomeHandle;

beforeEach(() => {
  iso = enterIsolatedHome();
});
afterEach(() => iso.restore());

test("首次调用生成 UUID v4 并落盘，再次调用返回同一个值", () => {
  expect(loadDeviceId()).toBeUndefined();
  const id = ensureDeviceId();
  expect(id).toMatch(UUID_V4);
  expect(readFileSync(getDeviceIdPath(), "utf8")).toBe(`${id}\n`);
  expect(ensureDeviceId()).toBe(id);
  expect(loadDeviceId()).toBe(id);
});

test("文件路径遵循 BAILIAN_CONFIG_DIR", () => {
  ensureDeviceId();
  expect(getDeviceIdPath()).toBe(join(iso.configDir, DEVICE_ID_FILE_NAME));
});

test.skipIf(process.platform === "win32")("文件权限为 0600", () => {
  ensureDeviceId();
  expect(statSync(getDeviceIdPath()).mode & 0o777).toBe(0o600);
});

test("配置目录不存在时自动创建", () => {
  process.env.BAILIAN_CONFIG_DIR = join(iso.home, "nested", "cfg");
  expect(ensureDeviceId()).toMatch(UUID_V4);
});

test("内容不合法时重建", () => {
  writeFileSync(getDeviceIdPath(), "not-a-uuid\n");
  const id = ensureDeviceId();
  expect(id).toMatch(UUID_V4);
  expect(readFileSync(getDeviceIdPath(), "utf8").trim()).toBe(id);
});

test("大写或前后空白的合法 UUID 归一化后接受，不重写", () => {
  const upper = "  0F8FAD5B-D9CB-469F-A165-70867728950E \n";
  writeFileSync(getDeviceIdPath(), upper);
  expect(ensureDeviceId()).toBe("0f8fad5b-d9cb-469f-a165-70867728950e");
  expect(readFileSync(getDeviceIdPath(), "utf8")).toBe(upper);
});

test("已有其他进程发布的文件时保留它（模拟 link EEXIST）", () => {
  const existing = "0f8fad5b-d9cb-469f-a165-70867728950e";
  writeFileSync(getDeviceIdPath(), `${existing}\n`);
  expect(ensureDeviceId()).toBe(existing);
});

test.skipIf(process.platform === "win32" || process.getuid?.() === 0)(
  "目录不可写时返回 undefined 且不抛错",
  () => {
    chmodSync(iso.configDir, 0o500);
    try {
      expect(ensureDeviceId()).toBeUndefined();
    } finally {
      chmodSync(iso.configDir, 0o700);
    }
  },
);

test("路径被目录占用（不可读为文件）时不覆盖，返回 undefined", () => {
  mkdirSync(getDeviceIdPath());
  expect(ensureDeviceId()).toBeUndefined();
  expect(statSync(getDeviceIdPath()).isDirectory()).toBe(true);
});

test("不留下临时文件", async () => {
  ensureDeviceId();
  const { readdirSync } = await import("node:fs");
  expect(readdirSync(iso.configDir).filter((name) => name.endsWith(".tmp"))).toEqual([]);
});
```

最后一个“路径被目录占用”的用例：在 Linux 和 macOS 上，`readFileSync` 读目录会报 `EISDIR`，落入 `unreadable` 分支；Windows 上的错误码可能不同。如果 CI 在 Windows 上失败，就给这个用例加 `skipIf(win32)`。

#### 新增多进程并发用例（可放在同一文件）

```ts
import { spawn } from "node:child_process";

test("8 个进程同时首次运行，收敛到同一个 ID", async () => {
  const script = `
    import { ensureDeviceId } from ${JSON.stringify(new URL("../src/telemetry/device-id.ts", import.meta.url).href)};
    process.stdout.write(String(ensureDeviceId()));
  `;
  const run = () =>
    new Promise<string>((resolve, reject) => {
      const child = spawn(process.execPath, ["--experimental-strip-types", "--input-type=module", "-e", script], {
        env: { ...process.env },
      });
      let out = "";
      child.stdout.on("data", (chunk) => (out += chunk));
      child.on("error", reject);
      child.on("close", () => resolve(out));
    });
  const ids = await Promise.all(Array.from({ length: 8 }, run));
  const finalId = readFileSync(getDeviceIdPath(), "utf8").trim();
  expect(new Set(ids)).toEqual(new Set([finalId]));
});
```

子进程怎么加载 `.ts`，要按仓库现有的 e2e 做法调整（参考 `packages/e2e/src/runner.ts` 的 `runNodeMain`）。上面的 `--experimental-strip-types` 只是示意。如果 `device-id.ts` 依赖的 `config/paths.ts` 能被 strip-types 直接加载，就可以这样用。

#### 修改 `packages/core/tests/telemetry-sink.test.ts`

`remoteSink` 现在会创建文件，所以这个测试文件必须隔离配置目录，否则会写到开发者真实的 `~/.bailian`：

```ts
import { enterIsolatedHome, type IsolatedHomeHandle } from "./helpers/isolation.ts";

let iso: IsolatedHomeHandle;
beforeEach(() => {
  iso = enterIsolatedHome();
});
afterEach(() => {
  iso.restore();
  // ……原有的清理逻辑……
});
```

并在 mock 的 Tracker 上记录构造参数，补两条用例：

```ts
const fixture = vi.hoisted(() => ({
  signals: [] as Array<AbortSignal | undefined>,
  complete: false,
  configs: [] as Array<Record<string, unknown>>,
}));
vi.mock("../lib/remote-telemetry/tracker.cjs", () => ({
  default: class {
    constructor(config: Record<string, unknown>) {
      fixture.configs.push(config);
    }
    // ……send / use / _sendAll 不变……
  },
}));

test("首次 remoteSink 才构造 tracker，并传入持久化的 device_id", async () => {
  const { remoteSink } = await import("../src/telemetry/sink.ts");
  expect(fixture.configs).toHaveLength(0);
  await remoteSink({ command: "fixture" } as TrackingEvent);
  await remoteSink({ command: "fixture" } as TrackingEvent);
  expect(fixture.configs).toHaveLength(1);
  const { loadDeviceId } = await import("../src/telemetry/device-id.ts");
  expect(fixture.configs[0]).toMatchObject({ pid: "bailian-cli-node", device_id: loadDeviceId() });
});

test("只 import sink 不上报时，不创建设备 ID 文件", async () => {
  await import("../src/telemetry/sink.ts");
  const { existsSync } = await import("node:fs");
  const { getDeviceIdPath } = await import("../src/telemetry/device-id.ts");
  expect(existsSync(getDeviceIdPath())).toBe(false);
});
```

注意 `afterEach` 里已有 `vi.resetModules()`，所以每个用例重新 import 时，`remoteInitialized` 都会重置。

#### 修改 `packages/core/tests/telemetry-tracker.test.ts`

补一条：`settings.telemetry=false` 时执行 `trackCommandExecution`，结束后 `telemetry-device-id` 不存在。这条验证目标 4 的端到端路径。

#### 真实 tracker 集成用例（加在 `telemetry-send.test.ts`）

这个文件用的是真实的 `tracker.cjs`，适合验证覆盖是否生效：

```ts
test("构造参数中的 device_id 覆盖内置 MAC 哈希并进入上报参数", () => {
  const id = "0f8fad5b-d9cb-469f-a165-70867728950e";
  const tracker = new Tracker({ pid: "fixture-pid", device_id: id }) as unknown as {
    getConfig(key: string): unknown;
    _getSendConfig(): Record<string, unknown>;
  };
  expect(tracker.getConfig("device_id")).toBe(id);
  expect(tracker._getSendConfig().device_id).toBe(id);
});
```

### 7. 文档

`docs/agents/telemetry-change.md`：

- 第 123 行“底层 Node tracker 还会附加……由本机网络标识计算的 MD5 `device_id`”改为：

  > 底层 Node tracker 还会附加公共设备字段：OS 类型/版本、Node 应用名与版本、平台，以及 `device_id`。`device_id` 来自 `~/.bailian/telemetry-device-id`（遵循 `BAILIAN_CONFIG_DIR`）：首次远端上报时生成随机 UUID v4，以 0600 权限原子发布；生成失败时退回 tracker 内置的 MAC MD5。遥测关闭时不生成该文件。本地 `telemetry.jsonl` 每行附带 `deviceId`。

- 第 121 至 122 行的“事件同时写入两处”下，补一行设备 ID 文件的说明。
- “C. 调整 AEM 事件”清单加两项：
  - [ ] `remoteSink` 从调用到 `remoteSendEvent` 入队之间不得出现 `await`（flush 只排空已入队事件）
  - [ ] 测试中调用 `remoteSink` 的用例必须使用 `enterIsolatedHome()`，避免写入真实配置目录
- “完成后自查”命令补上 `packages/core/tests/telemetry-device-id.test.ts`。

`CHANGELOG.md` 和 `CHANGELOG.zh.md`，放在下个版本的 Changed 小节：

> - **Telemetry device identifier** — the anonymous telemetry device ID is now a random UUID stored in `~/.bailian/telemetry-device-id` instead of a hash of the network MAC address, so it stays stable across network changes and no longer derives from hardware identifiers. Delete the file to reset it; `DO_NOT_TRACK=1` or `telemetry: false` still disables telemetry entirely.

> - **遥测设备标识** —— 匿名遥测设备 ID 改为保存在 `~/.bailian/telemetry-device-id` 的随机 UUID，不再由网卡 MAC 地址哈希得到；网络环境变化时保持稳定，也不再来自硬件标识。删除该文件即可重置；`DO_NOT_TRACK=1` 或 `telemetry: false` 仍可完全关闭遥测。

## 数据侧影响

| 影响 | 说明 | 应对 |
| --- | --- | --- |
| 升级当天所有老用户换新 ID | 设备数一次性断点，新增设备会虚高一个周期 | 提前告知看板负责人；或采用 D2 过渡期双写 |
| 容器和 CI 仍是一容器一 ID | 和现在一样，没有变好也没有变差 | 用 `callerType=script` 和 `env` 过滤 |
| 原本共用全零 MAC 哈希的机器被拆开 | 设备数会上升，这是修正而不是异常 | 在发版说明里给数据同学写明 |
| `bl`、`kscli`、`aliyun bl` 共用 ID | 和现在一致 | 继续用 `commandPrefix` 区分入口 |

## 待决策

| # | 问题 | 建议 |
| --- | --- | --- |
| D1 | 文件放 `~/.bailian/` 还是 OS 缓存目录 | 放 `~/.bailian/`，和 `telemetry.jsonl` 同目录，遵循 `BAILIAN_CONFIG_DIR`，测试隔离方便 |
| D2 | 是否过渡期双写 `legacy_device_id` | 看板需要连续性就做（第 5 步），保留 2 至 3 个版本；不需要就跳过，代码更简单 |
| D3 | 关闭遥测时是否删除已有文件 | 不删。和 aliyun 一致：关闭只停止采集，重置用二期的 purge 或手动删除 |
| D4 | 是否提供环境变量覆盖 ID（如 `BAILIAN_TELEMETRY_DEVICE_ID`），让 CI 固定 ID | 不提供。CI 用 `callerType` 区分即可，加覆盖口子容易被滥用来伪造设备数 |

## 二期（不在本次范围）

- `bl config telemetry purge`：删除 `telemetry-device-id` 和 `telemetry.jsonl`，不改变 `telemetry` 开关。实现时：
  - 在 `packages/cli/src/commands.ts` 注册路径；
  - 命令本身用 `auth: "none"`；
  - 不需要 `risk`（只删本地匿名数据）；
  - 删除前不需要加锁：正在运行的进程最多再用旧 ID 上报一次。
- `bl config show` 或 `bl doctor` 显示当前设备 ID 和文件路径，方便排查。

## 实施清单

1. 新增 `packages/core/src/telemetry/device-id.ts`（第 1 步）。
2. 改 `sink.ts`：延迟构造 tracker、传入 `device_id`、`localSink` 附带 `deviceId`（第 2、3 步）。
3. 定 D2；需要就加第 4、5 步。
4. 测试：新增 `telemetry-device-id.test.ts`；`telemetry-sink.test.ts` 加隔离和 2 条用例；`telemetry-tracker.test.ts` 加 1 条；`telemetry-send.test.ts` 加 1 条（第 6 步）。
5. 文档：`docs/agents/telemetry-change.md`、两份 CHANGELOG（第 7 步）。
6. 自查：

   ```bash
   vp check
   ```

   ```bash
   vp test packages/core/tests/telemetry-device-id.test.ts packages/core/tests/telemetry-sink.test.ts packages/core/tests/telemetry-tracker.test.ts packages/core/tests/telemetry-send.test.ts packages/runtime/tests/telemetry-stage.test.ts
   ```

7. 手动验证（隔离 HOME）：
   - `HOME=$(mktemp -d) bl config show` 执行两次，`~/.bailian/telemetry-device-id` 内容不变，`telemetry.jsonl` 每行的 `deviceId` 与之相同；
   - `DO_NOT_TRACK=1` 时该文件不生成；
   - 将配置目录设为只读后命令照常成功，退出码不变。

## 验证情况

- 以下几点都是静态读代码确认的：
  - tracker 构造参数覆盖 `device_id`：`tracker.cjs:689-699`、`:1102-1111`；
  - 配置字段进入上报：`_getSendConfig`，`:750-765`；
  - flush 只排空已入队事件：`sink.ts` 的 `flushTelemetry`；
  - 调用顺序：`tracker.ts` 的 `finally`，`create-cli.ts:316/361`。
- 第 1 步 `device-id.ts` 已单独实测：只把 `getConfigDir` 换成读 `BAILIAN_CONFIG_DIR`，用 Node 22 `--experimental-strip-types` 运行。结果如下：
  - 12 个进程同时首次运行，全部返回同一个 ID，与文件内容一致；
  - 目录 700、文件 600，没有残留 `.tmp`；
  - 内容损坏时会重建；
  - 路径被目录占用时返回 `undefined`，不破坏目录；
  - 只读目录的用例因为测试环境以 root 运行，无法验证（root 不受目录权限限制），这正是单测里 `skipIf(getuid() === 0)` 的原因。
- 其余代码片段（`sink.ts` 改动和各个测试文件）**没有**在 `modelstudioai/cli` 里编译或运行。审查环境中安装依赖和执行仓库代码都被权限策略拒绝。合并前请以第 6 步的测试为准；如果 `vp check` 报类型问题，以仓库现有写法为准调整。
