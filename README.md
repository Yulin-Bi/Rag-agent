# Rag-agent

Rag-agent 是一个 Java + React 的多模块 RAG 智能体项目，包含后端编排、AI 模型适配、MCP 服务和前端管理界面。

## 目录结构

- `bootstrap/`：业务接口、编排逻辑、知识库流程、上传、聊天和 RAG 编排
- `framework/`：共享基础设施和通用能力
- `infra-ai/`：AI 模型供应商适配、路由和集成逻辑
- `mcp-server/`：独立的 MCP 服务
- `frontend/`：React + Vite 前端界面
- `resources/`：数据库脚本、Docker 资源和知识库文档

## 技术栈

- Java 17
- Spring Boot 3.5.x
- Maven 多模块工程
- React 18 + TypeScript + Vite

## 本地运行

### 后端

在项目根目录执行：

```bash
mvn clean install
mvn -pl bootstrap -am spring-boot:run
```

如果你想先编译某个模块，也可以使用：

```bash
mvn -pl bootstrap -am compile
```

### 前端

进入前端目录后安装依赖并启动：

```bash
cd frontend
npm install
npm run dev
```

### 环境变量

前端本地配置文件是 `frontend/.env`，仓库里提供了示例文件 `frontend/.env.example`。

如果你需要新增本地配置，请复制示例文件后再修改，不要把真实密钥提交到仓库。

## 说明

- 如果你只想快速查看项目结构，直接浏览 `bootstrap/`、`infra-ai/` 和 `frontend/src/` 即可。
