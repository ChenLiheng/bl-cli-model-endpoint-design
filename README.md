# bl CLI 设计文档

bailian-cli（`bl`）相关的设计与评估文档。

| 文档 | 内容 |
| --- | --- |
| [model-endpoint-proposal.md](model-endpoint-proposal.md) | 改造方案：现状、目标设计、迁移、风险、待讨论决策 |
| [model-endpoint-implementation.md](model-endpoint-implementation.md) | 落地实现方案：按 Step 的代码改动、多方案待取舍、测试与发布清单 |
| [agent-friendly-cli-evaluation.md](agent-friendly-cli-evaluation.md) | CLI 的 Agent 友好度评估标准（ACF Score）、bailian-cli 评估，以及分阶段的详细优化方案 |
| [bailian-cli-code-review.md](bailian-cli-code-review.md) | 代码审查：疑似 Bug（含复现步骤）、设计问题与优化点 |
| [freetier-autostop-guard-review.md](freetier-autostop-guard-review.md) | 分支审查：`feat/freetier-autostop-guard`（`30f32bbe`）的问题与解决方案 |
| [pr-313-freetier-recovery-review.md](pr-313-freetier-recovery-review.md) | PR 审查：`modelstudioai/cli` PR #313（`3b3cb92b`）的免费额度恢复 hint、确认门槛与 `--config` 钉定；含“停用失败仍显示成功”的归属排查与各问题的解决方案 |
| [telemetry-device-id-persistence.md](telemetry-device-id-persistence.md) | 实施方案：遥测 `device_id` 改为本地持久化 UUID，含参考实现对比、完整代码与测试改动 |
| [telemetry-uid-lock-review.md](telemetry-uid-lock-review.md) | PR 审查：`modelstudioai/cli` PR #314 中 `telemetry-uid` 的 mkdir 锁问题，及无锁、带锁两种落地方案（含完整代码与压测结果） |

代码基线：`modelstudioai/cli` main `cadc301d`（v2.8.4）。状态：讨论稿。
