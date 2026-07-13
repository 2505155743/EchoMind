# EchoMind — 企业级多 Agent 智能客服 AI 系统

EchoMind 是一个基于大语言模型（LLM）的企业级智能客服 AI 系统。它采用多 Agent 编排架构，通过三层路由决策、三路融合意图识别、RAG 知识库检索、三级记忆管理和在线性能监控反馈闭环，为企业提供可观测、可评测、可扩展的智能客服解决方案。

## 项目目的

在真实的客服场景中，单一大模型往往面临以下挑战：

- 意图识别不精准 — 用户意图模糊时，单一模型难以准确分类
- 知识更新滞后 — 模型知识有截止日期，无法实时反映产品 API、退换货政策等业务变更
- 上下文爆炸 — 长对话中早期关键信息被遗忘，影响回复质量
- 性能不可控 — 缺乏对 Agent 在线表现的量化评估和动态调节机制
- 评测困难 — 客服场景的回复质量高度主观，传统指标难以衡量

EchoMind 的目标是系统性地解决这些问题，构建一个可观测（Observable）、可评测（Evaluable）、可动态调节（Adaptive）的企业级智能客服平台。

---

## 技术栈

| 层级 | 技术选型 |
|------|---------|
| 编程语言 | Python 3.12 |
| Web 框架 | FastAPI + Uvicorn |
| LLM SDK | Anthropic Python SDK（兼容 DeepSeek 等第三方 API） |
| 向量数据库 | ChromaDB 0.5.23（内置 all-MiniLM-L6-v2 Embedding 模型） |
| 工作记忆缓存 | Redis 7（支持 AOF 持久化） |
| 监控系统 | Prometheus + 自定义 PerformanceMonitor |
| 反向代理 | Nginx（限流、安全头、负载均衡） |
| 容器化 | Docker + Docker Compose（多阶段构建） |
| 数据序列化 | Pydantic v2 |
| HTTP 客户端 | httpx |

---

## 模块功能

### 1. 多 Agent 编排器

系统的核心调度大脑，实现三层路由决策：

- 第一层 — 意图路由：根据 IntentCategory 直接映射到对应的 Agent（TECHNICAL → TechnicalAgent、BILLING → BillingAgent 等）
- 第二层 — 性能路由：同类多实例时，综合成功率（70%）+ 延迟（30%）- 监控惩罚项，选择最优 Agent
- 第三层 — 降级路由：专属 Agent 不可用或失败时，自动降级到 GeneralAgent 兜底

内置四种 Agent 类型：

| Agent | 职责 |
|-------|------|
| GeneralAgent | 通用客服接待 |
| TechnicalAgent | 技术支持 |
| BillingAgent | 账单与退款服务 |
| EscalationAgent | 人工升级转接（占位） |

亮点机制：
- 并行协作：复杂问题（如同时涉及技术和账单）可并行派发给多个 Agent，结果合并后返回
- 升级检测：Agent 置信度低于阈值、紧急度为 CRITICAL、或内容含"转人工"关键词时自动触发升级
- Monitor 闭环反馈：接收 PerformanceMonitor 的在线表现数据，动态调整路由权重

### 2. 三路融合意图识别

采用三路融合加权策略，兼顾精度与速度：

| 策略 | 权重 | 说明 |
|------|------|------|
| LLM 语义理解 | 70%（有 Embedding）/ 85%（纯 LLM） | Few-shot + 历史上下文，Anthropic API |
| Embedding 向量相似度 | 20% | 模板文本的 n-gram 哈希向量，余弦相似度匹配 |
| 关键词模式匹配 | 10% / 15% | 零延迟的规则兜底 |

支持 10 种意图分类：QUERY、COMPLAINT、REQUEST、GREETING、ESCALATION、TECHNICAL、BILLING、ACCOUNT、FEEDBACK、OTHER

额外能力：
- 四级紧急度判定（LOW / MEDIUM / HIGH / CRITICAL）
- LLM 实体提取（订单号、产品、日期、金额、错误码等）
- 在线学习：通过 learn() 方法可将人工纠正样本实时加入匹配模板
- LRU 缓存（最多 1000 条），减少重复调用

### 3. RAG 知识库

基于 ChromaDB 的语义检索实现：

- 自动连接策略：优先连接独立 ChromaDB 服务，不可用时自动降级到本地 PersistentClient
- 文档分块：按 500 字自动切片（以句号分割保持语义完整性）
- 自动向量化：ChromaDB 内置 all-MiniLM-L6-v2 模型，无需外部 Embedding API
- 默认知识库：内置 6 篇文档（退款政策、订单查询、账户安全、技术故障排查、会员积分、配送说明），开箱即用

### 4. MCP 工具调用框架

完整的工具调用优化链路：

1. 查询改写（Query Rewriting）— LLM 将原始查询扩写成多个角度的子查询，解决"召回不全"
2. 结果重排（Reranking）— LLM 对召回结果按相关性重新打分排序，解决"召回不好"
3. 熔断器（Circuit Breaker）— 三态（CLOSED/OPEN/HALF-OPEN），连续失败后自动断开保护
4. TTL 缓存 — 相同参数直接返回缓存结果，减少重复调用
5. 参数校验 — 根据 JSON Schema 严格校验参数类型
6. 降级策略（Fallback）— 工具不可用时返回有意义的降级结果

当前已注册工具：knowledge_search — 基于 ChromaDB 的 RAG 知识库检索（300 秒 TTL 缓存、支持重排、有降级处理）

### 5. 三级记忆管理器

模拟人类记忆机制的三级架构：

| 记忆层级 | 存储介质 | 功能 | TTL/持久化 |
|---------|---------|------|-----------|
| 工作记忆 | Redis | 当前会话最近 20 条消息 | 24 小时自动过期 |
| 情景记忆 | ChromaDB | 跨会话的历史对话片段 | 持久化（语义检索） |
| 用户画像 | ChromaDB | 从对话中提炼的用户偏好和实体 | 持久化 |

关键设计：
- 工作记忆超过 15 条时自动触发压缩：LLM 生成摘要 + 保留最近 5 条 + 旧消息存入情景记忆
- 上下文构建时三级记忆融合，按重要性 + 时效性排序
- 所有 Embedding 通过 ChromaDB 内置模型生成

### 6. 性能监控器

在线表现监控与闭环反馈：

- 实时采集：每隔 N 秒从 Orchestrator 和 ToolManager 拉取统计
- Z-score 异常检测：滑动窗口算法自动发现指标突变
- 路由反馈：将成功率和延迟转化为 0-0.9 的 penalty，写回 Orchestrator 动态调整路由权重
- 阈值告警：Agent 成功率 < 90%、工具成功率 < 95%、延迟 > 3s 时触发告警
- Prometheus 指标：可选暴露 agent_success_rate、agent_latency_ms 等指标
- Webhook 通知：告警超阈值时可发送到外部 URL

### 7. 端到端评测框架

四维自动评测体系：

1. 意图识别评测 — 计算准确率 + Macro-F1 + 每类 Precision/Recall/F1
2. LLM-as-Judge 对话质量评分 — 从相关性、准确性、完整性、有用性四维度打分
3. 端到端对话评测 — 支持单轮和多轮对话的全流程评测
4. 回归检测 — 与历史基线对比，退化超过 5% 自动标记

内置 8 个意图测试用例和 5 个对话测试用例，开箱即用。

### 8. Skill 热加载引擎

轻量级业务能力注入系统：

- 支持 Markdown（含 Front Matter）、JSON、TXT 格式
- 支持关键词触发（用户消息命中后才注入对应 Skill）
- 支持 Agent 绑定（可指定 Skill 只对特定 Agent 生效）
- 运行时热加载：POST /skills/reload 无需重启服务
- 自动截断：按 max_prompt_chars 限制总长度，避免挤占主对话上下文

内置三个 Skill：

| Skill | 关键词触发 | 目标 Agent |
|-------|-----------|-----------|
| 通用客服接待规范 | 无（全匹配） | GeneralAgent |
| 技术支持处理规范 | 技术类关键词 | TechnicalAgent |
| 账单退款处理规范 | 账单类关键词 | BillingAgent |

---

## API 接口

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /health | 健康检查，返回 Agent 运行状态 |
| POST | /chat | 主对话接口，完整客服对话流程 |
| GET | /skills | 查看已加载的 Skills |
| POST | /skills/reload | 热重载 Skills |
| GET | /monitor | 实时监控摘要 |
| GET | /metrics | Prometheus 指标 |
| POST | /search | 知识库检索演示 |
| POST | /knowledge/add | 批量导入知识库文档 |
| POST | /knowledge/upload | 上传文件导入知识库 |
| GET | /knowledge/stats | 知识库统计 |
| POST | /eval/run | 运行端到端评测 |

---

## 快速开始

### 1. 配置环境变量

cp .env.example .env

编辑 .env，填入你的 API Key。

### 2. 启动服务

docker compose up -d

### 3. 验证

curl http://localhost:8000/health

浏览器打开 http://localhost:8000/docs 查看 Swagger UI。

---

## 部署架构

Nginx (80) → EchoMind (8000) → Redis (6379, 工作记忆缓存)
                              → ChromaDB (8001, 情景记忆 + 知识库)
                              → Prometheus (9090, 监控指标采集)

---

## 项目结构

EchoMind/
  agents/agent_orchestrator.py    # 多 Agent 编排器
  api/main.py                     # FastAPI 应用入口
  config/nginx/nginx.conf         # Nginx 配置
  config/prometheus.yml           # Prometheus 配置
  core/intent_recognizer.py       # 意图识别器
  core/skill_loader.py            # Skill 热加载引擎
  mcp/knowledge_base.py           # RAG 知识库
  mcp/tool_manager.py             # MCP 工具调用框架
  memory/conversation_memory.py   # 三级记忆管理器
  monitor/performance_monitor.py  # 性能监控器
  evaluation/evaluator.py         # 端到端评测框架
  skills/                         # 业务技能定义
  data/                           # 数据文件
  Dockerfile                      # Docker 多阶段构建
  docker-compose.yml              # 服务编排
  requirements.txt                # Python 依赖

