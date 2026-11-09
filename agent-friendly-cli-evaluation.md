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
    - 错误对象只有数字码，没有稳定的字符串错误类型。
- 有 6 项 P0 优化，改动都集中在 runtime 层，不涉及业务命令，见文末「优化建议」。

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
| F4 | 错误对象只有数字码 | `{"error":{"code":3,"message":"No API key found.","hint":"Set DASHSCOPE_API_KEY, …\nbl auth login …"}}` | 同一个退出码 3 下的不同原因（缺 key、key 无效、key 过期）无法区分；hint 是多行文本，修复命令要靠 Agent 自己抽取 |
| F5 | 未知参数没有建议 | `bl text chat --mesage hi` 返回 `Unknown flag "--mesage". Run with --help…` | 命令拼错时有建议，参数拼错却没有，Agent 只能再读一遍 help |
| F6 | Schema 没有结构化的默认值和输出结构 | flag 只有 `description / type / valueHint / required / choices` 五个键；默认值只写在描述里，如「(default: 4096)」；`constraints.crossFlagValidation: "not-exported"` | Agent 无法可靠地知道默认行为，也无法预知结果有哪些字段 |
| F7 | 运行时自动升级 | `versionCheckStage` 在大版本差距时调用 `performAutoUpdate`（`packages/runtime/src/middleware.ts:299-330`），只有设置 `NO_UPDATE_NOTIFIER=1` 才关闭 | 同一个 Agent 任务的前后两次调用可能跑在不同大版本上，结果不可复现 |
| F8 | 模型地址解析不一致 | `base_url` 与 `workspace_id` 两套机制，不同命令取舍不同（详见同仓库的 `model-endpoint-proposal.md`） | 同一份配置下，chat 和知识库可能打到不同的空间，Agent 很难排查 |
| F9 | `--dry-run` 不显示目标地址 | `bl text chat --message hi --dry-run --output json` 只输出 `request` 请求体，没有方法和 URL | Agent 无法在执行前确认请求会发往哪个地址、哪个空间 |
| F10 | 部分清除类操作没有门禁 | `bl auth logout` 不带 `--yes` 就直接执行，且默认作用域是 `all` | 本地凭证会被误清。影响较小，但与高风险门禁的设计不一致 |
| F11 | Skill 总量偏大 | 9 个 SKILL.md 共 127,803 字节，reference 目录约 1.3 MB | 按需加载时问题不大；但 hub skill `bailian-cli` 本身有 22 KB，加载一次的成本偏高 |

未实测的部分：真实 API 调用（没有凭证）、构建产物的启动耗时（这次是用 tsx 源码运行的，不具代表性），以及 L3 Agent 任务评测。D5、D6 的得分是根据 Schema 和代码给出的，没有端到端验证。

## 五、优化建议

### P0：改动集中在 runtime，收益最大

| # | 建议 | 解决 | 改动位置 |
| --- | --- | --- | --- |
| O1 | 新增 `--introspect=index`（或 `bl introspect --index`），只输出命令路径、auth 类型和一句话描述；根级全量 introspect 在非 TTY 下默认改为输出 index，并提示用 `<path> --introspect` 获取详情 | F1 | `runtime/src/registry.ts` |
| O2 | introspect 与 help 只输出一种语言（跟随 `language` 配置，或新增 `--lang`），默认 en-US | F1、F2 | 同上 |
| O3 | 显式 `--help` 写 stdout；非 TTY 下不打印横幅 | F2 | `runtime/src/output/banner.ts`、`registry.ts` |
| O4 | 增加 Agent 模式：检测到 `BL_AGENT=1`（或常见 Agent 环境变量）时，默认 `--output json`、关闭横幅和颜色、关闭自动升级 | F3、F7 | `core/src/output/formatter.ts`、`runtime/src/middleware.ts` |
| O5 | 错误对象扩展为 `{ code, type, message, hint, fix_commands[], retryable, retry_after_ms?, request_id? }`。`type` 用稳定的大写字符串；`code` 保持为退出码，保证兼容 | F4 | `core/src/errors/*`、`runtime/src/error-handler.ts` |
| O6 | 未知参数基于编辑距离给出建议，与未知命令的处理一致 | F5 | `runtime/src/args.ts` |

### P1

| # | 建议 | 解决 |
| --- | --- | --- |
| O7 | flag 定义增加结构化的 `default`；introspect 输出 `output` 结构（至少列出顶层字段与类型）；导出跨参数约束 | F6 |
| O8 | 自动升级只在 TTY 且交互式运行时生效；非 TTY 下只发一条 `UPDATE_AVAILABLE` warning | F7 |
| O9 | `--dry-run` 统一输出 `{ method, url, headers(已脱敏), body }` | F9 |
| O10 | 按 `model-endpoint-*.md` 方案统一模型地址解析 | F8 |
| O11 | 增加通用的 `--fields a,b.c` 投影，或 `--jq` 表达式 | D2.5 |

### P2

| # | 建议 | 解决 |
| --- | --- | --- |
| O12 | `auth logout` 的 `all` 作用域纳入风险门禁，或默认作用域改为 `api-key` | F10 |
| O13 | 精简 hub skill，只保留路由表，细节交给领域 skill | F11 |
| O14 | 在仓库内落地 L1 + L2 探测脚本（如 `tools/agent-probe/`），接入 CI；分数下降时阻断合入 | 防止回退 |

预计做完 P0 后，D1、D2、D3、D7、D8 各提升约 1 分，总分到 85 分左右（优秀）。

## 六、待讨论

1. 权重是否合理。例如面向开发者的 CLI 可以提高 D7；面向运维的 CLI 可以提高 D4。
2. Agent 模式的触发方式：显式环境变量，还是同时自动识别常见 Agent 的环境变量？自动识别更省事，但行为不够透明。
3. 非 TTY 默认输出 JSON 属于破坏性变更，会影响用管道处理 text 输出的现有脚本，是否放到大版本再改。
4. `agent-cli-probe` 是作为 bailian-cli 的内部工具，还是做成独立的开源项目，用来横向对比 gh、aws、gcloud、kubectl 等 CLI。
5. L3 任务集由谁维护，以及选哪几个 Agent 作为评测基线。
