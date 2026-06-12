# Rag-agent

Rag-agent 是一个面向企业的 RAG 智能体平台，基于 Java + React 构建，覆盖从文档解析、向量检索、意图路由到模型网关的完整链路。支持多种文档格式、可配置分块策略、意图树驱动的定向检索、MCP 工具调用、系统指令应答，以及模型健康检查与故障切换。

---

## 核心特性

### 文档解析与分块引擎

基于 Apache Tika 实现多格式文档解析（PDF、Word、Markdown、HTML 等），输出纯文本后进入可配置分块管线。

提供两种分块策略，可按文档类型灵活切换：

| 策略 | 目标 | 核心机制 |
|------|------|----------|
| 固定窗口重叠 | 通用文本 | 512 字符窗口 + 128 字符重叠，在句末标点和换行处边界对齐，保留上下文连续性 |
| 结构感知 | Markdown 等结构化文档 | 按标题、代码块、原子链接、段落分段，在 min(600) / target(1400) / max(1800) 字符预算内打包，不切断语义单元 |

两种策略均对 URL 断行、中文软换行等常见排版异常做归一化处理，避免中文词被拆散或 URL 被切烂，保证 chunk 的语义完整性。

### RAG 检索链路优化

查询进入后的完整管线：

```
术语归一化 → 查询改写 → 多问题拆分 → 意图分类 → 多通道并行检索 → 去重 → Rerank → 生成
```

关键优化点：

- **术语归一化**：启动时从数据库加载同义词/别名映射到内存，对用户口语化表达做标准化替换，消除词汇级歧义
- **查询改写**：LLM 去除礼貌用语和冗余修饰，保留专有名词、时间范围、角色身份等关键约束
- **多问题拆分**：含多个问句的复杂问题自动拆分，子问题并行查询各自的知识库，避免「混合向量」导致召回偏向
- **多通道并行**：意图定向检索（精确命中）与全局向量检索（低置信度兜底）并发执行，互补覆盖
- **Rerank 精排**：Cross-Encoder 对粗召回候选逐对打分重排序，解决双塔 embedding「相似度高 ≠ 答案正确」的问题

针对典型企业知识库场景的评估：Recall@5 提升约 15%，TopK 截断量以内命中率显著改善。

### 意图树与澄清反问

构建三层意图树（Domain → Category → Topic），覆盖三种意图类型：

| 意图类型 | 说明 | 行为 |
|----------|------|------|
| KB 知识库 | 企业文档检索 | 按意图节点关联的 Collection 定向检索 |
| MCP 工具 | 实时数据查询 | LLM 提取参数后调用外部工具/API |
| SYSTEM 系统 | 问候、自我介绍 | 直接走预设 prompt 模板回答 |

意图分类采用 LLM 全量打分模式：将所有叶子节点的描述和示例问题一次性送入模型，按 score 降序排序，根据置信度阈值决定路由策略：

- **高置信度**：命中意图节点 → 定向到对应 Collection / MCP 工具，检索效率最高
- **低置信度**：触发澄清反问（如「您想了解 OA 系统还是保险系统的数据安全？」），由用户选择后精准路由
- **低于最低阈值或无匹配**：启用全局检索通道兜底，扫全库召回

意图识别 Top-1 准确率达 90%+，低置信度反问机制有效避免了「答非所问」的体验问题。

### 模型网关与高可用

LLM 调用是 RAG 链路中延迟最高、稳定性最差的环节。项目内置模型路由层，通过 YAML 配置即可管理多供应商候选池：

```
硅基流动 (GLM-4.7) ──→ 百炼 (qwen-plus) ──→ 本地 Ollama (qwen3:8b)
     主力                   降级                   兜底
```

核心机制：

- **健康检查**：每个模型候选维护成功/失败计数，连续失败 N 次（默认 2）自动熔断，熔断窗口（默认 30s）结束后恢复探测
- **超时重试**：单次调用超时后自动切换到下一个候选，上层业务无感知
- **故障切换**：候选池按优先级排序，高优挂了自动降级到备选，全链路失败才抛异常
- **模型能力分层**：Chat、Embedding、Rerank 三类能力独立配置候选池，互不影响

P95 接口延迟控制在 10s 以内，单供应商故障不影响整体可用性。

---

## 系统架构

```
┌──────────────────────────────────────────────────────────┐
│                    前端 React + Vite                      │
│           管理后台 (知识库/意图/追踪/系统设置)              │
│           用户端 (对话界面 + SSE 流式输出)                 │
└──────────────────────┬───────────────────────────────────┘
                       │ /api/ragent
┌──────────────────────┴───────────────────────────────────┐
│                   bootstrap 服务层                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────────┐ │
│  │ Rewrite  │→│ Intent   │→│ Retrieve │→│ Generation  │ │
│  │ 改写拆分 │ │ 意图分类 │ │ 多通道检索│ │ Prompt组装  │ │
│  └──────────┘ └──────────┘ └──────────┘ └─────────────┘ │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────────┐ │
│  │ Ingest   │ │ Knowledge│ │  Trace   │ │ Dashboard   │ │
│  │ 文档入库 │ │ 知识库CRUD│ │ 链路追踪 │ │ 运营看板    │ │
│  └──────────┘ └──────────┘ └──────────┘ └─────────────┘ │
└──────────────────────┬───────────────────────────────────┘
                       │
┌──────────────────────┴───────────────────────────────────┐
│                  infra-ai 模型网关                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────────┐ │
│  │  Chat    │ │Embedding │ │ Rerank   │ │Model Health │  │
│  │  路由降级 │ │ 多供应商 │ │ 精排     │ │ 熔断恢复    │ │
│  └──────────┘ └──────────┘ └──────────┘ └─────────────┘ │
└──────────────────────┬───────────────────────────────────┘
                       │
┌──────────────────────┴───────────────────────────────────┐
│                    基础设施                               │
│  PostgreSQL + pgvector / Milvus │ Redis │ RocketMQ       │
│  MinIO/S3 (文件存储)  │  MCP Server (工具调用)           │
└──────────────────────────────────────────────────────────┘
```

---

## 项目结构

```
Rag-agent/
├── bootstrap/         # 业务实现层
│   └── src/main/java/com/.../
│       ├── admin/     # Dashboard 运营看板
│       ├── core/      # 分块引擎 + 文档解析
│       ├── framework/ # 统一返回体、异常处理、Trace 上下文
│       ├── ingestion/ # 文档入库管线 (Pipeline + Chunk)
│       ├── knowledge/ # 知识库 & 文档 CRUD
│       ├── rag/       # RAG 核心 — 改写、意图、检索、生成
│       ├── resources/ # 提示词模板 (.st)
│       └── user/      # 用户认证 (SaToken)
├── framework/         # 共享基础设施
│   └── src/main/java/com/.../framework/
│       ├── convention/  # Result、RetrievedChunk
│       ├── errorcode/   # IErrorCode + BaseErrorCode (A/B/C 三级)
│       ├── exception/   # Client/Service/Remote 三类异常
│       ├── trace/       # @RagTraceRoot / @RagTraceNode 注解
│       └── web/         # GlobalExceptionHandler
├── infra-ai/          # AI 模型适配层
│   └── src/main/java/com/.../infra/
│       ├── chat/      # LLM 对话客户端 (Ollama/SiliconFlow/BaiLian)
│       ├── embedding/ # 向量化客户端
│       ├── rerank/    # 重排序客户端
│       └── model/     # 路由选择器 + 健康检查 + 故障转移
├── mcp-server/        # MCP 工具服务
├── frontend/          # React 18 + TypeScript + Vite
│   └── src/
│       ├── pages/        # chat / admin (Dashboard/知识库/意图/追踪)
│       ├── components/   # 共享 UI 组件
│       ├── services/     # API 调用封装
│       └── stores/       # Zustand 状态管理
└── resources/         # SQL 脚本、Docker Compose 等
```

---

## 技术栈

| 层级 | 技术 |
|------|------|
| 后端框架 | Java 17, Spring Boot 3.5.x |
| 构建工具 | Maven 多模块 |
| ORM | MyBatis-Plus |
| 向量库 | pgvector (HSW 索引) / Milvus |
| 数据库 | PostgreSQL |
| 缓存 | Redis (Redisson) |
| 消息队列 | RocketMQ |
| 对象存储 | MinIO / S3 |
| 认证 | SaToken |
| 前端 | React 18 + TypeScript + Vite + Tailwind CSS |
| 文档解析 | Apache Tika |

---

## 本地运行

### 前置依赖

- PostgreSQL 15+ （需安装 pgvector 扩展）
- Redis 6+
- RocketMQ 或跳过消息队列（影响文档异步分块）
- Ollama（可选，本地模型运行）

### 后端

```bash
# 编译全部模块
mvn clean install

# 启动 bootstrap 服务
mvn -pl bootstrap -am spring-boot:run
```

服务默认运行在 `http://localhost:9090/api/ragent`。

### 前端

```bash
cd frontend
npm install
npm run dev
```

### 向量库切换

在 `application.yaml` 中配置：

```yaml
rag:
  vector:
    type: pg    # pg (pgvector) 或 milvus
```

### AI 模型配置

在 `application.yaml` 的 `ai` 段配置供应商和候选模型优先级，支持 Chat / Embedding / Rerank 三类能力独立配置。详见 `application.yaml` 注释。

---

## License

Apache License 2.0
