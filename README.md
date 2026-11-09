# bl CLI 模型端点统一改造

bl CLI 将模型服务地址从 dashscope 公共域名统一为 `https://{workspace_id}.{region}.maas.aliyuncs.com` 的设计文档。

| 文档 | 内容 |
| --- | --- |
| [model-endpoint-proposal.md](model-endpoint-proposal.md) | 改造方案：现状、目标设计、迁移、风险、待讨论决策 |
| [model-endpoint-implementation.md](model-endpoint-implementation.md) | 落地实现方案：按 Step 的代码改动、多方案待取舍、测试与发布清单 |

代码基线：`modelstudioai/cli` main `cadc301d`（v2.8.4）。状态：讨论稿。
