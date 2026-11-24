# 文本/多模态向量与重排序：CLI 需求实现计划

> 日期：2026-11-24 · 代码基线：`modelstudioai/cli` main `c6c9754`（PR #344 合入后）· 状态：实现前计划

## 结论

**两个需求都好实现，预计 2–3 人日（含测试与文档）。** 三个接口都是同步 HTTP，没有异步任务和轮询，请求、响应结构简单。CLI 里已经有可以直接复用的部件：模型域请求（`ctx.client.requestJson`）、本地图片转 Data URI、临时 OSS 上传、dry-run 和输出格式。

复杂度和风险集中在三处：

1. **多模态向量的模型能力差异很大**：7 个模型在融合向量的开法、支持的维度、多图、单请求条数、文件格式上各不相同，需要在 CLI 里维护一张能力表，在本地校验。见 §5.2。
2. **文本向量要按模型分批**：单请求条数上限分别是 20、10、25，CLI 需要自动分批并合并结果。重排序**不能**这样分批，因为分数是请求内的相对值。见 P-T3、P-R1。
3. **请求地址**：官方文档的示例已经统一改成 workspace 专属域名（`https://{WorkspaceId}.cn-beijing.maas.aliyuncs.com`），CLI 默认仍是 `https://dashscope.aliyuncs.com`。动手前要先用 curl 确认三个接口在默认域名上都能用。见 §8 第 1 步。

建议新增 3 条命令：

| 命令 | 接口 | 默认模型 |
| --- | --- | --- |
| `bl embedding text` | DashScope 文本向量（同步） | `qwen3.7-text-embedding` |
| `bl embedding multimodal` | DashScope 多模态向量 | `tongyi-embedding-vision-plus` |
| `bl rerank` | DashScope 文本 / 多模态重排序 | `qwen3.7-text-rerank` |

本文所有接口细节都来自 2026-11-24 抓取的官方文档（链接见 §11）。没有实测的结论都标了 **【待验证】**。

---

## 1. 范围

**本期做：**

- 文本向量：dense 向量、sparse 向量，以及 `text_type`、`instruct`、`dimension` 参数，超过单请求条数时自动分批。
- 多模态向量：文本、图片、视频的独立向量和融合向量；图片支持 URL、本地文件、Data URI 三种输入。
- 重排序：文本重排序（`qwen3.7-text-rerank`、`qwen3-rerank`、`gte-rerank-v2`）和多模态重排序（`qwen3-vl-rerank`）。
- 统一的 JSON 输出，可选写入 JSONL 文件；text 模式输出可读摘要。

**本期不做（非目标）：**

- 大批量离线向量化：已经有 `bl batch`，它支持 `/v1/embeddings` 端点（`core/src/batch/types.ts:3`），在帮助文本里引导用户去用即可。
- 异步文本向量接口 `text-embedding-async-v1/v2`：旧接口，没有新需求。
- OpenAI 兼容接口的 embeddings 和 reranks：功能是 DashScope 接口的子集，理由见 §3。
- 相似度计算、向量库写入等下游能力。

---

## 2. 接口速查（以官方文档为准）

### 2.1 文本向量（DashScope 同步接口）

- **端点**：`POST {base}/api/v1/services/embeddings/text-embedding/text-embedding`
- **请求体**：

```json
{
  "model": "qwen3.7-text-embedding",
  "input": { "texts": ["衣服的质量杠杠的", "..."] },
  "parameters": {
    "dimension": 1024,
    "text_type": "query",
    "output_type": "dense&sparse",
    "instruct": "Given a research paper query, retrieve relevant research paper"
  }
}
```

- **响应**（HTTP 接口里 `embeddings` 包在 `output` 中）：

```json
{
  "output": {
    "embeddings": [
      {
        "index": 0,
        "embedding": [-0.0069, "..."],
        "sparse_embedding": [{ "index": 7149, "value": 0.829, "token": "风" }]
      }
    ]
  },
  "usage": { "total_tokens": 27 },
  "request_id": "..."
}
```

- **参数**：

| 参数 | 位置 | 说明 |
| --- | --- | --- |
| `input.texts` | input | 字符串数组。HTTP 接口不能直接传文件，"文件输入"是 SDK 的能力，CLI 要自己按行读文件。 |
| `dimension` | parameters | 不同模型可选值不同，见下表；默认 1024。 |
| `text_type` | parameters | `query` 或 `document`（默认）；检索这类非对称任务才需要区分。 |
| `output_type` | parameters | `dense`（默认）、`sparse`、`dense&sparse`；仅部分模型支持。 |
| `instruct` | parameters | 英文任务说明；**只在 `text_type=query` 时生效**，并按输入条数重复计入 token 用量。 |

- **模型**（北京地域）：

| 模型 | 维度（默认值加粗） | 单请求最大条数 | 单条最大 token | 稀疏向量 | instruct |
| --- | --- | --- | --- | --- | --- |
| `qwen3.7-text-embedding` | 2560 / 2048 / 1536 / **1024** / 768 / 512 / 256 | 20 | 128,000 | ✅ | ✅ |
| `qwen3.7-text-embedding-flash` | **1024** / 768 / 512 / 256 | 20 | 128,000 | ✅（见 P-T6） | ✅ |
| `text-embedding-v4` | 2048 / 1536 / **1024** / 768 / 512 / 256 / 128 / 64 | 10 | 8,192 | ✅ | ✅ |
| `text-embedding-v3` | **1024** / 768 / 512 / 256 / 128 / 64 | 10 | 8,192 | ✅ | ❌ |
| `text-embedding-v2` / `v1` | 1536（固定） | 25 | 2,048 | ❌ | ❌ |

新加坡地域只有 `qwen3.7-text-embedding`、`text-embedding-v4`、`text-embedding-v3`。其中 `text-embedding-v4` 的单条上限是 8,192，`text-embedding-v3` 只有 1024 / 768 / 512 三档维度。

### 2.2 多模态向量

- **端点**：`POST {base}/api/v1/services/embeddings/multimodal-embedding/multimodal-embedding`。这一页文档的 HTTP 示例用的是 `https://dashscope.aliyuncs.com`，SDK 示例用的是 workspace 域名。
- **请求体**：

```json
{
  "model": "qwen3-vl-embedding",
  "input": {
    "contents": [
      { "text": "商品描述" },
      { "image": "https://... 或 data:image/png;base64,..." },
      { "video": "https://..." },
      { "multi_images": ["https://...", "https://..."] }
    ]
  },
  "parameters": {
    "dimension": 1024,
    "enable_fusion": true,
    "fps": 0.5,
    "instruct": "...",
    "res_level": 1,
    "max_video_frames": 8
  }
}
```

- **响应**：

```json
{
  "output": {
    "embeddings": [{ "index": 0, "embedding": ["..."], "type": "text" }]
  },
  "usage": { "...": "各模型字段不同，见 P-M8" },
  "request_id": "..."
}
```

- **`type` 的取值**：`text`、`image`、`video`、`multi_images`。融合向量另有特殊值：2026-03-06 快照版返回 `fused`；`qwen3-vl-embedding` 和 `qwen2.5-vl-embedding` 在融合模式下返回 `fusion`；`qwen3-vl-embedding` 在独立模式下返回 `vl`。
- **参数**：

| 参数 | 说明 |
| --- | --- |
| `dimension` | 可选值因模型而异，见 §5.2；`multimodal-embedding-v1` 不支持，固定 1024 维。 |
| `enable_fusion` | 仅 `qwen3-vl-embedding` 支持。 |
| `fps` | 视频抽帧比例，范围 [0, 1]，默认 1.0；**超出范围不报错**。 |
| `instruct` | 自定义任务说明，建议用英文。 |
| `res_level` | 分辨率档位 0–3，对应单图 127 / 402 / 578 / 1026 token，默认 1；仅 2026-03-06 快照版支持。 |
| `max_video_frames` | 最大采样帧数，≤64，默认 8；仅 2026-03-06 快照版支持。 |
| `output_type` | 只支持 `dense`，传其他值会被忽略。CLI 不暴露这个参数。 |

### 2.3 重排序（DashScope 接口）

- **端点**：`POST {base}/api/v1/services/rerank/text-rerank/text-rerank`。多模态的 `qwen3-vl-rerank` **也走这个 text-rerank 路径**。
- **请求体**：

```json
{
  "model": "qwen3.7-text-rerank",
  "input": {
    "query": "什么是文本排序模型",
    "documents": ["文档 A", "文档 B"]
  },
  "parameters": {
    "top_n": 2,
    "return_documents": false,
    "instruct": "Given a web search query, retrieve relevant passages that answer the query."
  }
}
```

  `qwen3-vl-rerank` 的 `query` 可以是字符串、`{"text": ...}` 或 `{"image": ...}`；`documents` 的元素可以是字符串、`{"text": ...}`、`{"image": ...}` 或 `{"video": ...}`。`parameters` 里还多一个 `fps`，范围 (0, 5]，默认 1.0。

- **响应**：

```json
{
  "output": {
    "results": [{ "index": 0, "relevance_score": 0.93, "document": { "text": "..." } }]
  },
  "usage": { "prompt_tokens": 79, "total_tokens": 79 },
  "request_id": "..."
}
```

  `results` 已按分数从高到低排列。只有请求里设置了 `return_documents: true` 才会返回 `document` 字段；`prompt_tokens` 只有 `qwen3.7-text-rerank` 返回。

- **模型**：

| 模型 | 最大文档数 | 单条最大 token | 单请求最大 token | instruct | return_documents |
| --- | --- | --- | --- | --- | --- |
| `qwen3.7-text-rerank` | 500 | 30,000 | 120,000（文档写"建议"） | ✅ | ❌ **不在支持列表里** |
| `qwen3-rerank` | 500 | 4,000 | 120,000 | ✅ | ✅ |
| `gte-rerank-v2` | 500 | 4,000 | 30,000 | ❌ | ✅ |
| `qwen3-vl-rerank` | 文本 100 / 图片 40 / 视频 4 | 8,000 | 120,000 | ✅ | ✅ |

  - 单请求 token 的计算方式：`query token 数 × 文档数 + 所有文档 token 之和`。
  - 单条 query 或文档超过上限时，接口直接返回 **HTTP 400，不会截断**。
  - 文档写明 `gte-rerank` 已于 2026-05-30 下线，推荐改用 `qwen3-rerank`；`gte-rerank-v2` 目前仍在模型列表里。

---

## 3. 关键设计决策

| # | 决策 | 选择 | 理由 |
| --- | --- | --- | --- |
| D1 | 协议 | **三个接口都走 DashScope 原生接口** | OpenAI 兼容 embeddings 不支持 `text_type`、`instruct` 和稀疏向量；多模态向量没有兼容接口；兼容 reranks 不支持 `gte-rerank-v2` 和 `qwen3-vl-rerank`（会返回 `model_not_supported`），而且路径前缀是 `/compatible-api/v1`，和 chat 用的 `/compatible-mode/v1` 不一样（P-R6）。 |
| D2 | 请求地址 | 和图片、视频、语音命令一样，用 `ctx.client.url(path)`，跟随 `--base-url`、`DASHSCOPE_BASE_URL` 和 Profile | 以后 model-endpoint 改造（见本仓库 [model-endpoint-proposal.md](model-endpoint-proposal.md)）合入后会自动切到 workspace 域名，新命令不用自己拼地址。前提是默认域名可用，见 §8 第 1 步；如果不可用，退回决策模型的做法，见 P-C1。 |
| D3 | 命令结构 | 新建顶层命令组 `embedding`（下设 `text`、`multimodal`），外加一个叶子命令 `rerank` | 文本向量和多模态向量的输入形态差别很大，合成一个命令会让参数校验非常复杂。`rerank` 的文本和多模态共用同一个接口，用一个命令就够了。顶层叶子命令已有先例 `decision`（`cli/src/commands.ts:424`）。备选方案见 §10 Q1。 |
| D4 | 文档原文怎么回填 | **永远不发 `return_documents`**，由 CLI 按 `index` 从本地输入回填 | `qwen3.7-text-rerank` 不支持这个参数，各模型返回的 `document` 结构也不统一；本地回填行为一致，还能省流量。 |
| D5 | 文本向量超过单请求条数 | 按模型的条数上限**自动分批、串行请求**，合并时修正 `index`、累加 `usage` | 用户不需要记每个模型的上限；串行请求可以避免触发限流（P-T3）。 |
| D6 | 重排序超过文档数上限 | **在本地直接报错，不分批** | 分数是"当前请求内的相对分数，不可跨请求比较"，分批后合并的排序没有意义（P-R1）。 |
| D7 | 向量输出 | JSON 模式输出完整向量；`--save <path>` 写 JSONL；text 模式只输出摘要 | 2560 维 × 20 条的向量直接打到终端没法看，也会拖慢 Agent（P-C4）。 |
| D8 | 默认模型 | 文本向量 `qwen3.7-text-embedding`；多模态向量 `tongyi-embedding-vision-plus`；重排序 `qwen3.7-text-rerank` | 文本向量和重排序选文档推荐的最新模型。多模态向量选 `tongyi-embedding-vision-plus`，因为它在北京和新加坡都可用，而且有免费额度；`qwen3-vl-embedding` 只在北京可用。见 §10 Q2。 |

---

## 4. 命令设计

### 4.1 `bl embedding text`

```bash
bl embedding text --text "衣服的质量杠杠的"
bl embedding text --text "风急天高猿啸哀" --text "渚清沙白鸟飞回" --dimension 512
bl embedding text --file corpus.txt --save vectors.jsonl
cat queries.txt | bl embedding text --file - --text-type query --instruct "Given a web search query, retrieve relevant passages"
bl embedding text --text "机器学习" --output-type both --model text-embedding-v4
```

| 参数 | 类型 | 映射 | 校验 |
| --- | --- | --- | --- |
| `--text <s>` | array | `input.texts[]` | 和 `--file` 至少提供一个，两者可以同时用（顺序是先 `--text` 后 `--file`）；不允许空字符串。 |
| `--file <path\|->` | string | 按行读取，追加到 `input.texts` | `-` 表示从 stdin 读取；处理规则见 P-T4。 |
| `--model <m>` | string | `model` | 默认 `qwen3.7-text-embedding`。 |
| `--dimension <n>` | number | `parameters.dimension` | 已知模型按 §2.1 的维度表校验；未知模型不校验，直接透传。 |
| `--text-type <query\|document>` | string | `parameters.text_type` | 枚举。 |
| `--instruct <s>` | string | `parameters.instruct` | **必须同时传 `--text-type query`**，否则报 usage 错误（P-T2）。 |
| `--output-type <dense\|sparse\|both>` | string | `parameters.output_type`，其中 `both` 映射为 `dense&sparse` | 命令行取值用 `both`，避免 shell 里的 `&`（P-T5）；已知不支持稀疏向量的模型直接报错。 |
| `--batch-size <n>` | number | CLI 内部分批大小 | 默认等于模型的条数上限，未知模型默认 10；不能超过已知模型的上限。 |
| `--save <path>` | string | 写 JSONL，每行一个 `{index, text, embedding, sparse_embedding?}` | 写之前先检查路径（目录、是否可写），避免请求做完才失败。 |

JSON 输出（`--output json`，默认）：

```json
{
  "model": "qwen3.7-text-embedding",
  "dimension": 1024,
  "count": 2,
  "embeddings": [{ "index": 0, "embedding": ["..."] }],
  "usage": { "total_tokens": 27 },
  "request_ids": ["..."],
  "saved": "vectors.jsonl"
}
```

- 用 `request_ids`（数组）而不是 `request_id`，因为分批时会有多个请求。
- 传了 `--save` 时，JSON 输出里**不再带 `embeddings` 数组**，只保留 `count` 和 `saved`，避免输出两份数据。

text 输出：每条一行，例如 `#0  dim=1024  norm=1.000  [-0.0069, 0.0306, … ]  衣服的质量杠杠的`（文本截断到 40 字），最后一行输出 token 用量。

### 4.2 `bl embedding multimodal`

```bash
bl embedding multimodal --image ./shoe.jpg
bl embedding multimodal --text "白色运动鞋" --image https://example.com/shoe.png --dimension 512
bl embedding multimodal --video https://example.com/clip.mp4 --fps 0.5
bl embedding multimodal --model qwen3-vl-embedding --fusion --text "找相似风格" --image ./shirt.png
bl embedding multimodal --contents-file contents.json      # 精确控制顺序、多图、多个融合对象
```

| 参数 | 类型 | 映射 | 说明 |
| --- | --- | --- | --- |
| `--text <s>` | array | `{"text": s}` | |
| `--image <url\|path\|data-uri>` | array | `{"image": ...}` | 本地文件转 Data URI（P-M4）。 |
| `--video <url\|path>` | array | `{"video": ...}` | 只支持 URL；本地文件见 P-M5。 |
| `--multi-images <a,b,...>` | array（每个值是逗号分隔的一组） | `{"multi_images": [...]}` | 只有 tongyi-embedding-vision 系列支持。 |
| `--contents-file <path\|->` | string | 直接作为 `input.contents` | 和上面四个参数互斥；内容是 JSON 数组。 |
| `--fusion` | switch | 按模型改写请求，见 §5.2 | 对不支持融合的模型直接报错。 |
| `--model` | string | `model` | 默认 `tongyi-embedding-vision-plus`。 |
| `--dimension` | number | `parameters.dimension` | 按模型校验；`multimodal-embedding-v1` 不允许传。 |
| `--fps <0..1>` | number | `parameters.fps` | **在本地校验范围**，因为服务端超出范围不报错（P-M7）。 |
| `--instruct` | string | `parameters.instruct` | |
| `--res-level <0-3>` | number | `parameters.res_level` | 只允许 2026-03-06 快照版使用。 |
| `--max-video-frames <n>` | number | `parameters.max_video_frames` | ≤64，只允许 2026-03-06 快照版使用。 |
| `--save <path>` | string | 写 JSONL：`{index, type, source, embedding}` | 和文本向量相同。 |

- **内容顺序**：用参数传入时，`input.contents` 按 `--text` → `--image` → `--video` → `--multi-images` 的固定顺序组装，每组内部保持命令行里的顺序（P-M2）。
- 输出里每条向量都带上 `source`（文本截断，或 URL / 本地路径）和 `type`，方便对应回输入。

### 4.3 `bl rerank`

```bash
bl rerank --query "什么是文本排序模型" --document "文本排序模型用于搜索…" --document "量子计算是…" --top-n 1
bl rerank --query "如何预防感冒" --documents-file candidates.txt --top-n 5
bl rerank --query "如何修改密码" --documents-file faq.jsonl --instruct "Retrieve semantically similar text."
bl rerank --model qwen3-vl-rerank --query-image ./query.jpg --document "红色连衣裙" --document-image https://example.com/a.jpg
```

| 参数 | 类型 | 映射 | 说明 |
| --- | --- | --- | --- |
| `--query <s>` | string | `input.query` | 和 `--query-image` 二选一。 |
| `--query-image <url\|path>` | string | `input.query = {"image": ...}` | 只允许 `qwen3-vl-rerank`。 |
| `--document <s>` | array | `documents[]`（字符串） | |
| `--documents-file <path\|->` | string | 每行一条文档，见 P-R4 | 以 `.json` / `.jsonl` 结尾时按 JSON 解析，可以写多模态对象。 |
| `--document-image <url\|path>` / `--document-video <url>` | array | `{"image": ...}` / `{"video": ...}` | 只允许 `qwen3-vl-rerank`。 |
| `--top-n <n>` | number | `parameters.top_n` | 正整数；大于文档数时服务端返回全部，CLI 不报错。 |
| `--instruct <s>` | string | `parameters.instruct` | `gte-rerank-v2` 不支持，报错（P-R5）。 |
| `--fps <n>` | number | `parameters.fps` | 只允许 `qwen3-vl-rerank`，范围 (0, 5]。 |
| `--model` | string | `model` | 默认 `qwen3.7-text-rerank`。 |

JSON 输出：

```json
{
  "model": "qwen3.7-text-rerank",
  "results": [
    { "rank": 1, "index": 0, "relevance_score": 0.9334, "document": "文本排序模型用于搜索…" }
  ],
  "usage": { "total_tokens": 79 },
  "request_id": "..."
}
```

- `document` 由 CLI 按 D4 在本地回填：文本文档是字符串，多模态文档是原来的对象。
- text 输出是一张表：`rank`、`score`（保留 4 位小数）、`index`、文档预览（截断）。

---

## 5. 实现细节

### 5.1 文件改动清单

| 文件 | 改动 |
| --- | --- |
| `packages/core/src/client/endpoints.ts` | 新增 `textEmbeddingPath()`、`multimodalEmbeddingPath()`、`rerankPath()`，都只返回 path（参照 `videoGeneratePath`）。 |
| `packages/core/src/types/api.ts` | 新增 `TextEmbeddingRequest/Response`、`MultimodalEmbeddingRequest/Response`、`RerankRequest/Response`，以及 `MultimodalContent`、`SparseEmbedding` 等类型。`usage` 定义成宽松的 `Record<string, number \| undefined>`（P-M8）。 |
| `packages/core/src/types/index.ts`、`packages/core/src/index.ts` | 导出新类型和新 path。 |
| `packages/core/src/embedding/models.ts`（新建） | **三张模型能力表**：文本向量、多模态向量、重排序（§5.2）。放在 core 而不是 commands，方便以后 batch 或 knowledge 复用。 |
| `packages/commands/src/commands/embedding/shared.ts`（新建） | 读文件或 stdin 并按行拆分、`--save` 写 JSONL、向量摘要格式化（维度、范数、前几个值）、媒体输入解析（URL / 本地路径 / Data URI）。 |
| `packages/commands/src/commands/embedding/text.ts`（新建） | 文本向量命令，含自动分批。 |
| `packages/commands/src/commands/embedding/multimodal.ts`（新建） | 多模态向量命令。 |
| `packages/commands/src/commands/rerank/index.ts`（新建） | 重排序命令。 |
| `packages/commands/src/index.ts` | 导出 `embeddingText`、`embeddingMultimodal`、`rerank`。 |
| `packages/cli/src/commands.ts` | import 并注册 `"embedding text"`、`"embedding multimodal"`、`rerank`。 |
| `packages/core/src/telemetry/tracker.ts:36` | `PARAM_ALLOWLIST` 加入 `dimension`、`textType`、`outputType`、`topN`、`fusion`。**绝对不要加 `text`、`query`、`document`、`instruct`**：这些是用户原文（P-C6）。 |
| `tools/generate-reference.ts:46` | `GROUP_OWNER_SKILL` 决定 `embedding`、`rerank` 归哪个 skill（§10 Q3）；不加就默认归 `bailian-cli`。 |
| `skills/*/SKILL.md` | 加一条路由说明："向量化或重排序用 `bl embedding` / `bl rerank`；大批量离线任务用 `bl batch`"。 |
| `skills/*/reference/*.md` | **不要手改**，运行 `pnpm sync:skill-assets` 生成。 |
| `CHANGELOG.md`、`CHANGELOG.zh.md` | 在"新增"下写 3 条命令。 |
| `packages/commands/tests/e2e/topic-routes.ts` | 新增 `EMBEDDING_ROUTES`、`RERANK_ROUTES`。 |
| `packages/commands/tests/e2e/embedding-*.e2e.test.ts`、`rerank.e2e.test.ts`（新建） | 用本地 mock server 测试，写法参照 `decision.e2e.test.ts`、`image-translate.e2e.test.ts`。 |
| `packages/core/tests/embedding-models.test.ts`（新建） | 能力表与校验函数的单测。 |

### 5.2 模型能力表（核心）

**文本向量**：

```ts
export const TEXT_EMBEDDING_MODELS = {
  "qwen3.7-text-embedding":       { maxBatch: 20, dims: [2560, 2048, 1536, 1024, 768, 512, 256], sparse: true,  instruct: true  },
  "qwen3.7-text-embedding-flash": { maxBatch: 20, dims: [1024, 768, 512, 256],                   sparse: true,  instruct: true  }, // sparse 见 P-T6
  "text-embedding-v4":            { maxBatch: 10, dims: [2048, 1536, 1024, 768, 512, 256, 128, 64], sparse: true, instruct: true },
  "text-embedding-v3":            { maxBatch: 10, dims: [1024, 768, 512, 256, 128, 64],          sparse: true,  instruct: false },
  "text-embedding-v2":            { maxBatch: 25, dims: [1536], fixedDim: true,                  sparse: false, instruct: false },
  "text-embedding-v1":            { maxBatch: 25, dims: [1536], fixedDim: true,                  sparse: false, instruct: false },
} as const;
```

**多模态向量**（最容易出错的一张表）：

| 模型 | 维度（默认） | 融合方式 | 独立向量 | multi_images | 单请求上限 | 图片格式 | 图片大小 | 视频 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `qwen3-vl-embedding` | 2560 / 2048 / 1536 / 1024 / 768 / 512 / 256（2560） | `parameters.enable_fusion=true` | ✅（type=`vl`） | ❌（多图要写成多个 `image` 条目） | 共 20 个元素，图片 ≤10，视频 ≤1 | JPEG/PNG/WEBP/BMP/TIFF/ICO/DIB/ICNS/SGI | ≤10MB | MP4/AVI/MOV，≤50MB |
| `qwen2.5-vl-embedding` | 2048 / 1024 / 768 / 512（1024） | **永远是融合** | ❌ | ❌ | 每种类型最多 1 个 | 同上 | ≤5MB | ≤50MB |
| `tongyi-embedding-vision-plus-2026-03-06` | 1152 / 1024 / 512 / 256 / 128 / 64（1152） | **把多种模态放进同一个 content 对象**；不用 `enable_fusion` | ✅ | ✅ | 共 20 个元素，图片 ≤64，视频 ≤8 | 同上 | 建议 ≤5MB，最大 10MB | 8 种格式，≤50MB，H.264/H.265 |
| `tongyi-embedding-vision-flash-2026-03-06` | 768 / 512 / 256 / 128 / 64（768） | 同上 | ✅ | ✅ | 同上 | 同上 | 同上 | 同上 |
| `tongyi-embedding-vision-plus` | 1152 / 1024 / 512 / 256 / 128 / 64（1152） | ❌ 不支持 | ✅ | ✅（≤8 张） | 不限元素数，只受 token 上限约束 | **JPG/PNG/BMP** | ≤3MB | ≤10MB |
| `tongyi-embedding-vision-flash` | 768 / 512 / 256 / 128 / 64（768） | ❌ | ✅ | ✅（≤8 张） | 同上 | 同上 | ≤3MB | ≤10MB |
| `multimodal-embedding-v1` | 固定 1024，不能传 `dimension` | ❌ | ✅ | ❌ | 共 20 个元素，图片和视频各 ≤1，文本 ≤20 | JPG/PNG/BMP | ≤3MB | ≤10MB |

**`--fusion` 怎么改写请求**：

- **`qwen3-vl-embedding`**：`contents` 保持不变，加上 `parameters.enable_fusion = true`。
- **2026-03-06 快照版**：把所有 `text`、`image`、`video` 合并进**一个** content 对象，例如 `[{"text": "…", "image": "…"}]`。注意同一个对象里每种 key 只能出现一次：如果有多个文本或多张图片，要么报错，要么提示用户改用 `--contents-file` 自己组织。建议报错。
- **`qwen2.5-vl-embedding`**：不传 `--fusion` 也是融合向量。帮助文本里要写明；用户传了 `--fusion` 也不报错。
- **其他模型**：传 `--fusion` 直接报 usage 错误。

**未知模型**（比如以后新出的模型）：跳过所有能力校验，参数原样透传，并输出一条 warning："未知模型，未做本地校验"。这样新模型不需要发版就能用。

**重排序**：

```ts
export const RERANK_MODELS = {
  "qwen3.7-text-rerank": { maxDocs: 500, multimodal: false, instruct: true,  maxRequestTokens: 120_000 },
  "qwen3-rerank":        { maxDocs: 500, multimodal: false, instruct: true,  maxRequestTokens: 120_000 },
  "gte-rerank-v2":       { maxDocs: 500, multimodal: false, instruct: false, maxRequestTokens: 30_000 },
  "qwen3-vl-rerank":     { maxDocs: { text: 100, image: 40, video: 4 }, multimodal: true, instruct: true, maxRequestTokens: 120_000 },
} as const;
```

### 5.3 文本向量的分批流程

```text
texts = [...--text, ...readLines(--file)]
batchSize = --batch-size ?? model.maxBatch ?? 10
for chunk, offset in chunks(texts, batchSize):        # 串行
    resp = POST textEmbeddingPath { model, input: { texts: chunk }, parameters }
    for e in resp.output.embeddings: e.index += offset
    merged.embeddings.push(...)
    merged.usage.total_tokens += resp.usage.total_tokens
    merged.request_ids.push(resp.request_id)
    progress("embedded {offset + chunk.length}/{texts.length}")
merged.embeddings.sortBy(index)                       # 服务端不保证按 index 顺序返回，见 P-T7
```

- **失败处理**：任意一批失败就停止。报错里要写明"已完成 N/M 条"；如果传了 `--save`，**已完成的部分保留在 `<path>.partial`**，不覆盖用户原来的文件（P-T8）。
- **dry-run**：只输出第一批的请求体和总批数，不发请求。

### 5.4 媒体输入解析（多模态向量和多模态重排序共用）

```text
resolveMedia(value, kind, model):
  http(s)://       → 原样使用（URL 里有非 ASCII 字符时先 encodeURI，见 P-M6）
  data:            → 原样使用，只在 dry-run 输出里脱敏
  oss://           → 原样使用（http 层会自动加 X-DashScope-OssResourceResolve 头）
  本地文件 + image → 先按模型检查扩展名和大小，再用 imageFileToDataUri（core/src/files/upload.ts:259）
  本地文件 + video → 默认报错："视频只支持 URL"；可选的上传方案见 P-M5
  其他             → 报 "File not found"
```

- dry-run 输出必须用 `redactDataUri`（`core/src/files/upload.ts:272`）脱敏，否则会把几 MB 的 base64 打到终端。
- **不能直接复用 `client.resolveImageInput`**：它在非 Token Plan 连接下会把本地图片上传到临时 OSS，而 OSS 上传需要把文件绑定到一个模型。向量和重排序模型能不能用临时 OSS，文档没写【待验证】。Data URI 在文档里有明确支持，更稳。

---

## 6. 坑位清单

### 通用（C）

| # | 坑 | 后果 | 对策 |
| --- | --- | --- | --- |
| P-C1 | **请求地址**：文档的示例已经统一改成 `https://{WorkspaceId}.cn-beijing.maas.aliyuncs.com`；CLI 默认是 `https://dashscope.aliyuncs.com`，而公共域名后续会被废弃（见本仓库 model-endpoint 方案） | 如果默认域名上没有这些接口，所有请求都会 404 | §8 第 1 步先 curl 验证。**如果默认域名不可用**，参照 `decision/shared.ts:181` 的 `resolveDecisionEndpoint`：用 `ctx.client.url(path, () => workspaceMaaSBaseUrl(ws, region))`，再加上 `--workspace-id` 和 `--region` 参数。 |
| P-C2 | **地域**：新加坡的模型很少。文本向量只有 3 个；多模态向量只有 `tongyi-embedding-vision-plus/flash`；重排序文档没给新加坡的模型列表【待验证】 | 用国际站配置调用北京独有的模型，会报模型不存在 | 能力表加一个 `regions` 字段。base URL 是新加坡时，如果模型不在列表里，提示"该模型在当前地域不可用"。只提示，不拦截。 |
| P-C3 | **SDK 和 HTTP 的参数结构不一样**：SDK 是扁平参数（`input`、`dimension`、`top_n`），HTTP 要嵌套成 `input.texts`、`input.contents`、`parameters.*` | 照着 SDK 示例写，请求会 400 或参数被忽略 | 只照着文档的 HTTP / curl 示例写，单测里断言请求体结构。 |
| P-C4 | **输出体积**：2560 维 × 20 条 ≈ 5 万个浮点数，JSON 大约 1MB；text 模式默认的格式化会把整个对象打出来 | 终端刷屏，Agent 的上下文被撑爆 | text 模式输出摘要；JSON 模式照常输出，但在帮助里推荐用 `--save`。 |
| P-C5 | **Token Plan**：内置的 `token-plan` Profile 的能力列表（`core/config/profile-presets.ts:26-37`）里没有 embedding 和 rerank | 当前 Profile 是 token-plan 时，命令会回退到 default Profile，并打印一条 fallback 提示（`runtime/src/middleware.ts:206-216`）。这是符合预期的行为，不是 bug | 不改能力列表；在 notes 里写一句"Token Plan 不支持向量和重排序"。Token Plan 网关到底支不支持【待验证】。 |
| P-C6 | **遥测隐私**：遥测只上报白名单里的参数（`core/telemetry/tracker.ts:36` 的 `PARAM_ALLOWLIST`） | 如果把 `text`、`query`、`document` 加进白名单，会上报用户原文 | 只加数值和枚举类参数；code review 时专门检查这一项。 |
| P-C7 | **数字参数解析太宽松**（runtime 的通用问题）：`Number("")` 为 0 | `--dimension ""` 会被当成 0，`--top-n ""` 也一样 | 在 `validate` 里显式检查 `Number.isInteger(n) && n > 0`，不依赖 runtime。 |
| P-C8 | **dry-run 不能有副作用**：`--save` 不写文件，不上传文件，不读 stdin 以外的大文件内容 | dry-run 被 Agent 当作安全预览使用，有副作用会出问题 | dry-run 分支在所有副作用之前 return，参照 `decision/predict.ts:156` 的写法。 |

### 文本向量（T）

| # | 坑 | 后果 | 对策 |
| --- | --- | --- | --- |
| P-T1 | **维度和模型绑定**：2560 只有 `qwen3.7-text-embedding` 支持，2048 / 1536 只有 v4 和 qwen3.7 支持，v1 / v2 固定 1536 | 传错维度会 400，而且报错信息不一定说清楚原因 | 已知模型在本地校验，报错时把这个模型的可选值列出来。 |
| P-T2 | **`instruct` 只在 `text_type=query` 时生效**；不传 `text_type` 时服务端会静默忽略它 | 用户以为指令生效了，其实没有 | `--instruct` 不配 `--text-type query` 时报 usage 错误。另外要说明：instruct 会按条数重复计 token，20 条就计 20 次。 |
| P-T3 | **单请求条数上限**：20 / 10 / 25；分批后多个请求并发容易触发限流 | 一次传太多条会 400；并发请求会触发 429 | 自动分批 + 串行请求。429 时按 `Retry-After` 退避重试最多 2 次（可选，不做也要在文档里说明）。 |
| P-T4 | **按行读文件**：空行、`\r\n`、UTF-8 BOM、末尾的换行都要处理；单行可能很长（128K token） | 空行会发给服务端变成空字符串（可能 400）；`\r` 会混进文本，导致向量变化 | 去掉 BOM，按 `/\r?\n/` 拆分，**跳过空行**，同时记录每行的原始行号 `line`，输出到 `--save` 的结果里。帮助文本里写明会跳过空行。 |
| P-T5 | **`dense&sparse` 里的 `&` 在 shell 里是后台运行符** | 用户写 `--output-type dense&sparse` 不加引号，命令会被截断 | 命令行取值用 `both`，内部映射；同时也接受加了引号的原值。 |
| P-T6 | **文档自相矛盾**：总览页说 flash 支持三种向量类型，同步接口页的 `output_type` 说明里没有列 flash；响应字段说明又说 sparse 只适用于 v3 和 v4 | 能力表可能写错 | 以实测为准：§8 第 2 步分别对 flash 和 qwen3.7 测一次 `dense&sparse`。 |
| P-T7 | **返回顺序**：文档没有保证 `embeddings` 按 `index` 顺序返回 | 直接按数组下标对应输入会错位 | 永远按 `index` 字段对应输入，合并后再排序。 |
| P-T8 | **分批中途失败** | 前面几批已经计费，结果却全丢了 | 失败时保留 `.partial` 文件，报错里写明进度和最后一个 `request_id`。 |
| P-T9 | **usage 字段不统一**：flash 返回 `prompt_tokens` 和 `total_tokens`，其他模型只返回 `total_tokens` | 合并时出现 `undefined + n = NaN` | 按字段逐个累加，缺失的字段当作 0，输出时去掉全为 0 的字段。 |
| P-T10 | **`encoding_format=base64`** 只在 OpenAI 兼容接口有，而且"老网关和长请求只返回 float" | 如果以后走兼容接口并依赖 base64，结果不可靠 | 本期走原生接口，不涉及；记在这里备忘。 |

### 多模态向量（M）

| # | 坑 | 后果 | 对策 |
| --- | --- | --- | --- |
| P-M1 | **四种融合机制**：`enable_fusion`、同一对象合并、永远融合、不支持 | 用错方式要么报错，要么**静默返回独立向量** | 按 §5.2 的规则改写请求。单测覆盖每个模型 × 传与不传 `--fusion`。 |
| P-M2 | **命令行参数之间没有顺序**：`--text a --image x --text b` 解析后，各参数之间的先后顺序丢失 | 用户以为 index 0 是 a、1 是 x，实际是按固定顺序组装的 | 固定按 text → image → video → multi_images 的顺序组装，帮助文本里写明；每条输出都带 `source`，需要精确顺序时用 `--contents-file`。 |
| P-M3 | **请求为空时不报错**：tongyi-embedding-vision 系列在 `contents` 为空时返回 **200 和空的 embeddings** | CLI 输出"成功"却没有向量 | 本地要求至少有一个元素；服务端返回空数组时也报错。 |
| P-M4 | **图片格式和大小**：老模型只支持 JPG / PNG / BMP；CLI 的 MIME 表（`core/files/upload.ts:203`）包含 `.heic`、`.tiff`、`.webp`；base64 会让体积膨胀约 33% | HEIC 编码后发过去会被服务端拒绝；3MB 的限制可能是指编码前的大小，编码后就超了 | 按能力表检查扩展名和**原始文件大小**。限制到底是指原始文件还是编码后【待验证】，保守起见按"编码后 ≤ 限制"来检查。 |
| P-M5 | **视频只支持 URL**；本地视频能不能走临时 OSS（`oss://`），文档没写 | 用户传本地视频会失败 | v1 直接报错："视频只支持公网 URL，可以先上传到 OSS 再传链接"。如果实测 `client.uploadFile(path, model)` 拿到的 `oss://` 可用，再放开【待验证】。 |
| P-M6 | **URL 里含中文**：示例 URL 里有 `+` 和中文 | 服务端可能下载失败 | 对含非 ASCII 字符的 URL 做 `encodeURI`（已经编码过的部分保持不变）。 |
| P-M7 | **`fps` 超出 [0, 1] 不报错**；`res_level`、`max_video_frames` 只有快照版支持，其他模型传了很可能被忽略 | 参数没生效，用户却不知道 | 在本地校验范围，并限制只有对应模型才能用这些参数。 |
| P-M8 | **usage 结构四个模型各不相同**：<br>• tongyi 系列有 `input_tokens_details`；<br>• qwen3-vl 的 `input_tokens` 只算文本，另有顶层 `image_tokens`；<br>• qwen2.5-vl 没有 `total_tokens`；<br>• v1 有 `image_count` 和 `duration` | 类型定义写死字段会出错；text 模式的用量摘要不准 | 类型定义宽松，原样透传；text 模式只显示存在的字段。 |
| P-M9 | **`multimodal-embedding-v1` 传 `dimension` 会报 400** | | 能力表里标记 `fixedDim`，传了就在本地报错。 |
| P-M10 | **单请求元素上限**：各模型差别很大（20 个元素、图片 10 / 64 / 8 张、视频 1 / 8 个；qwen2.5-vl 每种类型只能 1 个） | 超出时报 400 | 本地校验，报错时写明这个模型的上限。多模态**不自动分批**：融合向量没法拆，独立向量分批的语义也容易混淆，让用户自己拆。 |
| P-M11 | **`type` 字段的取值**：除了模态名，还有 `fused`、`fusion`、`vl` | 下游按 `type` 分支处理时会漏掉 | 文档和输出说明里把所有取值列全。 |

### 重排序（R）

| # | 坑 | 后果 | 对策 |
| --- | --- | --- | --- |
| P-R1 | **分数只在同一个请求内可比**，不能跨请求比较 | 如果像文本向量那样自动分批再合并排序，结果是错的 | **不分批**，文档数超过上限时直接报错，提示先用向量检索把候选集缩小（典型流程：向量召回 50–100 条 → 重排序取前 5–10 条）。 |
| P-R2 | **单条超长不截断，直接 400**：qwen3.7 单条 30K、qwen3 和 gte 单条 4K、vl 单条 8K；query 本身也有 4K 上限 | 一条长文档会让整个请求失败 | CLI 没法精确计算 token，只做字符数的粗略预警（比如 1 个字符约等于 1 token，超过上限时给 warning，不拦截）；报错时提示"是否有单条超长的文档"。 |
| P-R3 | **单请求 token 上限** = query × 文档数 + 所有文档之和；gte 只有 30K | 长 query 配上 500 条文档很容易超限 | 同上，粗略预警。 |
| P-R4 | **`--documents-file` 的格式**：按纯文本每行一条时，文档里不能有换行 | 多段落的文档会被拆成多条 | `.txt` 文件按行读；`.jsonl` 文件每行是一个 JSON 字符串或对象；`.json` 文件是数组。按扩展名分流，帮助文本里写明。 |
| P-R5 | **参数支持度不同**：`instruct` 在 gte 上不生效；`return_documents` 不支持 qwen3.7；`fps` 只有 vl 支持 | 参数被静默忽略或报错 | 能力表校验；`return_documents` 按 D4 永远不发。 |
| P-R6 | **OpenAI 兼容 rerank 的路径是 `/compatible-api/v1/reranks`**，不是 `/compatible-mode/v1`，而且请求体是扁平结构 | 以后如果改用兼容接口，很容易拼错路径 | 本期不用；记在这里备忘。 |
| P-R7 | **`gte-rerank` 已下线**，`gte-rerank-v2` 可能也会跟着下线 | 用户脚本里写死的模型会突然失效 | 默认值不用 gte；帮助文本里标注"gte-rerank-v2 不推荐"。 |
| P-R8 | **vl 的文档数上限按类型分别计算**（文本 100 / 图片 40 / 视频 4），混合时怎么算文档写的是"因类型而异" | 混合模态时的上限不清楚 | 本地按三种类型分别检查上限；混合模态的具体规则【待验证】。 |
| P-R9 | **`top_n` 大于文档数时返回全部** | 不是错误 | 不报错；输出里 `results` 的长度可能小于 `top_n`。 |

---

## 7. 测试计划

### 单测（`packages/core/tests/embedding-models.test.ts`）

- 能力表：每个已知模型的维度校验通过和失败的用例；未知模型直接透传。
- `--fusion` 改写：7 个模型 × 传与不传 `--fusion`，断言改写后的请求体（`enable_fusion`、合并对象、报错）。
- 读文件按行拆分：BOM、`\r\n`、空行、末尾换行、只有空行的文件（应报错）。
- 文本向量分批合并：25 条 / 每批 10 条 → 3 批，`index` 修正和 `usage` 累加正确；usage 字段缺失时不出现 NaN。
- 重排序回填：按 `index` 回填文本文档和多模态对象。

### e2e（本地 mock server，写法参照 `decision.e2e.test.ts`）

- `--help` 的中英文输出。
- 参数校验返回退出码 2：缺少输入；`--instruct` 不配 `--text-type query`；维度不合法；对不支持的模型传 `--fusion`；`--fps` 超范围；重排序文档数超限。
- dry-run：输出端点和请求体；Data URI 已脱敏；不发任何请求，也不写 `--save` 文件。
- 文本向量：25 条输入触发 3 次请求，断言 mock 收到的每个请求体，以及合并后的输出。
- 文本向量第 2 批返回 500：退出码非 0，报错写明"已完成 10/25"，并生成 `.partial` 文件。
- 多模态：本地图片转成 Data URI 后发送；mock 返回空 embeddings 时报错（P-M3）。
- 重排序：输出按 `rank` 排列，`document` 已回填；请求体里**没有** `return_documents`。
- 请求头：`Authorization: Bearer …`；有 `oss://` 时带 `X-DashScope-OssResourceResolve`。

### live e2e（需要真实 API Key，默认跳过）

每个命令一个最小用例，用默认模型。由已有的 `isDashScopeE2EReady()` 一类开关控制是否运行。

### 合并前检查

```bash
pnpm vp check
```

```bash
pnpm vp test packages/core/tests/embedding-models.test.ts packages/commands/tests/e2e/embedding-text.e2e.test.ts packages/commands/tests/e2e/embedding-multimodal.e2e.test.ts packages/commands/tests/e2e/rerank.e2e.test.ts
```

```bash
pnpm sync:skill-assets
```

最后一条运行后，`git status` 里应该只出现预期的 reference 文件变更。

---

## 8. 实施步骤（明天按顺序做）

每一步都写了完成标准，做完一步再做下一步。

**第 1 步：确认请求地址（约 30 分钟，必须最先做）**

在默认域名和 workspace 域名上各发一次请求，确认三个接口都能通。决定 D2 走哪种方案。

```bash
curl -sS https://dashscope.aliyuncs.com/api/v1/services/embeddings/text-embedding/text-embedding -H "Authorization: Bearer $DASHSCOPE_API_KEY" -H 'Content-Type: application/json' -d '{"model":"qwen3.7-text-embedding","input":{"texts":["你好"]},"parameters":{"dimension":256}}' | head -c 300
```

```bash
curl -sS https://dashscope.aliyuncs.com/api/v1/services/embeddings/multimodal-embedding/multimodal-embedding -H "Authorization: Bearer $DASHSCOPE_API_KEY" -H 'Content-Type: application/json' -d '{"model":"tongyi-embedding-vision-plus","input":{"contents":[{"text":"你好"}]}}' | head -c 300
```

```bash
curl -sS https://dashscope.aliyuncs.com/api/v1/services/rerank/text-rerank/text-rerank -H "Authorization: Bearer $DASHSCOPE_API_KEY" -H 'Content-Type: application/json' -d '{"model":"qwen3.7-text-rerank","input":{"query":"什么是排序模型","documents":["排序模型用于搜索","量子计算"]},"parameters":{"top_n":1}}'
```

再把域名换成 `https://<你的WorkspaceId>.cn-beijing.maas.aliyuncs.com` 各跑一次。

- **完成标准**：三个接口在默认域名上都返回 200，就采用方案 D2；任何一个返回 404 或 403，就改用 P-C1 里的 workspace 方案。

**第 2 步：实测文档里没写清的地方（约 30 分钟）**

| 待验证项 | 对应坑位 |
| --- | --- |
| `qwen3.7-text-embedding-flash` 是否支持 `dense&sparse` | P-T6 |
| 返回的 `index` 是否按顺序 | P-T7 |
| `qwen3.7-text-rerank` 传 `return_documents: true` 时的行为 | D4 |
| tongyi 系列传空 `contents` 时的行为 | P-M3 |
| 本地视频上传成 `oss://` 后能否用于向量化 | P-M5 |

- **完成标准**：把结果补进 §5.2 的能力表，表里不再有【待验证】。

**第 3 步：core 层（约 2 小时）**

新增 endpoints、类型和 `embedding/models.ts` 能力表，加上对应单测。

- **完成标准**：能力表和请求改写的单测全部通过。

**第 4 步：`bl rerank`（约 2 小时）**

最简单，先做它，跑通从参数到输出的整条链路。

- **完成标准**：mock e2e 通过；用真实 Key 跑通第 1 步里的例子，并且 `document` 字段已回填。

**第 5 步：`bl embedding text`（约 3 小时）**

包括分批、`--save` 和 `.partial` 文件。

- **完成标准**：25 条分批的 e2e 用例和失败用例都通过；`--save` 写出的 JSONL 能被 `jq` 逐行解析。

**第 6 步：`bl embedding multimodal`（约 4 小时）**

工作量最大，重点是 §5.2 的融合改写和各项上限校验。

- **完成标准**：7 个模型 × 传与不传 `--fusion` 的单测全部通过；本地图片和 URL 图片都能用真实 Key 跑通。

**第 7 步：收尾（约 1 小时）**

遥测白名单、`GROUP_OWNER_SKILL`、SKILL.md 路由说明、`pnpm sync:skill-assets`、CHANGELOG 中英文、`vp check`。

- **完成标准**：§7 的"合并前检查"全部通过。

---

## 9. 风险汇总

| 风险 | 可能性 | 影响 | 应对 |
| --- | --- | --- | --- |
| 默认域名上没有这些接口 | 中 | 高 | 第 1 步先验证；备选方案见 P-C1。 |
| 多模态模型能力表写错 | 中 | 中 | 第 2 步实测；未知模型直接透传，不阻塞使用。 |
| 模型更新快（文档里已经有 2026-03-06 快照版，还有 qwen3.7 系列） | 高 | 低 | 能力表集中放在一个文件；未知模型透传；默认模型用常量，方便以后调整。 |
| 新加坡地域支持的模型少 | 中 | 中 | P-C2 只提示不拦截。 |
| 向量输出太大，影响 Agent 使用 | 中 | 中 | text 模式只输出摘要；推荐 `--save`。 |

## 10. 待决策问题（实现前和产品对齐）

- **Q1 命令命名**：本文推荐 `bl embedding text|multimodal` + `bl rerank`。备选方案：
  - `bl text embed` + `bl vision embed` + `bl text rerank`：挂在已有的命令组下，但多模态重排序放不进去；
  - `bl embed` 一个命令按输入自动判断：更省事，但参数校验会很乱。
- **Q2 默认模型**：多模态向量默认用 `tongyi-embedding-vision-plus`（两个地域都可用）还是 `qwen3-vl-embedding`（效果更新，但只在北京可用）？文本向量默认用 `qwen3.7-text-embedding` 还是和 `bl batch` 示例保持一致，用 `text-embedding-v4`？
- **Q3 skill 归属**：`embedding` 和 `rerank` 归 `bailian-cli`（不改 `GROUP_OWNER_SKILL` 时的默认值），还是归 `bailian-knowledge`（场景上和 RAG 最贴近）？
- **Q4 `--save` 的格式**：只支持 JSONL，还是同时支持 `.npy` 或 CSV？建议 v1 只做 JSONL。
- **Q5 429 自动重试**：要不要做？CLI 里其他命令目前没有统一的重试机制，单独为这几个命令加可能不一致。

## 11. 参考文档（2026-11-24 抓取）

- 文本与多模态向量化总览：<https://help.aliyun.com/zh/model-studio/embedding>
- 文本向量同步接口：<https://docs.bailian.console.aliyun.com/zh/model-studio/text-embedding-synchronous-api>
- OpenAI 兼容 Embedding：<https://docs.bailian.console.aliyun.com/zh/model-studio/embedding-interfaces-compatible-with-openai>
- 文本向量批处理接口：<https://docs.bailian.console.aliyun.com/zh/model-studio/text-embedding-batch-api>
- 多模态向量：<https://docs.bailian.console.aliyun.com/zh/model-studio/multimodal-embedding-api-reference>
- 排序模型（Rerank）API：<https://docs.bailian.console.aliyun.com/zh/model-studio/text-rerank-api>
- 重排序使用指南：<https://docs.bailian.console.aliyun.com/zh/model-studio/rerank>
