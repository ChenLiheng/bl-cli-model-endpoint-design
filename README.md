# bl CLI 设计文档

bailian-cli（`bl`）相关的设计与评估文档。

| 文档 | 内容 |
| --- | --- |
| [model-endpoint-proposal.md](model-endpoint-proposal.md) | 改造方案：现状、目标设计、迁移、风险、待讨论决策 |
| [model-endpoint-implementation.md](model-endpoint-implementation.md) | 落地实现方案：按 Step 的代码改动、多方案待取舍、测试与发布清单 |
| [agent-friendly-cli-evaluation.md](agent-friendly-cli-evaluation.md) | CLI 的 Agent 友好度评估标准（ACF Score），及 bailian-cli 评估与优化建议 |
| [bailian-cli-code-review.md](bailian-cli-code-review.md) | 代码审查：疑似 Bug（含复现步骤）、设计问题与优化点 |

代码基线：`modelstudioai/cli` main `cadc301d`（v2.8.4）。状态：讨论稿。
