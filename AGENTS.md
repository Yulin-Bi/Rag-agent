# AGENTS.md

这是一个 Java + React 多模块仓库。处理任务时优先 **最小改动** 原则，将修改限制在真正持有该行为的模块内，交付前按验证命令检查。

## 项目结构与代码放置

| 模块 | 职责 |
|------|------|
| `bootstrap/` | 业务接口、编排逻辑、知识库流程、上传、聊天和 RAG 编排 |
| `framework/` | 共享基础设施、通用工具、跨模块通用能力 |
| `infra-ai/` | AI 模型供应商逻辑、路由、集成适配器 |
| `mcp-server/` | 独立的 MCP 服务 |
| `frontend/src/` | React + Vite 管理端和聊天界面的 UI 组件、页面、状态和服务 |

跨模块修改时，先改底层持有行为的模块，再改调用方。新增检索或后处理行为时，先找现有扩展点，再考虑新增流水线。

## 开发规则

- **后端**：Java 17 + Spring Boot 3.5.x，保留 Spotless 强制执行的 Apache 2.0 许可证头。
- **前端**：React 18 + TypeScript + Vite + ESLint + Prettier。
- **设计**：优先沿用仓库已有的策略模式、注册表、工厂模式和责任链，不要额外引入新模式或抽象，除非现有边界确实需要。
- **风格**：与上下文保持一致，避免无关重构。
- **测试**：行为变化时补充或更新测试。
- **注释**：保持简短，只在代码意图不直观时补充。

## 验证命令

优先使用能覆盖当前改动范围的最小命令。只改一个模块时，优先用模块级 Maven 命令。

| 范围 | 命令 |
|------|------|
| 后端全量测试 | `./mvnw test` |
| 单个模块测试 | `./mvnw -pl bootstrap -am test` |
| 单个模块编译 | `./mvnw -pl bootstrap -am compile` |
| 前端 lint | `cd frontend && npm run lint` |
| 前端构建 | `cd frontend && npm run build` |

## 参考文档

- [README.md](README.md) — 项目总览
- [docs/refactoring-summary.md](docs/refactoring-summary.md) — 架构和设计模式说明
- [docs/quick-start.md](docs/quick-start.md) — 启动和运行入口
- [frontend/TESTING.md](frontend/TESTING.md) — 前端开发和代理配置说明
