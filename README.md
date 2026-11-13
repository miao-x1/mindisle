# 心屿 MindIsle · 心理陪伴智能体

心屿是我独立开发的心理陪伴智能体项目，分用户端和管理员端：用户端提供共情对话与情绪支持，管理员端面向心理辅导老师，提供风险报告、个案跟进、知识库维护和预警台账。

做这个项目的初衷是想回答一个问题：心理支持类对话直接交给大模型，回复往往"听着舒服，但不可靠"。我的做法是把一次对话拆成意图理解、风险识别、知识检索、回复生成几个环节，用事件驱动的多 Agent runtime 编排，配独立的安全审查和一套可量化的评测，验证这条链路是否比单个大 Prompt 更稳、更安全。

技术选型上：FastAPI + 原生 HTML/CSS/JS（页面不复杂，没必要上框架），MySQL 存全量业务数据，Redis 存会话短期记忆，RAG 用 Chroma 向量 + BM25 混合召回，模型支持 Ollama 本地模型与 OpenAI-compatible API 两种接入，Docker Compose 一键启动。

## 快速开始

```bash
docker compose up -d --build
```

会启动三个容器：

- `mysql`：MySQL 8.4，容器内 `3306`，宿主机映射 `13306`
- `redis`：Redis 7，容器内 `6379`，宿主机映射 `16379`
- `app`：心屿 FastAPI 服务，宿主机端口 `8080`

打开 `http://127.0.0.1:8080`，默认账号：用户 `user / user123`，管理员 `admin / admin123`。

## Agent loop

每轮对话进入事件驱动多 Agent runtime。Coordinator 维护共享黑板和任务板，各 Agent 按能力认领任务、发布 artifact，安全审查通过后由 Coordinator 采纳，SSE 流式输出：

```text
TURN_STARTED
-> CoordinatorAgent 创建任务
-> UnderstandingAgent / SafetyAgent / ContextAgent / ResponseAgent 认领任务并发布 artifact
-> SafetyAgent 审查候选回复
-> CoordinatorAgent FINAL_ACCEPTED
-> SSE 流式输出
```

分工上我刻意让 SafetyAgent 独立于回复生成：它单独评估风险，必要时发布 `SAFETY_OVERRIDE`，候选回复也要过它审查，相当于给回复链路加了一道不受生成过程影响的安全门。

- `CoordinatorAgent`：任务板、预算、安全门槛、冲突仲裁和最终采纳
- `UnderstandingAgent`：判断 `CHAT / CONSULT / RISK`，发布 intent artifact
- `SafetyAgent`：独立风险评估 + 候选回复审查，可一票否决
- `ContextAgent`：聚合 Redis / MySQL 记忆、RAG 检索结果和 Skill 约束
- `ResponseAgent`：基于黑板 artifact 生成候选回复

runtime 入口在 `app/agents/event_driven_runtime.py`，共享返回类型在 `app/agents/result.py`。`AGENT_FRAMEWORK` 环境变量仍会读取，但当前只支持 `event_driven_multi_agent`，历史值会在状态接口标记为 fallback 并实际使用事件驱动 runtime。

## 目录结构

```text
app/
├── agents/          # 事件驱动多 Agent runtime
├── api/             # FastAPI 路由
├── core/            # 配置、数据库、安全、启动初始化
├── knowledge/       # 内置心理健康知识库
├── mcp_tools/       # MCP 工具服务
├── models/          # SQLAlchemy 实体
├── rag_eval/        # RAG 评测脚本和数据集
├── schemas/         # Pydantic DTO
├── services/        # AI、聊天、知识库、评估、报告、工具服务
└── static/          # 原生前端页面

models/mindisle-qwen2.5-7b-ft/   # Ollama 本地微调模型定义（Modelfile）
skills/                          # 内置标准 Skill（SKILL.md）
scripts/                         # 开发、模型创建与打包脚本
```

## RAG 混合召回

检索时同时取 Chroma 向量候选和 BM25 关键词候选，按权重融合后进入本地 reranker。Embedding 服务不可用或缺少 API key 时自动降级为本地 BM25 + `hybrid_score` reranker，聊天链路不会中断：

```env
OPENAI_API_KEY=你的_API_Key
OPENAI_EMBEDDING_MODEL=text-embedding-3-small
KNOWLEDGE_VECTOR_ENABLED=true
KNOWLEDGE_VECTOR_REQUIRED=false
KNOWLEDGE_CANDIDATE_K=16
KNOWLEDGE_HYBRID_VECTOR_WEIGHT=0.65
KNOWLEDGE_HYBRID_BM25_WEIGHT=0.35
KNOWLEDGE_RERANK_ENABLED=true
CHROMA_PERSIST_DIR=data/chroma
CHROMA_SNAPSHOT_DIR=data/chroma-snapshots
```

管理员接口：

```bash
curl -u admin:admin123 http://127.0.0.1:8080/api/admin/knowledge/status
curl -u admin:admin123 -X POST http://127.0.0.1:8080/api/admin/knowledge/rebuild-vector
curl -u admin:admin123 -X POST http://127.0.0.1:8080/api/admin/knowledge/backup
```

内置知识库在 `app/knowledge/*.md`，启动时自动切块入库，覆盖情绪低落、焦虑恐慌、睡眠作息、考试季、人际关系、咨询转介等主题；也可以在管理员页面直接上传 PDF / Markdown / txt 追加，新分块会同步写入 MySQL 和 Chroma，已有分块在首次向量检索时自动补建索引。

## 工具队列与预警

高风险对话会触发心理报告，之后的 Excel 台账写入、个案创建、邮件预警都不阻塞聊天流，而是写入 `tool_jobs` 队列异步执行：

```text
EXCEL_REPORT
CASE_CREATE -> ALERT_SEND
```

Excel 写入用进程内锁串行化，个案创建保持幂等，预警发送用独立线程池并支持每分钟限流；失败任务按延迟重试，超过 `TOOL_QUEUE_MAX_ATTEMPTS` 后进入 `dead_letter_records`。

```env
TOOL_QUEUE_ENABLED=true
TOOL_QUEUE_EXCEL_WORKERS=1
TOOL_QUEUE_EMAIL_WORKERS=2
ALERT_EMAIL_RATE_LIMIT_PER_MINUTE=30
ALERT_EMAIL_DELIVERY_MODE=log
```

`ALERT_EMAIL_DELIVERY_MODE=log` 适合本地演示，生产环境改为 `smtp` 并配置：

```env
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USERNAME=your-account@example.com
SMTP_PASSWORD=your-smtp-password
SMTP_USE_TLS=true
SMTP_USE_SSL=false
ALERT_EMAIL_FROM=your-account@example.com
ALERT_EMAIL_TO=counselor@example.com,admin@example.com
ALERT_EMAIL_SUBJECT_PREFIX=[心屿高风险预警]
```

未配置 SMTP 或收件人不会中断聊天，`alert_records` 会写入 `FAILED` 记录，提示缺少的配置项。

## 模型接入

通过 `AI_PROVIDER` 在两种方式间切换。

### Ollama 本地模型

默认预留本地微调模型名 `mindisle-qwen2.5-7b-ft:latest`，GGUF 权重放到 `models/mindisle-qwen2.5-7b-ft/`：

```bash
UPSTREAM_GGUF=/path/to/mindisle-qwen2.5-7b-ft-q4_k_m.gguf ./scripts/create-finetuned-model.sh
./scripts/start-ollama.sh
AI_PROVIDER=ollama ./scripts/run-dev.sh
```

`/api/agent/status` 返回的 `finetunedModel.ggufExists`、`finetunedModel.modelfileExists` 可以确认模型资产是否就绪。

### OpenAI-compatible API

```bash
AI_PROVIDER=openai \
OPENAI_API_KEY=你的_API_Key \
OPENAI_MODEL=gpt-4o-mini \
OPENAI_EMBEDDING_MODEL=text-embedding-3-small \
uvicorn app.main:app --host 127.0.0.1 --port 8080
```

向量检索复用同一个 `OPENAI_API_KEY` 调 embeddings，兼容任意 OpenAI-compatible 服务（例如通义百炼、DeepSeek）。

### 调用示例

用户流式聊天：

```bash
curl -N -u user:user123 \
  -H 'Content-Type: application/json' \
  -d '{"message":"我最近很焦虑，晚上总是睡不着"}' \
  http://127.0.0.1:8080/api/chat/stream
```

高风险示例（如"我不想活了，感觉撑不下去了"）会触发心理报告、风险个案创建和预警工具计划，用户端感知不到这些后台处理。

管理员查看报告：

```bash
curl -u admin:admin123 http://127.0.0.1:8080/api/admin/reports
```

## 评测体系

配了三套评测，全部输出可复现的报告文件。

### RAG 检索评测

```bash
AI_PROVIDER=mock python -m app.rag_eval.runner
```

输出 `target/rag-eval-report.json`，指标包含 Recall@K、MRR、NDCG、HitRate。

### LLM-as-Judge 回答质量评测

内置 30 条用例，覆盖普通问答、心理支持、高风险危机和多轮上下文。Judge 按 1-5 分评价 `safety`、`relevance`、`helpfulness`、`correctnessGroundedness`、`empathyTone`、`clarity` 六个维度，Judge 的 API Key 只从环境变量读取，不写入报告：

```bash
JUDGE_API_KEY=你的评测密钥 \
JUDGE_MODEL=gpt-4o-mini \
python -m app.answer_eval.runner --candidate-model mindisle-qwen2.5-7b-ft:latest --progress
```

输出 `target/answer-quality-eval-report.json`。评测固定使用数据集里的预期意图、风险和参考信息，只测回复生成质量，避免把路由和 RAG 的错误混进模型分数。

### 多 Agent 对照盲评

同一批数据上跑实验组（完整多 Agent 链路）和基线组（同模型、同温度、同知识库的单 Agent 大 Prompt），Judge 隐藏架构身份、以确定性伪随机平衡 A/B 位置成对盲评：

```bash
JUDGE_API_KEY=你的评测密钥 \
JUDGE_MODEL=gpt-4o-mini \
python -m app.agent_eval.runner --candidate-model mindisle-qwen2.5-7b-ft:latest --progress
```

输出 `target/multi-agent-comparison-report.json`，包含胜负、分维度均分、路由/风险准确率、延迟和排除平局后的二项符号检验。只有有效样本不少于 20、多 Agent 胜场更多、`p < 0.05`、总体分差为正且危机场景安全分不下降，结论才标记 `SUPPORTED`；否则如实标记样本不足，避免预设"多 Agent 一定更好"。

首次接入 Judge 可以用 `--limit 2` 做低成本连通性检查。

## 数据库配置

MySQL 保存完整聊天记录和业务数据，Redis 只保存每个会话最近 `REDIS_MEMORY_MAX_MESSAGES` 条短期上下文，TTL 由 `REDIS_MEMORY_TTL_SECONDS` 控制：

```sql
CREATE DATABASE mindisle DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'mindisle'@'%' IDENTIFIED BY 'mindisle';
GRANT ALL PRIVILEGES ON mindisle.* TO 'mindisle'@'%';
FLUSH PRIVILEGES;
```

```env
DATABASE_URL=mysql+pymysql://mindisle:mindisle@127.0.0.1:3306/mindisle?charset=utf8mb4
REDIS_URL=redis://127.0.0.1:6379/0
REDIS_MEMORY_TTL_SECONDS=86400
REDIS_MEMORY_MAX_MESSAGES=40
```

## 开发与测试

```bash
pip install -r requirements.txt
python -m unittest discover -s tests          # 基础回归（unittest，不依赖 pytest）
python -m app.harness.runner                  # 工程 harness
python -m app.mcp_tools.server                # MCP 工具服务（建议 Python 3.10/3.11）
```

工程 harness 用 mock AI、临时 SQLite 和内存短期记忆验证六条核心链路：风险安全、Agent 路由、标准 Skill 加载、RAG 评测、API（认证/SSE/知识库）、工具队列（依赖/幂等/限流/死信），报告输出到 `target/harness/`。

MCP 暴露的工具：`mindisle_excel_report`、`mindisle_case_create`、`mindisle_alert_send`、`mindisle_alert_ack`、`mindisle_case_note_add`、`mindisle_alert_notify`。

内置标准 Skill 在 `skills/*/SKILL.md`，由 `MindIsleSkillRegistry` 加载，覆盖基础共情与表达边界、高风险安全计划、焦虑 grounding、睡眠建议、现实压力拆解、转介资源和个案交接摘要模板。
