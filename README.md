# Yu AI Agent 使用指南

本项目提供两个可直接使用的 AI 应用：带多轮记忆的 **AI 恋爱大师**，以及可调用本地工具完成任务的 **YuManus 超级智能体**。包含 Spring Boot 后端、Vue 3 前端和一个可选的图片搜索 MCP 服务。

## 环境要求

- JDK 21
- Node.js 16+、npm 7+
- 阿里云 DashScope API Key（对话和向量嵌入必需）

可选功能还需要：联网搜索的 SearchAPI Key、PgVector 的 PostgreSQL + pgvector，或 MCP 所需的 Node.js / Java。

## 配置密钥

编辑 `src/main/resources/application.yml`，将 API Key 改为环境变量引用：

```yaml
spring:
  ai:
    dashscope:
      api-key: ${DASHSCOPE_API_KEY}
      chat:
        options:
          model: qwen-plus
search-api:
  api-key: ${SEARCH_API_KEY}
```

PowerShell 临时设置：

```powershell
$env:DASHSCOPE_API_KEY = "你的 DashScope API Key"
$env:SEARCH_API_KEY = "你的 SearchAPI Key"
```

在 IntelliJ IDEA 以调试模式启动时，终端中的临时环境变量不会自动传入 Run/Debug Configuration。请打开 **Run → Edit Configurations → YuAiAgentApplication**，在 **Environment variables** 中添加：

```text
DASHSCOPE_API_KEY=你的 DashScope API Key;SEARCH_API_KEY=你的 SearchAPI Key
```

保存后重新点击 Debug。也可以在 Windows 用户环境变量中创建这两个变量，然后**完全退出并重新打开 IDEA**。切勿把密钥写进提交到 Git 的 `application.yml`。

`model` 必须替换为你的 DashScope 账户可用的模型。当前依赖版本下，`qwen3.7-plus` 会导致 DashScope 返回 `InvalidParameter: url error`；请使用 `qwen-plus`，或先确认目标模型与当前 SDK 版本兼容。当前配置文件里存在疑似真实密钥：请不要提交真实密钥，建议立即在服务商控制台轮换该密钥，再改用环境变量或不提交的 `application-prod.yml`。

### Ollama（可选）

`application.yml` 已声明 Ollama 的本地地址与 `gemma3:1b`，但现有 Web API 注入的是 DashScope 的 `ChatModel`，所以页面和接口仍需要 DashScope Key。若要运行本地模型示例或自行切换模型：

```powershell
ollama pull gemma3:1b
```

## 启动后端

在项目根目录执行：

```powershell
.\mvnw.cmd spring-boot:run
```

后端地址为 `http://localhost:8123`，API 均带 `/api` 前缀。验证服务：

```powershell
Invoke-WebRequest http://localhost:8123/api/health
```

应返回 `ok`。接口文档：

- http://localhost:8123/api/swagger-ui.html
- http://localhost:8123/api/doc.html（Knife4j，若当前依赖版本提供此入口）

## 启动前端

另开终端：

```powershell
cd yu-ai-agent-frontend
npm install
npm run dev
```

打开 Vite 输出的地址（默认 `http://localhost:3000`）。开发环境会访问 `http://localhost:8123/api`，后端已放行跨域。

构建生产前端：

```powershell
npm run build
```

生产前端请求同域 `/api`，因此反向代理需把 `/api` 转发到后端 8123 端口。

## API 使用

所有接口均为 `GET`。`message` 需 URL 编码；恋爱大师使用相同 `chatId` 才会保留一段会话的上下文。记忆仅在内存中保存，最多 20 条消息，服务重启后清空。

| 功能 | 地址 | 参数 | 返回 |
| --- | --- | --- | --- |
| 健康检查 | `/api/health` | 无 | `ok` |
| 恋爱大师（同步） | `/api/ai/love_app/chat/sync` | `message`, `chatId` | 完整文本 |
| 恋爱大师（SSE） | `/api/ai/love_app/chat/sse` | `message`, `chatId` | 文本流 |
| 恋爱大师（标准 SSE 事件） | `/api/ai/love_app/chat/server_sent_event` | `message`, `chatId` | SSE event stream |
| 恋爱大师（SseEmitter） | `/api/ai/love_app/chat/sse_emitter` | `message`, `chatId` | SSE event stream |
| YuManus 智能体 | `/api/ai/manus/chat` | `message` | SSE event stream |

同步调用：

```powershell
curl.exe -G "http://localhost:8123/api/ai/love_app/chat/sync" `
  --data-urlencode "message=我和伴侣最近总因小事争吵，怎么办？" `
  --data-urlencode "chatId=demo-user-001"
```

查看流式输出：

```powershell
curl.exe -N -G "http://localhost:8123/api/ai/love_app/chat/sse" `
  --data-urlencode "message=给我一个破冰聊天的建议" `
  --data-urlencode "chatId=demo-user-001"
```

YuManus 能让模型读写本机文件、下载资源、执行终端命令和生成 PDF。仅在隔离的开发环境使用，且不要向不可信用户公开该接口。

## 启用当前临时注释的配置

### PgVector 持久化知识库

默认是内存向量库：每次启动读取 `src/main/resources/document` 下的 Markdown。要改用 PostgreSQL + pgvector：

1. 准备启用了 `vector` 扩展的 PostgreSQL。
2. 取消 `application.yml` 中 `spring.datasource` 与 `spring.ai.vectorstore.pgvector` 两段的注释并填写实际连接信息：

   ```yaml
   spring:
     datasource:
       url: jdbc:postgresql://localhost:5432/yu_ai_agent
       username: postgres
       password: 你的数据库密码
     ai:
       vectorstore:
         pgvector:
           index-type: HNSW
           dimensions: 1536
           distance-type: COSINE_DISTANCE
           max-document-batch-size: 10000
   ```

3. 取消 `src/main/java/com/yupi/yuaiagent/rag/PgVectorVectorStoreConfig.java` 中 `@Configuration` 的注释。
4. 在 `LoveApp#doChatWithRag` 中启用 `new QuestionAnswerAdvisor(pgVectorVectorStore)`，并停用当前的 `loveAppVectorStore` 顾问；否则仍会使用内存库。

`dimensions: 1536` 必须等于所用嵌入模型的向量维度。代码首次启动会创建 `public.vector_store` 并写入文档。

### MCP 工具服务

取消 `application.yml` 的 MCP 注释，并按需保留 SSE 或 stdio：

```yaml
spring:
  ai:
    mcp:
      client:
        sse:
          connections:
            server1:
              url: http://localhost:8127
        stdio:
          servers-configuration: classpath:mcp-servers.json
```

- **SSE 图片搜索服务**：在 `yu-image-search-mcp-server` 执行 `.\mvnw.cmd spring-boot:run`，服务默认监听 8127，再启用 `server1`。
- **stdio 服务**：主服务将读取 `src/main/resources/mcp-servers.json`。高德地图条目需填入 `AMAP_MAPS_API_KEY`，且系统必须能找到 `npx.cmd`。
- **图片搜索 stdio 服务**：先在 `yu-image-search-mcp-server` 执行 `.\mvnw.cmd package`。另外将 `ImageSearchTool.java` 的 Pexels API Key 改为自己的；该密钥现在不从 YAML 读取。

只启用 MCP 客户端配置并不会让现有页面自动使用 MCP：`LoveApp#doChatWithMcp` 已实现调用，但 `AiController` 尚未为它暴露 HTTP 接口；YuManus 也只使用本地注册工具数组，不会自动加入 `ToolCallbackProvider` 的 MCP 工具。

## 排错

- **数据库连接失败**：PgVector 已启用但数据库或 `vector` 扩展尚未就绪；恢复注释使用内存库，或检查连接信息。
- **401 / 额度不足**：检查 DashScope、SearchAPI 的密钥、模型权限与余额。
- **前端无法连接**：先访问健康检查，再核对开发环境地址 `http://localhost:8123/api`。
- **SSE 无响应**：先用上面的 `curl.exe -N` 命令直连，并查看后端模型调用日志。
- **MCP 启动失败**：检查 8127 端口、Java / `npx.cmd` 的 PATH，以及 stdio 模式引用的 JAR 是否已构建。
- **`Port 8123 was already in use`**：已有后端实例占用端口。停止 IDEA 中旧的运行实例，或在 PowerShell 执行 `Get-NetTCPConnection -LocalPort 8123 -State Listen` 查出 `OwningProcess` 后再确认并停止对应 Java 进程。
