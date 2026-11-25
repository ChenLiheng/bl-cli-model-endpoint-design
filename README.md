# bl CLI 设计文档

bailian-cli（`bl`）相关的设计与评估文档。

| 文档 | 内容 |
| --- | --- |
| [AGENT.md](AGENT.md) | Agent 工作约束：本仓库只用于审查第三方仓库 PR 和书写计划；对第三方仓库只读，禁止提交代码、评论等任何操作；审查结论落地到一个文档 |
| [model-endpoint-proposal.md](model-endpoint-proposal.md) | 改造方案：现状、目标设计、迁移、风险、待讨论决策 |
| [model-endpoint-implementation.md](model-endpoint-implementation.md) | 落地实现方案：按 Step 的代码改动、多方案待取舍、测试与发布清单 |
| [agent-friendly-cli-evaluation.md](agent-friendly-cli-evaluation.md) | CLI 的 Agent 友好度评估标准（ACF Score）、bailian-cli 评估，以及分阶段的详细优化方案 |
| [bailian-cli-code-review.md](bailian-cli-code-review.md) | 代码审查：疑似 Bug（含复现步骤）、设计问题与优化点 |
| [freetier-autostop-guard-review.md](freetier-autostop-guard-review.md) | 分支审查：`feat/freetier-autostop-guard`（`30f32bbe`）的问题与解决方案 |
| [pr-313-freetier-recovery-review.md](pr-313-freetier-recovery-review.md) | PR 审查：`modelstudioai/cli` PR #313（`3b3cb92b`）的免费额度恢复 hint、确认门槛与 `--config` 钉定；含“停用失败仍显示成功”的归属排查与各问题的解决方案 |
| [pr-329-speech-voice-translate-review.md](pr-329-speech-voice-translate-review.md) | PR 审查：`modelstudioai/cli` PR #329（`bbe7edb9`）的 Qwen-MT 翻译命令、`speech generate` 与音色管理；含 dry-run 不一致、区域判断死代码、流式累计内容降级等问题的解决方案 |
| [pr-343-video-portrait-style-review.md](pr-343-video-portrait-style-review.md) | PR 审查：`modelstudioai/cli` PR #343（`4a61c5ae`）的 `video portrait`（VideoRetalk）与 `video style`（视频风格重绘）；与官方 API 文档逐项核对，建议移除万相 2.1 VACE 的 `video retouch`，含地域检查、非 ASCII URL、超时、边界情况、命令一致性与发布流程等问题的解决方案 |
| [pr-321-speech-timeout-review.md](pr-321-speech-timeout-review.md) | PR 审查：`modelstudioai/cli` PR #321（`3e5a7400`）让 `speech synthesize` 的 `--timeout` 覆盖音频下载（Issue #309）；含传输层默认 300s 超时仍生效、MCP 与 `--stream` 路径未覆盖、空 body 覆盖原文件、timeout 溢出等问题的实测与解决方案 |
| [embedding-rerank-implementation-plan.md](embedding-rerank-implementation-plan.md) | 实现计划：文本/多模态向量（`bl embedding text\|multimodal`）与重排序（`bl rerank`）；含接口速查、命令设计、模型能力表、文件改动清单、按步骤的实施顺序，以及 30+ 条坑位与待验证项 |
| [telemetry-device-id-persistence.md](telemetry-device-id-persistence.md) | 实施方案：遥测 `device_id` 改为本地持久化 UUID，含参考实现对比、完整代码与测试改动 |
| [pr-306-exit-after-run-review.md](pr-306-exit-after-run-review.md) | PR 审查：`modelstudioai/cli` PR #306（`38e4d91f`）成功路径主动退出（Issue #294 进程滞留约 50 秒）；含 Bun 下管道输出被截断的实测、更新检查延迟与 registry 配置问题、各问题的解决方案与分环境复现/测试方法 |
| [pr-348-flush-before-exit-review.md](pr-348-flush-before-exit-review.md) | PR 审查：`modelstudioai/cli` PR #348（`2db845f`）退出前刷新 stdout/stderr（`runToExit`）；含 Ctrl-C 后退出码被 `exit(0)` 覆盖、`handleError` 公开 API 破坏性变更、超时警告污染 JSON 输出等问题的解决方案 |
| [pr-349-telemetry-site-routing-review.md](pr-349-telemetry-site-routing-review.md) | PR 审查：`modelstudioai/cli` PR #349（`eeffcc5`）AEM 遥测按国内/国际站分流；含与当前 main（首装统计、`--biz-from`）的 5 个文件冲突的逐文件解决方案、`first_install` 站点决策，以及默认 baseUrl 被记成 `positive-domestic` 等问题的修复方案 |
| [telemetry-uid-lock-review.md](telemetry-uid-lock-review.md) | PR 审查：`modelstudioai/cli` PR #314 中 `telemetry-uid` 的 mkdir 锁问题，及无锁、带锁两种落地方案（含完整代码与压测结果） |

代码基线：`modelstudioai/cli` main `cadc301d`（v2.8.4）。状态：讨论稿。
