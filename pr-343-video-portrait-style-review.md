# PR #343 审查：视频口型替换（VideoRetalk）与视频风格重绘

> 日期：2026-11-24 · 审查对象：`modelstudioai/cli` PR #343（head `4a61c5ae`，1 个提交 `feat: add video portrait, style, and retouch commands`，基于 main `73c1573`）· 状态：讨论稿

## 结论与范围

结论：`video portrait`（VideoRetalk）和 `video style`（视频风格重绘）对照官方 API 文档逐项核对，请求结构、端点、参数名、取值范围、默认值、结果字段和任务状态处理都正确，没有阻塞性 bug；另有 6 个边界问题（B1–B6），其中 B1（轮询出错丢失 `task_id`）、B2（下载路径要等任务成功后才报错）可能导致重复计费或白等，建议合并前处理。**`video retouch`（万相 2.1 VACE，`wanx2.1-vace-plus`）建议从本 PR 中整体移除**，移除后可以合并。其余问题建议在同一轮或后续 PR 中处理。

| # | 级别 | 问题 | 位置 | 建议 |
| --- | --- | --- | --- | --- |
| R1 | 需要处理（高） | PR 夹带 `video retouch`（万相 2.1 VACE，848 行），不在本次需求的两份文档范围内，且该模型已归入"早期视频模型" | `video/retouch.ts` 等 9 处 | 从本 PR 中整体移除；如需万相视频编辑，另开 PR 接入万相 2.7-视频编辑 |
| P1 | 需要处理（中） | 两个命令都只支持北京地域，但代码不检查 base URL；用国际站配置时只能拿到服务端报错 | `video/portrait.ts:107`、`video/style.ts:111` | 提交前检测非北京 base URL，给出明确的地域提示 |
| P2 | 需要处理（低） | 含非 ASCII 字符的 URL 被直接拒绝；文档的要求是"先进行 URL 编码" | `video/specialized-shared.ts:41` | 自动做 URL 编码后提交，或在报错里提示用户先编码 |
| P3 | 需要处理（低） | 轮询超时下限强制为 600 秒，用户传的更小 `--timeout` 不生效且帮助里没有说明；本地上传沿用全局超时（默认 300 秒），大文件可能先超时 | `video/specialized-shared.ts:18`、`:128`；`core/src/client/client.ts:173` | 帮助文本说明下限；上传使用单独、更宽松的超时 |
| S1 | 建议 | `--fps` 的帮助文本没写服务端默认值 15，其他参数都写了 | `video/style.ts:41` | 补上"默认 15" |
| S2 | 建议 | 输入文件限制（时长、大小、分辨率、帧率）没写进帮助文本，`portrait` 本地上传前也不检查文件大小 | `video/portrait.ts`、`video/style.ts` | 帮助文本写明限制；`portrait` 上传前检查文件大小 |
| S3 | 建议 | `portrait` 看不出是"音频驱动口型"；`video style` 与 `video edit`（帮助里也写了风格转换）职责重叠 | `cli/src/commands.ts`、`skills/bailian-gen/SKILL.md` | 考虑改名为 `retalk` / `lipsync`；在两个命令的帮助里互相说明区别 |
| S4 | 建议 | 结果里 `output_video_url` 既在 `output` 下又在顶层出现一次 | `video/specialized-shared.ts:153-161` | 顶层只保留统一的 `video_url` |
| S5 | 建议 | `video style` 不支持本地文件，与 `portrait` 不一致 | `video/style.ts:148` | **保持现状即可**：风格重绘文档没写可以用临时存储上传，验证后再放开 |
| B1 | 边界（中） | 轮询期间遇到网络抖动或 5xx，命令直接失败，报错里没有 `task_id`；提交后也没输出 `task_id`，用户只能重新提交，重复计费 | `video/specialized-shared.ts:120-136`、`shared/dashscope-task.ts:93-111` | 提交后立即输出 `task_id`；所有轮询错误都带上 `task_id` 和"勿重新提交"提示 |
| B2 | 边界（中） | `--download` 指向目录或不可写路径时，要等任务成功（已计费）后才报错 | `video/specialized-shared.ts:148-150`、`runtime/src/utils/download.ts:18-37` | 在 `validate` 里提前检查 `--download` 路径 |
| B3 | 边界（低） | `bl file upload --model videoretalk` 返回的 `oss://` 地址传给 `portrait`，报 "File not found: oss://..." | `video/specialized-shared.ts:51-68` | `oss://` 直接透传（`vision describe` 已这样处理） |
| B4 | 边界（低） | `--async` 和 `--download`、`--poll-interval` 同时使用时，后两者被静默忽略 | `video/specialized-shared.ts:122-125` | 在 `validate` 里报冲突，或输出 warning |
| B5 | 边界（低） | 第一次查询紧跟在提交之后，`UNKNOWN` 被当作终止状态；`--poll-interval` 大于剩余时间时，最后一次 sleep 会超出截止时间 | `shared/dashscope-task.ts:67`、`runtime/src/utils/polling.ts:48` | 首次查询前先等一个间隔；sleep 时长取间隔和剩余时间的较小值 |
| B6 | 边界（低，非本 PR 引入） | 数字参数传空字符串时 `Number("")` 为 0，`--style ""` 被当作 0 接受；`0x3`、`1e0` 这类写法也能通过 | `runtime/src/args.ts:230-238` | 在通用参数解析里拒绝空字符串和非十进制写法，单独提 PR |

PR 改动概览（17 个文件，+2770 / −4）：

- 新增 `video portrait`：固定模型 `videoretalk`，端点 `image2video/video-synthesis`；视频、音频、参考图都支持本地文件走临时 OSS 上传。
- 新增 `video style`：固定模型 `video-style-transform`，端点 `video-generation/video-synthesis`；只接受公网 URL。
- 新增 `video retouch`：固定模型 `wanx2.1-vace-plus`，5 种 `--function` 模式（本审查建议移除）。
- 新增共享模块 `video/specialized-shared.ts`：负责参数校验、本地文件上传，以及提交、轮询、下载的整个流程。
- `DashScopeTaskResponse` 新增 `PRE-PROCESSING`、`POST-PROCESSING`、`SUSPENDED` 三种状态，以及 `output_video_url`、`orig_prompt`、`actual_prompt` 字段。
- `video task get` 和 `video download` 支持读取 `output_video_url`。
- 新增三个命令的 e2e 测试（本地 mock server），更新 skill 文档。

审查方法：

- 检出 PR head，阅读 `portrait.ts`、`style.ts`、`specialized-shared.ts` 以及 core 的类型、端点、http 层（OSS 请求头）、轮询与下载代码；`retouch.ts` 只确认了模型和文档归属，没有逐行审查。
- 抓取官方文档全文逐项核对：
  - VideoRetalk：<https://help.aliyun.com/zh/model-studio/videoretalk-api>（对应 docs.bailian.console.aliyun.com/zh/model-studio/videoretalk-api）
  - 视频风格重绘：<https://docs.bailian.console.aliyun.com/zh/model-studio/video-style-transform-api-reference>
  - 万相 2.1 视频编辑（VACE）：<https://help.aliyun.com/zh/model-studio/wanx-vace-api-reference>
- **没有运行测试**：审查环境的权限不允许安装依赖、执行测试。live e2e 也没有运行（需要真实 API Key）。

## 与官方文档的逐项核对

### VideoRetalk（`bl video portrait`）

| 项 | 文档 | PR | 结论 |
| --- | --- | --- | --- |
| 提交端点 | `POST /api/v1/services/aigc/image2video/video-synthesis` | `image2videoPath()` | ✅ |
| 异步请求头 | 必须带 `X-DashScope-Async: enable` | `async: true` | ✅ |
| 模型 | `videoretalk` | 固定 | ✅ |
| `input.video_url` / `input.audio_url` | 必填 | `--video` / `--audio` 必填 | ✅ |
| `input.ref_image_url` | 可选，视频中有多张人脸时指定目标人脸 | `--ref-image` | ✅ |
| `parameters.video_extension` | Boolean，默认 false；为 true 时"倒放-正放"交替延长视频 | `--video-extension` | ✅ |
| `parameters.query_face_threshold` | Integer，120–200，默认 170；`ref_image_url` 为空时不生效 | 校验整数 120–200；不带 `--ref-image` 时直接报错 | ✅ 比文档更严格，合理 |
| 本地文件 | 可用临时存储空间上传；`oss://` 地址必须带 `X-DashScope-OssResourceResolve: enable` | 支持上传；http 层检测到 `oss://` 自动加该请求头（`core/src/client/http.ts:85`） | ✅ |
| 任务状态 | PENDING / PRE-PROCESSING / RUNNING / POST-PROCESSING / SUCCEEDED / FAILED / UNKNOWN | 类型已补；非终止状态继续轮询 | ✅ |
| 结果 | `output.video_url`，作业完成后 24 小时有效 | 取 `video_url`，任务成功后立即下载 | ✅ |
| 地域 | 仅华北 2（北京），必须用该地域的 API Key | 只在 notes 里写了"Beijing" | ⚠️ 见 P1 |

文档中的输入限制（CLI 目前没有写进帮助文本，见 S2）：

- 视频：mp4 / avi / mov，≤300MB，2 秒 < 时长 < 120 秒，15–60fps，H.264 / H.265，边长 640–2048。
- 音频：wav / mp3 / aac，≤30MB，2 秒 < 时长 < 120 秒，需要清晰的人声。
- 参考图：jpeg / jpg / png / bmp / webp，≤10MB，长宽比 ≤2，最大边长 ≤4096。

### 视频风格重绘（`bl video style`）

| 项 | 文档 | PR | 结论 |
| --- | --- | --- | --- |
| 提交端点 | `POST /api/v1/services/aigc/video-generation/video-synthesis` | `videoGeneratePath()` | ✅ |
| 异步请求头 | 必须带 `X-DashScope-Async: enable`，缺少会报 "current user api does not support synchronous calls" | `async: true` | ✅ |
| 模型 | `video-style-transform` | 固定 | ✅ |
| `parameters.style` | int，0–7，默认 0：0 日式漫画、1 美式漫画、2 清新漫画、3 3D 卡通、4 国风卡通、5 纸艺风格、6 简易插画、7 国风水墨 | `--style`，校验 0–7 整数，帮助文本列出了风格名 | ✅ |
| `parameters.video_fps` | int，[15, 25]，**默认 15** | `--fps`，校验 15–25 整数 | ✅ 范围正确；默认值没写，见 S1 |
| `parameters.animate_emotion` | bool，默认 true | `--animate-emotion` | ✅ |
| `parameters.min_len` | 只能是 720 或 540，默认 720，影响计费 | `--min-len`，校验 540 / 720 | ✅ |
| `parameters.use_SR` | bool，默认 false；`min_len=540` 时开启会输出 1080P、按 540P 计费 | `--super-resolution`，映射为 `use_SR` | ✅ 大小写正确 |
| 输入 URL | 公网 URL；含中文等非 ASCII 字符时"请先进行 URL 编码"；**没有提到临时存储上传** | 只接受公网 URL；非 ASCII URL 直接拒绝 | ✅ / ⚠️ 见 P2、S5 |
| 任务状态 | PENDING / RUNNING / SUSPENDED / SUCCEEDED / FAILED / UNKNOWN | `SUSPENDED` 已补，会继续轮询 | ✅ |
| 结果 | `output.output_video_url`，任务结果 24 小时有效 | 用 `output_video_url` 取结果；`task get`、`download` 也补上了该字段 | ✅ |
| 地域 | 仅华北 2（北京） | 只在 notes 里写了"Beijing only" | ⚠️ 见 P1 |

文档中的输入限制：MP4 / AVI / MKV / MOV / FLV / TS / MPG / MXF；单边 256–4096 像素，长短边比 ≤1.8；时长 ≤30 秒；大小 ≤100MB。

## R1：移除 `video retouch`（万相 2.1 VACE）

### 原因

1. **超出需求范围**：本次需求对应的是 VideoRetalk 和视频风格重绘两份文档；`retouch` 对应的是另一份文档《万相-视频编辑 API 参考（2.1）》（<https://help.aliyun.com/zh/model-studio/wanx-vace-api-reference>）。PR 的代码和提交说明里都没有附这个链接。
2. **模型已归入早期模型**：阿里云帮助中心的目录里，《万相-视频编辑（2.1）》现在列在 **「万相-早期视频模型（2.1-2.6）」** 下面。万相当前的视频编辑主线是 **「万相 2.7-视频编辑」**。文档里没有看到下线公告，模型应该还能调用，移除是产品取舍，不是因为它不能用。
3. **和已有命令重叠**：`bl video edit`（默认 `happyhorse-1.0-video-edit`）已经覆盖风格转换、物体替换等常用视频编辑能力。
4. **维护成本最高**：`retouch.ts` 有 848 行、近 30 个参数、5 种 `--function` 模式，占了整个 PR 改动量的一大半，却对应最旧的模型。

### 需要移除的改动

| 文件 | 处理 |
| --- | --- |
| `packages/commands/src/commands/video/retouch.ts` | 删除 |
| `packages/commands/tests/e2e/video-retouch.e2e.test.ts` | 删除 |
| `packages/commands/src/index.ts` | 删除 `videoRetouch` 导出 |
| `packages/cli/src/commands.ts` | 删除 `"video retouch": videoRetouch` 注册和对应 import |
| `packages/commands/tests/e2e/topic-routes.ts` | 删除 `"video retouch"` 路由和 `VIDEO_RETOUCH_ROUTES` |
| `packages/core/src/types/api.ts` | 删除 `WanxVaceFunction`、`DashScopeVideoVaceRequest` |
| `packages/core/src/types/index.ts` | 删除上述两个类型的导出 |
| `packages/core/src/types/api.ts`、`video/task-get.ts:44-45` | `orig_prompt`、`actual_prompt` 是给 VACE 加的，没有其他命令使用时一并删除 |
| `skills/bailian-gen/SKILL.md` | 删除 "VACE unified video edit" 一行，以及 "`bl video edit` vs `bl video retouch`" 那段说明；`bl video edit` 一行的 "— not VACE" 改回原文 |
| `skills/bailian-gen/reference/index.md`、`reference/video.md` | 删除 `bl video retouch` 条目和整个命令小节；命令组列表去掉 `retouch` |
| `video/specialized-shared.ts` | 注释 "shared by portrait, style, and retouch" 改为 "portrait and style" |

移除后，`specialized-shared.ts`、`DashScopeTaskResponse` 新增的状态，以及 `output_video_url` 相关改动都要保留，`portrait` 和 `style` 都在用。

### 后续

如果需要万相系列的视频编辑能力，另开 PR 接入「万相 2.7-视频编辑」。更合理的做法是作为 `bl video edit` 的一个可选 `--model`，而不是再新增命令。该文档本次没有审查。

## P1：没有地域检查

两份文档都写明"仅适用于华北 2（北京），且必须使用该地域的 API Key"。PR 只在 notes 里提到北京（`portrait.ts:107`、`style.ts:111`）。用户如果配置了国际站（新加坡）的 base URL 或 API Key，CLI 照样提交，只能拿到服务端返回的、不太直观的报错。

建议：在 `runSpecializedVideoLifecycle` 提交前判断 `ctx.client.url(...)` 的 host，不是北京时给出本地化提示，例如"该模型仅支持华北 2（北京）地域，请使用北京地域的 base URL 和 API Key"。至少在服务端报错时追加这条 hint。dry-run 时也可以作为 warning 输出。

## P2：非 ASCII URL 直接拒绝

`mediaInputError`（`specialized-shared.ts:41`）遇到含非 ASCII 字符的 http(s) URL 直接报 usage 错误。风格重绘文档的原话是"若原始 URL 包含中文字符等非 ASCII 字符，请先进行 URL 编码"，说明服务端只需要编码后的 URL。

建议：在 `resolveMediaUrl` 和预览请求里对 URL 做 `encodeURI`，已编码的部分保持不变，然后提交。如果不想改行为，至少在报错里提示"请先对 URL 进行编码"。两个命令都适用。

## P3：超时设置

1. `SPECIALIZED_VIDEO_TIMEOUT_FLOOR_SEC = 600`（`specialized-shared.ts:18`），轮询超时是 `Math.max(settings.timeout, 600)`（`:128`）。用户传 `--timeout 60` 实际仍然会等 600 秒，帮助文本里没有说明。建议在命令 notes 里写明"等待结果时轮询至少 600 秒"，或者只在用户没显式传 `--timeout` 时才用这个下限。
2. 本地上传沿用全局 `settings.timeout`（`core/src/client/client.ts:173`，默认 300 秒）。VideoRetalk 的视频最大 300MB，如果全局超时设得短或网络慢，上传会先超时。建议上传使用单独、更宽松的超时，或者按文件大小估算。

## S1–S5：建议

- **S1**：`style.ts:41` 的 `--fps` 描述改为 "Output video FPS 15–25 (default: 15; maps to video_fps)"，和其他参数的写法保持一致。
- **S2**：把上面"与官方文档的逐项核对"里列出的输入限制写进两个命令的 notes 或 skill 文档。`portrait` 支持本地上传，上传前可以先检查文件大小（视频 ≤300MB、音频 ≤30MB、参考图 ≤10MB），避免传完才被服务端拒绝。
- **S3**：`portrait` 这个名字看不出是"音频驱动口型替换"，可以考虑 `retalk` 或 `lipsync`。`video style` 和 `video edit` 都涉及"风格转换"，建议在两个命令的帮助里各加一句说明区别：`style` 是 8 种预设风格的整体重绘，`edit` 是基于 prompt 的通用编辑。
- **S4**：`specialized-shared.ts:153-161` 的结果先展开了整个 `taskResponse`（其中已有 `output.output_video_url`），又在顶层再写一次 `output_video_url`。顶层已经有统一的 `video_url`，可以去掉这次重复。
- **S5**：`style.ts:148` 用 `allowLocal: false` 拒绝本地文件，和 `portrait` 不一致，但有依据：VideoRetalk 文档明确写了可以用临时存储空间上传，风格重绘文档没写。保持现状；等实测验证 `video-style-transform` 能解析 `oss://` 地址后再放开，放开时只需把 `allowLocal` 改为 `true` 并加上 `resolveMediaUrl`。

## B1–B6：边界情况

### B1：轮询中途出错会丢失 `task_id`（中）

`runSpecializedVideoLifecycle` 提交成功后没有输出 `task_id`（`specialized-shared.ts:120-136`），直接进入轮询。`pollDashScopeTask` 只把超时类错误包装成带 `task_id` 的报错（`dashscope-task.ts:93-111`）。以下情况都会让命令直接失败，而且报错里没有 `task_id`：

- 某次查询遇到网络抖动（ECONNRESET 等）或网关 5xx；
- 用户按 Ctrl+C 中断；
- 任务 `FAILED`（报错里只有 `requestId`）。

这两个模型的任务通常要跑几分钟，轮询下限又是 600 秒，期间出现一次抖动的概率不低。服务端任务仍在执行，但用户拿不到 `task_id`，只能重新提交，**重复计费**。下载失败的路径（`downloadDashScopeTaskResult`）已经正确带上了 `task_id` 和"勿重新提交"提示，轮询这条路径也应该一样。

建议：

1. 提交成功后立即 `ctx.output.progress` 输出 `task_id`。
2. `pollDashScopeTask` 的 catch 对所有错误都附加 `task_id`，并给出 `bl video task get <task_id>` / `bl video download <task_id>` 的提示。
3. 可选：单次查询失败时重试几次再放弃。

### B2：`--download` 路径问题要等任务成功后才暴露（中）

`downloadFile` 的 `resolveDownloadTarget`（`runtime/src/utils/download.ts:18-37`）在目标已存在、但不是普通文件时（比如目录）报 "Download destination must be a regular file"。这个检查发生在任务成功之后，用户已经等了几分钟、模型也已计费。目标目录没有写权限时同样如此。

```bash
bl video portrait --video a.mp4 --audio a.wav --download ./out/   # ./out 是已有目录
```

建议在 `validate` 里提前检查：`--download` 已存在且不是普通文件时直接报 usage 错误；父目录存在但不可写时也提前报错。错误发生在下载阶段时，现有提示已经能让用户用 `task_id` 重新下载，这一点是好的。

### B3：`oss://` 地址被当作本地路径（低）

`mediaInputError` 把所有非 http(s) 的值都当本地路径处理（`specialized-shared.ts:51-68`）。用户先用 `bl file upload --file a.mp4 --model videoretalk` 拿到 `oss://` 地址，再传给 `portrait`，会得到 "File not found: oss://..."。而 core 的 `isLocalFile` 已经把 `oss://` 视为远程地址，`vision describe` 也直接透传 `oss://`。

建议：`portrait` 接受 `oss://` 并原样透传，http 层会自动加 `X-DashScope-OssResourceResolve`。`style` 维持 `allowLocal: false` 时，`oss://` 也应给出"仅支持公网 URL"的明确报错，而不是 "File not found"。

### B4：`--async` 与 `--download`、`--poll-interval` 冲突时静默忽略（低）

`--async` 在提交后直接返回（`specialized-shared.ts:122-125`），同时传入的 `--download`、`--poll-interval` 不生效，也没有任何提示。脚本或 Agent 可能以为文件已经保存。建议在 `validate` 里把这种组合当作 usage 错误，或至少输出 warning。

### B5：轮询节奏的两个小问题（低）

- 提交后立刻发起第一次查询。如果服务端短时间内还查不到任务、返回 `UNKNOWN`，会被当作终止状态直接报错（`dashscope-task.ts:67`）。VideoRetalk 文档对 `UNKNOWN` 的解释是"作业不存在或状态未知"。建议首次查询前先等一个轮询间隔，或者前几次遇到 `UNKNOWN` 时继续等待。
- `poll` 在循环末尾固定 sleep `intervalSec`（`runtime/src/utils/polling.ts:48`）。用户传很大的 `--poll-interval` 时，最后一次 sleep 会超出截止时间。建议 sleep 取 `min(intervalSec, 剩余时间)`。

### B6：数字参数的解析宽松（低，非本 PR 引入）

通用参数解析用 `Number(value)`（`runtime/src/args.ts:230-238`）：`Number("")` 为 0，所以 `--style ""` 会被当作 `--style 0` 接受，`--style 0x3`、`--style 1e0` 也能通过整数校验。这是 runtime 的通用问题，不是本 PR 引入的，但 `--style` 这类枚举型数字参数更容易踩到。建议单独提 PR，在通用解析里拒绝空字符串和非十进制写法。

## 做得好的地方

- 抽出共享的提交、轮询、下载流程，三个命令（移除 retouch 后两个）的行为一致；下载失败时保留 `task_id`，并提示用户"勿重新提交"。
- `task get` 和 `download` 同步支持 `output_video_url`，`--async` 之后用 `video download` 也能拿到结果。
- 新增的三种任务状态有对应的 e2e 用例（"wait mode continues through PRE-PROCESSING / SUSPENDED"）。
- dry-run 用 `<local-file:...;pending-upload>` 占位，不会为了预览而真的上传文件。
- 校验逻辑放在 `validate` 里，和认证无关，参数错误时不需要 API Key 就能快速失败。
