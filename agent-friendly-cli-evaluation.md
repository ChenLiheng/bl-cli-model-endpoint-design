# CLI 的 Agent 友好度评估标准，及 bailian-cli 评估

> 日期：2026-11-09 · 状态：讨论稿
> 评估对象：`modelstudioai/cli` main `cadc301d`（bl 2.8.4），在非 TTY、干净 HOME、无凭证的环境下实测。

## 结论

- 业界缺少统一的「CLI 对 Agent 友好度」标准。本文提出一个 **10 个维度、100 分制** 的评估框架（ACF Score，Agent-friendly CLI Score）。每个检查项都有可自动执行的探测方法，分三层落地：静态检查、动态探测、Agent 任务评测。
- 按这个框架，bailian-cli 得 **68 / 100（可用，接近良好）**。它在「非交互与安全」上是我见过做得最好的 CLI 之一：高风险命令要求 `--yes`、确认失败返回退出码 7 和执行计划、有 `--dry-run`。输出契约（stdout 一个 JSON 文档、stderr JSONL）也很严谨。
- 主要短板是 **上下文成本** 与 **确定性**：
    - 根级 `--introspect` 有 1,062,846 字节，约 30 万 token；`--help` 有 38 KB，在非 TTY 下也带 ASCII 横幅，且写到 stderr。
    - 非 TTY 下默认输出仍是 text。
    - 大版本差距时会自动升级自身。
    - 本地错误只有数字码，没有稳定的字符串错误类型；服务端错误的退出码一律为 1。
- 第五节给出 16 项优化方案（A1–A16），按三个阶段落地：前两个阶段只增不改，预计总分分别到约 80、88 分；第三个阶段与模型端点改造一起作为 3.0 发布，预计约 92 分。

## 一、为什么需要一个标准

Agent 调用 CLI 的方式和人很不一样，人用着顺手的设计，对 Agent 可能是障碍。

| Agent 的特点 | 对 CLI 的要求 |
| --- | --- |
| 没有 TTY，没人能回答交互提示 | 任何操作都能用参数一次完成；需要确认时快速失败并说明怎么确认 |
| 读的是文本，每个字节都要花上下文 | 帮助与输出要短，可按需展开；不输出装饰性内容 |
| 依赖解析结果做下一步决策 | 结构化输出、稳定字段、stdout / stderr 分离 |
| 出错后要自己修复 | 错误可机器识别，带修复动作，区分可重试与不可重试 |
| 会犯拼写错误、会猜参数 | 未知参数给「你是不是想用」建议；参数校验在副作用之前完成 |
| 可能误操作 | 危险操作有门禁，支持预演，操作可审计 |
| 任务要可复现 | 不在运行中静默改变自身或环境；配置来源可见 |
| 常处理耗时任务 | 异步提交、任务 ID、可恢复轮询、明确超时 |

现有的相关规范只覆盖了一部分：[clig.dev](https://clig.dev) 面向人类可用性；POSIX / GNU 约定只管参数语法；MCP 解决的是协议，不评价 CLI 本身。因此需要一个面向 Agent、可量化、可自动化的评估框架。

## 二、评估框架：ACF Score

### 2.1 维度与权重

每个维度按 0–3 分打分，乘以权重后换算成百分制。

```text
总分 = Σ(维度得分 / 3 × 权重)
```

| # | 维度 | 权重 | 关注点 |
| --- | --- | --- | --- |
| D1 | 可发现性 | 15 | Agent 能否在不读源码的情况下，低成本地找到命令和参数 |
| D2 | 结构化输出 | 15 | 输出能否被稳定解析 |
| D3 | 错误可操作性 | 15 | 出错后 Agent 能否自行判断原因并修复 |
| D4 | 非交互与安全 | 15 | 无人值守能否跑完；危险操作是否有门禁 |
| D5 | 输入友好 | 8 | 复杂输入能否不经 shell 转义传入 |
| D6 | 长任务与异步 | 8 | 耗时任务能否提交、查询、恢复 |
| D7 | 上下文效率 | 10 | 帮助、Schema、输出的 token 成本 |
| D8 | 确定性与隔离 | 8 | 相同输入是否得到相同行为；是否会静默改变环境 |
| D9 | 认证 | 3 | 能否非交互地配置和检查凭证 |
| D10 | 生态集成 | 3 | 是否提供 Skill / MCP / 机器可读文档 |

权重的依据是：D1–D4 决定 Agent「能不能用」，D5–D8 决定「用得好不好」，D9–D10 是加分项。权重本身也是可讨论的参数，见文末「待讨论」。

### 2.2 评分等级

| 分数 | 等级 | 含义 |
| --- | --- | --- |
| 85–100 | 优秀 | Agent 可以只凭 Schema 无人值守地完成大多数任务 |
| 70–84 | 良好 | 需要少量提示词或 Skill 补充 |
| 50–69 | 可用 | 能用，但会浪费较多 token 或需要较多重试 |
| < 50 | 不友好 | 需要人工介入或专门封装 |

### 2.3 检查项

每个维度的 0–3 分按下表判定。「测量方法」一列都可以写成脚本，见第三节。

**D1 可发现性**

| 检查项 | 测量方法 |
| --- | --- |
| D1.1 有机器可读的命令 Schema（命令树、参数类型、必填、枚举、默认值） | 执行 `<cli> --introspect` 或同类命令，校验 JSON 结构 |
| D1.2 Schema 可按命令取子集，也有只含路径和一句话描述的精简索引 | 对比全量与子集的字节数 |
| D1.3 `--help` 写 stdout、退出码 0；非 TTY 下没有横幅和颜色码 | `<cli> --help </dev/null 2>/dev/null \| wc -c` |
| D1.4 每个命令带可直接运行的示例 | 统计 Schema 中 examples 的覆盖率 |
| D1.5 Schema 声明了输出结构（至少列出顶层字段） | 检查 Schema 是否包含 output 字段 |

计分：0 分为只有人类可读的 help；1 分为有 Schema 但不完整；2 分为 Schema 完整但成本高或缺输出结构；3 分为以上全部满足。

**D2 结构化输出**

| 检查项 | 测量方法 |
| --- | --- |
| D2.1 有 JSON 输出模式，且 stdout 只有一个合法 JSON 文档 | 用 `json.loads(stdout)` 解析 |
| D2.2 进度、提示、警告走 stderr，JSON 模式下也是结构化的 | 逐行解析 stderr |
| D2.3 非 TTY 时默认输出机器格式，或支持一个环境变量全局切换 | 不带 `--output` 运行，检查 stdout 能否解析 |
| D2.4 字段名稳定，变更有版本号 | Schema 带 `schema_version`；CHANGELOG 有 Breaking 标注 |
| D2.5 支持字段选择或投影（如 `--fields`、`--jq`） | 检查是否存在这类参数 |

**D3 错误可操作性**

| 检查项 | 测量方法 |
| --- | --- |
| D3.1 退出码有语义（用法、认证、配额、网络、超时、需确认等），且写进文档 | 构造各类失败，检查退出码 |
| D3.2 错误对象有稳定的字符串类型，例如 `AUTH_MISSING_API_KEY` | 解析错误 JSON |
| D3.3 错误对象标明是否可重试，以及建议的等待时间 | 构造 429 或 5xx |
| D3.4 修复建议是可执行的命令，放在结构化字段里，而不只是一段文字 | 解析 hint 字段 |
| D3.5 未知命令、未知参数给出「你是不是想用」建议 | 故意拼错参数 |
| D3.6 服务端错误保留 HTTP 状态码、业务错误码和 Request ID | Mock 服务端返回错误 |

**D4 非交互与安全**

| 检查项 | 测量方法 |
| --- | --- |
| D4.1 stdin 不是 TTY 时绝不阻塞等待输入 | 加 `</dev/null` 并设超时运行全部命令 |
| D4.2 破坏性操作要求显式确认参数（如 `--yes`），未确认时快速失败并返回专用退出码 | 不带 `--yes` 运行删除类命令 |
| D4.3 未确认时返回将要执行的计划 | 检查错误 JSON |
| D4.4 支持 `--dry-run`，输出请求的方法、URL 和请求体 | 运行 `--dry-run` |
| D4.5 创建类操作支持幂等键，或给出「已存在」的明确信号 | 重复执行同一个创建命令 |

**D5 输入友好**

| 检查项 | 测量方法 |
| --- | --- |
| D5.1 复杂结构（JSON、长文本）可从文件或 stdin 读入（`-` 表示 stdin） | 检查 Schema 中带 `file` 类参数的命令 |
| D5.2 可重复参数、布尔参数的语法一致 | 静态扫描 Schema |
| D5.3 参数命名在各命令间一致（如 `--workspace-id` 不出现多种写法） | 静态扫描同义参数 |

**D6 长任务与异步**

| 检查项 | 测量方法 |
| --- | --- |
| D6.1 异步任务可只提交、立即返回任务 ID（如 `--no-wait`） | 检查 Schema |
| D6.2 可用任务 ID 查询状态、下载结果；中断后可恢复 | 中断后重新查询 |
| D6.3 超时可配置，超时退出码有专门语义 | 设置一个极短的 `--timeout` |

**D7 上下文效率**

| 检查项 | 测量方法 |
| --- | --- |
| D7.1 根级 help 不超过 8 KB，单命令 help 不超过 4 KB | 测字节数 |
| D7.2 根级 Schema 不超过 50 KB，或有不超过 20 KB 的精简索引 | 测字节数 |
| D7.3 Schema 和 help 只输出一种语言（可指定） | 检查是否有多语言并存 |
| D7.4 列表类命令默认分页，并给出下一页游标 | 运行 list 命令 |
| D7.5 非 TTY 下没有 ANSI 颜色码、横幅、动画 | 用正则匹配 `\x1b\[` 和横幅字符 |

**D8 确定性与隔离**

| 检查项 | 测量方法 |
| --- | --- |
| D8.1 不会在运行时自动升级自身；升级只能显式触发 | 检查代码，或注入一个更高的版本号 |
| D8.2 配置的优先级明确（flag > env > 配置文件），且能查看最终生效值及其来源 | `config show` |
| D8.3 同一份配置下，不同命令对同一概念（如地址、空间）的解析一致 | 交叉检查 |
| D8.4 遥测、更新检查可以用环境变量关闭，并写进文档 | 检查文档和环境变量 |
| D8.5 语言、时区、颜色不依赖宿主环境的隐式设置 | 在不同 `LANG` 下运行 |

**D9 认证**：凭证可通过环境变量注入；有非交互的登录方式；`auth status` 能输出结构化结果。

**D10 生态集成**：提供 Agent Skill 或 MCP Server；文档与 CLI 版本同步发布；有 `llms.txt` 一类的入口。

## 三、怎么落地：三层评估

| 层 | 做什么 | 产出 | 成本 |
| --- | --- | --- | --- |
| L1 静态检查 | 读 Schema、help 与源码，执行 D1、D5、D7 的大部分检查项 | 维度得分 + 违规清单 | 低，可放进 CI |
| L2 动态探测 | 在干净 HOME、非 TTY、无网络或 Mock 服务端的沙箱里运行一组探测命令，检查 D2、D3、D4、D6、D8 | 维度得分 + 每项的实际输出作为证据 | 中，可放进 CI |
| L3 Agent 任务评测 | 准备 20–30 个真实任务（如「生成一张图并保存到 out/」「查询某任务状态」），让 2–3 个不同的 Agent 只凭 `--introspect` 和 Skill 完成 | 任务成功率、平均工具调用次数、平均 token、错误自愈率、越权操作次数 | 高，按版本跑 |

L1 和 L2 可以写成一个通用工具，例如 `agent-cli-probe <bin>`：输入一个 CLI，输出分数卡 JSON 和 Markdown 报告，因此可以横向对比不同的 CLI。L3 是最终的验收标准。L1 和 L2 分数高、L3 成功率低，说明 Schema 或文档的「语义」有问题。例如参数描述有歧义，这类问题光靠格式检查发现不了。

L3 建议的核心指标：

| 指标 | 定义 | 目标（建议） |
| --- | --- | --- |
| 任务成功率 | 一次会话内完成且结果正确的比例 | ≥ 90% |
| 平均调用次数 | 完成一个任务的 CLI 调用次数 | ≤ 3 |
| 发现成本 | 第一次正确调用前读取 help 或 Schema 消耗的 token | ≤ 5k |
| 错误自愈率 | 出错后不靠人工、在 2 次重试内修复的比例 | ≥ 80% |
| 越权操作 | 未经确认执行破坏性操作的次数 | 0 |

## 四、bailian-cli 评估结果

### 4.1 得分

| 维度 | 权重 | 得分（0–3） | 折算 | 一句话 |
| --- | --- | --- | --- | --- |
| D1 可发现性 | 15 | 2 | 10.0 | `--introspect` 很完整，但全量过大、help 在 stderr 且带横幅、缺默认值与输出结构 |
| D2 结构化输出 | 15 | 2 | 10.0 | 输出契约严谨；非 TTY 默认仍为 text，没有字段选择 |
| D3 错误可操作性 | 15 | 2 | 10.0 | 退出码语义完整；缺字符串错误类型、可重试标记和未知参数建议 |
| D4 非交互与安全 | 15 | 3 | 15.0 | `--yes` + 退出码 7 + 执行计划 + `--dry-run`，设计成熟 |
| D5 输入友好 | 8 | 2 | 5.3 | 支持 `--messages-file -`；工具类 JSON 仍需要在命令行里转义 |
| D6 长任务与异步 | 8 | 2 | 5.3 | 有 `--no-wait`、`task get`、`batch get`；恢复语义未验证 |
| D7 上下文效率 | 10 | 1 | 3.3 | 根级 Schema 1 MB，中英双语并存；help 38 KB |
| D8 确定性与隔离 | 8 | 1 | 2.7 | 会自动升级；模型地址有两套解析机制；遥测默认开启 |
| D9 认证 | 3 | 3 | 3.0 | 支持环境变量注入；`auth status --output json` 结构清晰 |
| D10 生态集成 | 3 | 3 | 3.0 | 9 个 Skill 随版本同步，可配置 MCP |
| **合计** | 100 | | **67.7** | **可用，接近良好** |

### 4.2 做得好的地方（建议保持）

- **退出码有语义并对外公开**：`0 / 1 / 2 USAGE / 3 AUTH / 4 QUOTA / 5 TIMEOUT / 6 NETWORK / 7 CONFIRMATION_REQUIRED / 10 CONTENT_FILTER / 130`，并写进了 `--introspect` 的 `exitCodes` 字段（`packages/core/src/errors/codes.ts`）。
- **高风险操作门禁**：约 41 个命令文件声明了 `risk`（grep 统计），runtime 统一注入 `--yes`。未确认时抛 `ConfirmationRequiredError`，退出码 7，并附上 plan（`packages/runtime/src/confirm.ts`）。这正是 Agent 需要的「先问再做」协议。
- **输出契约**：stdout 只有一个 JSON 文档，stderr 每行一条 JSON；JSON 模式下隐藏进度；quiet 不会隐藏警告和错误（`docs/agents/output-contract.md`）。很多 CLI 做不到这一点。
- **错误处理边界**：服务端错误原样透传，保留 HTTP 状态码、错误码和 Request ID，不做二次包装（AGENTS.md 第 3 条）。这避免了 Agent 被 CLI 的「翻译」误导。
- **未知命令提示同级命令**：`bl text chta` 会列出 `text chat`，退出码 2。
- **按命令取 Schema**：`bl text chat --introspect` 只返回这一个命令，带 `required`（299 处）和 `choices`（89 处）。
- **Skill 体系**：Skill 与 reference 由命令表自动生成，版本号自动同步，不容易和 CLI 实际行为脱节。

### 4.3 不够友好的地方（附实测证据）

| # | 问题 | 证据 | 影响 |
| --- | --- | --- | --- |
| F1 | 根级 `--introspect` 过大 | `bl --introspect` 有 1,062,846 字节，共 334 个命令，每段描述都同时包含 en-US 和 zh-CN | Agent 只要不小心执行一次根级 introspect，就会占掉大半个上下文窗口，甚至被截断 |
| F2 | `--help` 写 stderr，且非 TTY 下仍有横幅 | `bl --help 2>/dev/null` 输出 0 字节；`2>&1` 后有 38,223 字节，其中 5 行是 ASCII 横幅 | 很多 Agent 框架默认只取 stdout，会拿到空结果；横幅白白消耗 token |
| F3 | 非 TTY 默认输出 text | `detectOutputFormat()` 在未指定时返回 `"text"`（`packages/core/src/output/formatter.ts:6-11`） | Agent 每次都要记得加 `--output json`，或预先设置 `DASHSCOPE_OUTPUT`；忘了就得去解析自由文本 |
| F4 | 本地错误只有数字码 | 本地错误：`{"error":{"code":3,"message":"No API key found.","hint":"Set DASHSCOPE_API_KEY, …\nbl auth login …"}}`。服务端错误已带 `http_status`、`api_code`、`request_id`，但退出码一律为 1（见 `bailian-cli-code-review.md` B4） | 同一个退出码下的不同原因（缺 key、key 无效、key 过期）无法区分；hint 是多行文本，修复命令要靠 Agent 自己抽取 |
| F5 | 未知参数没有建议 | `bl text chat --mesage hi` 返回 `Unknown flag "--mesage". Run with --help…` | 命令拼错时有建议，参数拼错却没有，Agent 只能再读一遍 help |
| F6 | Schema 没有结构化的默认值和输出结构 | flag 只有 `description / type / valueHint / required / choices` 五个键；默认值只写在描述里，如「(default: 4096)」；`constraints.crossFlagValidation: "not-exported"` | Agent 无法可靠地知道默认行为，也无法预知结果有哪些字段 |
| F7 | 运行时自动升级 | `versionCheckStage` 在大版本差距时调用 `performAutoUpdate`（`packages/runtime/src/middleware.ts:299-330`），只有设置 `NO_UPDATE_NOTIFIER=1` 才关闭 | 同一个 Agent 任务的前后两次调用可能跑在不同大版本上，结果不可复现 |
| F8 | 模型地址解析不一致 | `base_url` 与 `workspace_id` 两套机制，不同命令取舍不同（详见同仓库的 `model-endpoint-proposal.md`） | 同一份配置下，chat 和知识库可能打到不同的空间，Agent 很难排查 |
| F9 | `--dry-run` 不显示目标地址 | `bl text chat --message hi --dry-run --output json` 只输出 `request` 请求体，没有方法和 URL | Agent 无法在执行前确认请求会发往哪个地址、哪个空间 |
| F10 | 部分清除类操作没有门禁 | `bl auth logout` 不带 `--yes` 就直接执行，且默认作用域是 `all` | 本地凭证会被误清。影响较小，但与高风险门禁的设计不一致 |
| F11 | Skill 总量偏大 | 9 个 SKILL.md 共 127,803 字节，reference 目录约 1.3 MB | 按需加载时问题不大；但 hub skill `bailian-cli` 本身有 22 KB，加载一次的成本偏高 |

未实测的部分：真实 API 调用（没有凭证）、构建产物的启动耗时（这次是用 tsx 源码运行的，不具代表性），以及 L3 Agent 任务评测。D5、D6 的得分是根据 Schema 和代码给出的，没有端到端验证。

## 五、优化方案

本节针对 4.3 的 F1–F11，以及代码审查中影响 Agent 使用的 B4、B5、D3，逐项给出实现方案。代码位置以 `modelstudioai/cli` main `cadc301d` 为准。

### 5.1 总览

| # | 方案 | 解决 | 维度 | 改动层 | 破坏性 | 工作量 | 阶段 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| A1 | introspect 分级：index / 单命令 / 全量，并只输出一种语言 | F1 | D1 D7 | runtime | 有：根级默认改为 index | 2 人天 | 1（新增 `--introspect=index`）、3（改默认） |
| A2 | help 写 stdout；非 TTY 不打印横幅和欢迎页 | F2 | D1 D7 | runtime | 轻微 | 0.5 人天 | 1 |
| A3 | Agent 模式（`BL_AGENT=1`）：默认 JSON、无装饰、不自动升级 | F3 F7 | D2 D8 | core + runtime | 无（需显式开启） | 1.5 人天 | 2 |
| A4 | 结构化错误：`type`、`retryable`、`fix_commands` | F4 | D3 | core + runtime | 无（只增字段） | 2 人天 | 1 |
| A5 | 未知参数给出「你是不是想用」建议 | F5 | D3 | runtime | 无 | 0.5 人天 | 1 |
| A6 | Schema 增加 `default`、数值约束和输出结构 | F6 | D1 | core + runtime + commands | 无（只增字段） | 3 人天 | 1–2 |
| A7 | 自动升级只在交互式 TTY 下执行 | F7 | D8 | runtime | 轻微 | 0.5 人天 | 1 |
| A8 | `--dry-run` 统一输出方法和 URL | F9 | D4 | core + commands | 无（只增字段） | 2 人天 | 2 |
| A9 | 模型地址统一 | F8 | D8 | 全部 | 有 | 见端点改造方案 | 3 |
| A10 | `--fields` 输出投影 | D2.5 | D2 D7 | runtime | 无 | 1 人天 | 2 |
| A11 | `auth logout --scope all` 纳入确认门禁 | F10 | D4 | commands | 有 | 0.5 人天 | 3 |
| A12 | 精简 hub skill | F11 | D7 | skills | 无 | 1 人天 | 2 |
| A13 | HTTP 状态码映射到退出码 | B4 | D3 | core | 有 | 0.5 人天 | 3 |
| A14 | 屏蔽代理实验性警告，保证 stderr 是 JSONL | B5 | D2 | runtime | 无 | 0.5 人天 | 1 |
| A15 | 幂等请求自动重试并遵守 `Retry-After` | D3 | D3 D6 | core | 无 | 1.5 人天 | 2 |
| A16 | `agent-probe` 探测脚本接入 CI | — | 防回退 | tools | 无 | 2 人天 | 1 |

工作量是一位熟悉该仓库的工程师的粗估，包括单测。

### 5.2 A1：introspect 分级，单一语言

**现状**：`runtime/src/create-cli.ts:256` 对任何路径都调用 `buildCommandSchema(registry, identity, res.path)`；根路径会把 334 个命令的全部字段、中英两种语言一起输出，共 1,062,846 字节。`runtime/src/introspect.ts:114-120` 对 description、notes、examples 都保留 `LocalizedText` 双语对象。

**方案**：

1. 新增三个层级，由 `--introspect` 的取值选择：

    | 调用 | 输出 | 目标大小 |
    | --- | --- | --- |
    | `bl --introspect=index` | 每个命令只有 `path`、`auth`、`summary`（一句话、单语言）、`risk`、`async` | ≤ 20 KB |
    | `bl <path> --introspect` | 单个命令的完整 Schema（现状），单语言 | ≤ 8 KB |
    | `bl --introspect=full` | 现在的全量输出，单语言 | 约 450 KB |

2. 根级 `bl --introspect` 不带取值时：阶段 1 保持现状，并在 stderr 输出一条 `INTROSPECT_LARGE` warning，提示改用 `=index`；阶段 3 改为默认输出 index。
3. 语言：introspect 默认 en-US。新增 `--lang <en-US|zh-CN>`，也可读取 `language` 配置。`LocalizedText` 在 `buildCommandSchema` 内部按语言展开为字符串。`schema_version` 升到 2，以便调用方区分。
4. 在 `introspect.ts` 中新增 `buildCommandIndex(registry, identity, lang)`，复用 registry 的遍历逻辑。`summary` 直接取 `description` 的第一句。

```ts
// runtime/src/introspect.ts（草图）
export interface CommandIndexEntry {
  path: string[];
  auth: AuthRequirement;
  summary: string;          // 单语言，取 description 第一句
  risk?: CommandRisk;       // 有风险门禁的命令
  async?: boolean;          // 支持 --no-wait 的命令
}
export function buildCommandIndex(registry, identity, lang): { bin; version; schema_version: 2; commands: CommandIndexEntry[] }
```

`args.ts:115` 目前只识别 `--introspect` 开关，需要改为接受可选值：`--introspect`、`--introspect=index`、`--introspect=full`。

**测试**：给 index 和单命令 Schema 的字节数设上限断言（20 KB / 8 KB），防止回退；为 `schema_version: 2` 补一份快照测试。

### 5.3 A2：help 写 stdout，非 TTY 不打印装饰

**现状**：`create-cli.ts:204` 和 `:321` 把 help 写到 `process.stderr`；根 help 的 ASCII 横幅在 `registry.ts:393-399`，欢迎页在 `output/banner.ts:20-35`，两者都不判断 TTY。

**方案**：

- 用户显式传 `--help` 时写 stdout，退出码 0，与 POSIX 和 gh、kubectl 等常见 CLI 一致。
- 命令没写完时自动打印的提示性 help（`create-cli.ts:320-322`，退出码 0）继续写 stderr：用户没有请求 help，不应占用 stdout。
- `printRootHelp` 和 `printWelcomeBanner` 只在 `out.isTTY` 时输出横幅和 emoji；非 TTY 下只保留命令表。
- 非 TTY 下根 help 的命令表改成每行 `path  summary` 的紧凑格式，预计从 38 KB 降到约 15 KB。

**兼容性**：依赖「help 在 stderr」的脚本极少；在 CHANGELOG 中注明。

### 5.4 A3：Agent 模式

**现状**：`core/src/output/formatter.ts:6-11` 在未显式指定时总是返回 `"text"`；自动升级与横幅都不区分调用方。

**方案**：引入一个明确的开关，Agent 只需要设置一次：

| 触发方式 | 优先级 |
| --- | --- |
| `BL_AGENT=1`（推荐写进 Agent 的环境配置或 Skill） | 最高 |
| `--agent` 全局 flag（单次调用） | 同上 |
| 自动识别常见 Agent 环境变量（如 `CLAUDECODE`、`CODEX_*`、`QWEN_CODE_*`） | 默认关闭，见待讨论 2 |

Agent 模式下各项默认值变化如下，用户显式传入的参数仍然优先：

| 项目 | 普通模式 | Agent 模式 |
| --- | --- | --- |
| 输出格式 | text | json |
| 横幅、颜色、emoji、进度动画 | TTY 时开启 | 关闭 |
| 流式输出 | TTY 时开启 | 关闭（一次性返回结果） |
| 自动升级 | 大版本差距时执行 | 只输出 `UPDATE_AVAILABLE` warning |
| help / introspect 语言 | 跟随配置 | en-US |
| 根级 introspect | 全量 | index |

实现：在 `core/src/config/loader.ts` 的 `buildOutputSettings` 中读取 `env.BL_AGENT`，得到 `agentMode: boolean` 并放进 `Settings`；`detectOutputFormat(flagValue, { agentMode })` 在未显式指定时返回 `"json"`；`versionCheckStage`、banner、stream 默认值各自读取 `settings.agentMode`。`auth status` 和 `config show` 的结果中增加 `agent_mode` 字段，方便排查。

Skill（`skills/bailian-protocol/SKILL.md`）里加一句：执行任何 bl 命令前先 `export BL_AGENT=1`。

### 5.5 A4：结构化错误

**现状**：`core/src/errors/base.ts:36-49` 的 `toJSON()` 输出 `code`（即退出码）、`message`、多行文本 `hint`，服务端错误另带 `http_status`、`api_code`、`request_id`。本地错误无法区分具体原因。

**方案**：只增字段，不改已有字段。

```jsonc
{
  "error": {
    "code": 3,                              // 退出码，保持不变
    "type": "AUTH_MISSING_API_KEY",         // 新增：稳定的大写字符串
    "message": "No API key found.",
    "hint": "…",                            // 保留，给人看
    "fix_commands": [                       // 新增：可直接执行的修复命令
      "bl auth login --api-key <key>",
      "export DASHSCOPE_API_KEY=<key>"
    ],
    "retryable": false,                     // 新增
    "retry_after_ms": 2000,                 // 新增：仅在可重试且服务端给出 Retry-After 时出现
    "http_status": 401, "api_code": "…", "request_id": "…"   // 服务端错误，保持不变
  }
}
```

实现：

1. `BailianErrorOptions` 增加 `type?: string`、`fixCommands?: string[]`、`retryable?: boolean`、`retryAfterMs?: number`。
2. 新建 `core/src/errors/types.ts`，集中登记错误类型常量，作为对外契约。首批覆盖约 15 个最常见的本地错误：缺 key、未知命令、未知参数、参数缺失、参数取值非法、需要确认、超时、DNS 失败、连接拒绝、代理错误、文件不存在、权限不足、配置损坏、workspace 缺失、地域不支持。
3. 服务端错误的 `type` 统一为 `API_ERROR`，具体原因看 `http_status` 和 `api_code`，不翻译服务端错误，遵守 AGENTS.md 第 3 条。
4. `retryable` 的判定复用 `core/src/utils/retry.ts` 中的 `isRetryable`（目前未导出，需要导出）。
5. `fix_commands` 中的命令用 `formatCommand(identity, …)` 生成，不写死 `bl`。
6. `--introspect` 的顶层增加 `errorTypes` 列表，与 `exitCodes` 并列。

**测试**：为每个已登记的 `type` 写一个触发用例，断言 JSON 结构；增加一个检查，确保 `types.ts` 中登记的类型都被代码使用过，避免再出现 B4 那种「公布了但从不使用」的情况。

### 5.6 A5：未知参数建议

**现状**：未知命令会列出同级命令（`registry.ts:241`），但未知参数只提示 `Run with --help`（`args.ts:166-170`）。

**方案**：在 `args.ts` 抛错前，对当前命令可用的 flag 名计算编辑距离（Damerau–Levenshtein，阈值 ≤ 2，或前缀匹配），取最接近的 1–3 个：

```text
Unknown flag "--mesage". Did you mean "--message"?
```

JSON 错误中同时给出 `"suggestions": ["--message"]`，`fix_commands` 给出替换后的完整命令。未知命令也补上同样的编辑距离逻辑（目前只列出同级命令）。

### 5.7 A6：Schema 补全

**现状**：flag 定义只有 `description / type / valueHint / required / choices`；默认值写在描述文字里，如「(default: 4096)」；跨参数约束标记为 `not-exported`；没有输出结构。

**方案**：

1. `core/src/types/command.ts` 的 `FlagDef` 增加可选字段：`default?: unknown`、`integer?: boolean`、`min?: number`、`max?: number`、`deprecated?: string`、`env?: string`（如 `DASHSCOPE_API_KEY`）。
2. runtime 在参数解析阶段统一执行 `integer / min / max` 校验，顺带修复代码审查中的 B7。
3. `defineCommand` 增加可选的 `output?: OutputSchema`。用一个精简的 JSON Schema 子集描述顶层字段，例如 `{ type: "object", properties: { task_id: { type: "string" }, status: { type: "string", enum: […] } } }`。introspect 原样导出。
4. 跨参数约束（互斥、依赖）声明为数据：`constraints: [{ kind: "exclusive", flags: ["message", "messagesFile"] }, { kind: "requires", flag: "workspaceId", needs: ["prime"] }]`，同时用于校验和导出，替代现在写在 `validate` 函数里的命令式逻辑。
5. 迁移顺序：先改 text chat、image generate、video generate、batch、knowledge search 等最常用的 20 个命令，其余命令逐步补齐。用 lint 规则统计 `default` 与 `output` 的覆盖率，纳入 A16 的分数。

### 5.8 A7：自动升级只在交互式 TTY 下执行

**现状**：`runtime/src/middleware.ts:299-330` 在大版本差距时调用 `performAutoUpdate`，只能用 `NO_UPDATE_NOTIFIER=1` 关闭。

**方案**：只有同时满足 `process.stdin.isTTY && process.stdout.isTTY && !settings.agentMode && !process.env.CI` 时才自动升级，其余情况只输出 `UPDATE_AVAILABLE` warning。同时修复代码审查 B3：拿不到校验和时拒绝安装（fail closed）。

### 5.9 A8：`--dry-run` 统一输出方法和 URL

**现状**：仓库中约 278 处 `settings.dryRun` 分支，每个命令各自拼预览对象。例如 text chat 在 `commands/text/chat.ts:352-358` 只在 batch 或 prime 模式下输出 `endpoint`，普通模式只有 `request`。

**方案**：在 `Client` 上加一个预览方法，命令只需要把原本要发出的请求交给它：

```ts
// core/src/client/client.ts（草图）
preview(opts: ClientRequestOpts): DryRunPreview {
  const { url, headers, body, method } = this.toOpts(opts);   // 与真实请求同一套拼装逻辑
  return { method: method ?? "GET", url, headers: redact(headers), body };
}
```

- 输出结构统一为 `{ method, url, headers（已脱敏）, body }`，命令可以在外层附加自己的字段（如 `timeout`）。
- 由于复用 `toOpts`，预览的 URL 与真实请求必然一致，这在模型端点改造后尤其重要。
- 迁移：先改常用的 20 个命令；增加一条 lint 规则，新代码的 `dryRun` 分支必须调用 `client.preview`。

### 5.10 A10：`--fields` 输出投影

**方案**：在 runtime 输出层（`output/reporter.ts` 提交 result 前）支持全局 `--fields a,b.c,items[].id`，只对 JSON 模式生效。路径语法只支持点号和 `[]`，不引入完整的 jq。投影发生在序列化之前，不影响命令逻辑。对 list 类命令，可以把 token 消耗降低一个数量级。

### 5.11 A11：`auth logout` 纳入确认门禁

**方案**：`auth logout` 不带 `--scope` 时默认只清当前域（API Key）；`--scope all` 声明 `risk`，非 TTY 下必须带 `--yes`，否则以退出码 7 返回将要清除的键列表。属于破坏性变更，放在阶段 3。

### 5.12 A12：精简 hub skill

**方案**：`skills/bailian-cli/SKILL.md`（22 KB）只保留路由表与通用约定，目标 ≤ 6 KB。命令细节交给各领域 skill 和 `bl <path> --introspect` 按需获取。SKILL.md 中写明「需要参数细节时执行 `bl <path> --introspect`」，不再复制参数表。

### 5.13 A13：HTTP 状态码映射到退出码

**方案**：在 `core/src/errors/api.ts` 的 `mapApiError` 中只映射退出码，不改 message：

| HTTP 状态 | 退出码 |
| --- | --- |
| 401、403 | 3 AUTH |
| 429 | 4 QUOTA |
| 408、504 | 5 TIMEOUT |
| 内容审核类 `api_code`（需与后端确认枚举） | 10 CONTENT_FILTER |
| 其他 | 1 GENERAL |

这会改变已有脚本看到的退出码，放在阶段 3，并在 CHANGELOG 中列出对照表。

### 5.14 A14：stderr 保持 JSONL

**方案**：在 `create-cli.ts` 启动时注册 `process.on("warning")` 过滤器：`code === "UNDICI-EHPA"` 的警告直接丢弃，其余 Node 警告在 JSON 模式下转为 `{"warning":{"code":"NODE_WARNING",…}}` 单行输出。长期看，改用非实验性的 `ProxyAgent` 自行读取代理环境变量。输出契约测试中增加设置了 `HTTPS_PROXY` 的用例。

### 5.15 A15：幂等请求自动重试

**方案**：在 `Client.request` 层接入已有的 `withRetry`（`core/src/utils/retry.ts`）：

- 只对 GET 请求和异步任务查询自动重试；提交类 POST 默认不重试，避免重复创建任务。
- 重试条件：429、502、503、504、网络错误；最多 3 次。若有 `Retry-After` 头则按它等待，否则指数退避，上限 10 秒。
- 每次重试在 stderr 输出一条 `RETRYING` debug 记录。
- 新增全局 `--retries <n>`（0 表示关闭）；Agent 模式下默认值相同。
- 不可重试的错误在 A4 的结构中标记 `retryable: false`，由调用方决定是否重新提交。

### 5.16 A16：`agent-probe` 接入 CI

**方案**：新增 `tools/agent-probe/`，实现第三节的 L1 和 L2：

1. 在干净 HOME、非 TTY、`BL_AGENT` 开和关两种模式下运行一组固定探测（help、introspect、未知参数、未知命令、缺凭证、dry-run、Mock 服务端返回 401 / 429 / 5xx、代理环境变量）。
2. 按 2.3 的检查项逐条判定，输出 `agent-probe.json` 分数卡和 Markdown 报告。
3. 接入 CI：总分下降、或任一字节数上限被突破时失败。
4. 脚本保持与 bailian-cli 解耦（输入只是一个可执行文件路径），以后可用于横向对比其他 CLI（见待讨论 4）。

## 六、实施路线

分三个阶段推进，前两个阶段不破坏现有行为，第三个阶段与模型端点改造一起作为大版本发布。

| 阶段 | 版本 | 内容 | 预计 ACF 总分 |
| --- | --- | --- | --- |
| 当前 | 2.8.4 | — | 68 |
| 阶段 1：只增不改 | 2.9.0 | A1（新增 `=index`、单语言）、A2、A4、A5、A6（常用 20 个命令）、A7、A14、A16 | 约 80 |
| 阶段 2：Agent 模式 | 2.10.0 | A3、A6（其余命令）、A8、A10、A12、A15 | 约 88 |
| 阶段 3：破坏性变更 | 3.0.0 | A1（根级默认 index）、A9、A11、A13；评估是否让非 TTY 默认 JSON（待讨论 3） | 约 92 |

各阶段预计分数按维度估算如下（0–3 分）：

| 维度 | 当前 | 阶段 1 | 阶段 2 | 阶段 3 |
| --- | --- | --- | --- | --- |
| D1 可发现性 | 2 | 3 | 3 | 3 |
| D2 结构化输出 | 2 | 2 | 3 | 3 |
| D3 错误可操作性 | 2 | 3 | 3 | 3 |
| D4 非交互与安全 | 3 | 3 | 3 | 3 |
| D5 输入友好 | 2 | 2 | 2 | 2 |
| D6 长任务与异步 | 2 | 2 | 3 | 3 |
| D7 上下文效率 | 1 | 2 | 3 | 3 |
| D8 确定性与隔离 | 1 | 2 | 2 | 3 |
| D9 认证 | 3 | 3 | 3 | 3 |
| D10 生态集成 | 3 | 3 | 3 | 3 |

验收方式：每个阶段发布前运行 A16 的 `agent-probe`，分数卡随版本附在 CHANGELOG 中；阶段 2 之后补一轮第三节所述的 L3 任务评测，以任务成功率和平均 token 作为最终验收指标。

## 七、待讨论

1. 权重是否合理。例如面向开发者的 CLI 可以提高 D7；面向运维的 CLI 可以提高 D4。
2. Agent 模式的触发方式：只用显式的 `BL_AGENT=1`，还是同时自动识别常见 Agent 的环境变量？自动识别更省事，但行为不够透明，且各 Agent 的环境变量名会变化。
3. 非 TTY 默认输出 JSON 属于破坏性变更，会影响用管道处理 text 输出的现有脚本。有了 A3 之后是否还需要改这个默认值。
4. `agent-probe` 是作为 bailian-cli 的内部工具，还是做成独立的开源项目，用来横向对比 gh、aws、gcloud、kubectl 等 CLI。
5. L3 任务集由谁维护，以及选哪几个 Agent 作为评测基线。
6. A4 的错误类型清单与 A13 中内容审核类错误码，需要与后端对齐命名。
7. A15 自动重试是否覆盖异步任务的提交接口：提交接口如果支持幂等键，可以安全重试；否则维持不重试。
