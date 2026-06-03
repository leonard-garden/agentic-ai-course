# Bài 01: MCP — "JDBC cho AI"

> **Module**: 03 — Model Context Protocol  
> **Thời lượng**: ~3 giờ đọc + thực hành  
> **Level**: Intermediate

---

## Mục tiêu bài học

- Hiểu MCP là gì và tại sao nó ra đời
- Nắm vững architecture thông qua JDBC analogy
- Phân biệt 3 primitives: Tools, Resources, Prompts
- Biết transport types và khi nào dùng loại nào
- Có cái nhìn tổng quan về MCP ecosystem hiện tại (2026)

---

## 1. Bối cảnh: Vấn đề trước khi có MCP

### The Integration Explosion

Hãy tưởng tượng năm 2023. Bạn đang build một AI-powered application cho team backend:

- Bạn muốn AI đọc được PostgreSQL database → viết custom integration
- Bạn muốn AI browse GitHub issues → viết custom integration
- Bạn muốn AI đọc application logs → viết custom integration
- Bạn muốn AI call internal APIs → viết custom integration

Mỗi integration là một đoạn code riêng, cần maintain riêng, test riêng, document riêng. Và đây mới chỉ là **một** AI application của **bạn**.

Nhân lên cho toàn bộ industry:

```
Anthropic Claude cần integrate với: Postgres, MySQL, MongoDB, Redis, GitHub, GitLab,
Jira, Slack, Notion, Google Drive, S3, filesystem, REST APIs, GraphQL, SOAP...

OpenAI GPT cần integrate với: (cùng danh sách trên)

Google Gemini cần integrate với: (cùng danh sách trên)
```

Kết quả: **N AI models × M tools = N×M custom integrations**. Mỗi cái đều được viết khác nhau, behave khác nhau, break theo cách khác nhau.

Đây là chính xác vấn đề mà Java community đã giải quyết vào năm 1997 với JDBC.

---

## 2. JDBC Analogy — Core Mental Model

Đây là analogy quan trọng nhất của toàn bộ module. Hãy dành thời gian để thực sự hiểu nó.

### Câu chuyện JDBC

Trước JDBC, mỗi database vendor có proprietary API riêng:

```java
// Oracle-specific (1996)
OracleConnection conn = new OracleConnection("oracle.server", "user", "pass");
OracleResultSet rs = conn.oracleQuery("SELECT * FROM users");

// MySQL-specific (1996)  
MySQLDriver driver = MySQLDriver.connect("mysql.server");
MySQLResult result = driver.runQuery("SELECT * FROM users");

// PostgreSQL-specific (1996)
PgConnection pg = PgConnection.open("pg.server");
PgResults pgrs = pg.execute("SELECT * FROM users");
```

Vấn đề: Viết app dùng Oracle, sau muốn chuyển sang PostgreSQL → rewrite toàn bộ data access layer.

JDBC giải quyết bằng cách định nghĩa **standard interface**:

```java
// JDBC (1997) — works với bất kỳ database nào
Connection conn = DriverManager.getConnection("jdbc:postgresql://server/db", "user", "pass");
PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
stmt.setLong(1, userId);
ResultSet rs = stmt.executeQuery();
```

Architecture:

```
Java Application
      ↓ (calls standard JDBC API)
JDBC Driver Manager
      ↓ (routes to appropriate driver)
Database Driver (Oracle/MySQL/Postgres/H2/...)
      ↓ (speaks native protocol)
Actual Database
```

**Key insight**: Java app không cần biết đang dùng database nào. Chỉ cần nói JDBC.

### MCP — Cùng pattern, cho AI

MCP áp dụng chính xác pattern này cho AI integration:

```
Trước MCP:
AI Application ──custom code──→ PostgreSQL
AI Application ──custom code──→ GitHub  
AI Application ──custom code──→ Filesystem
AI Application ──custom code──→ Slack

Sau MCP:
AI Application ──MCP protocol──→ PostgreSQL MCP Server ──→ PostgreSQL
AI Application ──MCP protocol──→ GitHub MCP Server     ──→ GitHub
AI Application ──MCP protocol──→ Filesystem MCP Server ──→ Filesystem
AI Application ──MCP protocol──→ Slack MCP Server      ──→ Slack
```

So sánh trực tiếp:

| JDBC | MCP |
|------|-----|
| Java Application | AI Model / Claude Code |
| JDBC API | MCP Protocol (JSON-RPC) |
| Database Driver | MCP Server |
| Database | External System (DB, API, File, etc.) |
| `DriverManager.getConnection()` | MCP server config |
| `PreparedStatement` | MCP Tool call |
| `ResultSet` | MCP Tool response |
| Connection pooling | MCP session management |

### Vì sao analogy này đặc biệt phù hợp với Java engineers?

Vì bạn đã **sống** với JDBC suốt career. Bạn hiểu:
- Standard protocol tốt hơn proprietary integration
- Driver pattern cho phép swap implementation mà không break application
- Connection management và lifecycle matter
- Error handling cần chuẩn hóa

Tất cả những insight đó đều apply trực tiếp vào MCP.

---

## 3. MCP Architecture Deep Dive

### Các thành phần

```
┌─────────────────────────────────────────────────────────────┐
│                    MCP HOST                                  │
│  (Claude Code, Claude Desktop, hoặc custom AI application)  │
│                                                             │
│  ┌─────────────────┐    ┌─────────────────┐                │
│  │   MCP Client 1  │    │   MCP Client 2  │                │
│  │  (filesystem)   │    │   (postgres)    │                │
│  └────────┬────────┘    └────────┬────────┘                │
└───────────┼─────────────────────┼────────────────────────────┘
            │ JSON-RPC             │ JSON-RPC
            │ over stdio           │ over SSE/HTTP
            ▼                     ▼
┌───────────────────┐   ┌──────────────────────┐
│  MCP SERVER       │   │  MCP SERVER           │
│  (filesystem)     │   │  (postgres)           │
│                   │   │                       │
│  Tools:           │   │  Tools:               │
│  - read_file      │   │  - query              │
│  - write_file     │   │  - list_tables        │
│  - list_dir       │   │  - describe_table     │
│                   │   │                       │
│  Resources:       │   │  Resources:           │
│  - file://...     │   │  - db://schema        │
└─────────┬─────────┘   └──────────┬────────────┘
          │                        │
          ▼                        ▼
   Local Filesystem          PostgreSQL Database
```

### Ba vai trò

**MCP Host**: Application chứa AI model và quản lý MCP connections.
- Claude Code là một MCP Host
- Claude Desktop là một MCP Host
- Bạn có thể build custom MCP Host bằng Claude API + MCP SDK

**MCP Client**: Component bên trong Host, manage connection tới một MCP Server cụ thể.
- Mỗi configured server có một MCP Client tương ứng
- Client handles handshake, capability negotiation, message routing

**MCP Server**: Process expose tools/resources/prompts theo MCP protocol.
- Có thể là Node.js, Python, Java, Rust — bất kỳ ngôn ngữ nào
- Chạy như separate process (stdio) hoặc remote service (SSE)
- Chứa actual integration logic với external system

### Communication Flow

Khi Claude Code muốn call một MCP tool:

```
1. User types: "Query the users table and show me the last 5 signups"

2. Claude decides: cần dùng postgres MCP server

3. Claude Code (Host) → MCP Client → sends JSON-RPC request:
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "query",
    "arguments": {
      "sql": "SELECT id, email, created_at FROM users ORDER BY created_at DESC LIMIT 5"
    }
  },
  "id": 1
}

4. MCP Server receives request → executes query → returns response:
{
  "jsonrpc": "2.0",
  "result": {
    "content": [
      {
        "type": "text",
        "text": "id: 1042, email: user@example.com, created_at: 2026-06-03T10:30:00Z\n..."
      }
    ]
  },
  "id": 1
}

5. Claude Code passes result back to Claude model

6. Claude generates human-readable response to user
```

---

## 4. Ba Primitives: Tools, Resources, Prompts

MCP server có thể expose ba loại capability. Hiểu rõ sự khác biệt là critical.

### 4.1 Tools — "Functions AI có thể gọi"

Tools là **actions** — có side effects, nhận input, trả về output.

```
Đặc điểm:
- AI chủ động invoke khi cần
- Có thể có side effects (write DB, send email, call API)
- Có input parameters (strongly typed)
- Trả về kết quả

Ví dụ Tools:
- execute_sql(query: string) → QueryResult
- create_github_issue(title: string, body: string) → Issue
- send_slack_message(channel: string, message: string) → MessageId
- restart_service(serviceName: string) → Status
- run_flyway_migration() → MigrationResult
```

Java analogy: Tools giống như **methods bạn expose qua REST API**. Có endpoint, có request body, có response.

Trong Spring AI MCP, bạn annotate methods với `@Tool`:

```java
@Tool(description = "Execute a SQL query against the application database. 
                     Use for read-only SELECT queries.")
public String executeQuery(
    @ToolParam(description = "SQL SELECT query to execute") String sql
) {
    // implementation
}
```

### 4.2 Resources — "Data AI có thể đọc"

Resources là **data** — static hoặc dynamic, được identify bằng URI.

```
Đặc điểm:
- AI subscribe/read khi cần context
- Thường read-only (không có side effects)
- Addressed by URI scheme (file://, db://, git://, etc.)
- Có thể live-update (server pushes changes)

Ví dụ Resources:
- file:///project/src/main/java/...  → Java source files
- db://schema                        → Database schema description
- git://log/main                     → Recent git commits
- config://application.yml           → Application configuration
- metrics://jvm                      → JVM metrics snapshot
```

Java analogy: Resources giống như **@GetMapping endpoints** — pure read, trả về data, không thay đổi state.

```java
@McpResource(uri = "db://schema", 
             description = "Full database schema with tables, columns, and constraints")
public String getDatabaseSchema() {
    return schemaService.generateMarkdownSchema();
}
```

### 4.3 Prompts — "Template instructions"

Prompts là **pre-built instruction templates** — giúp guide AI trong specific tasks.

```
Đặc điểm:
- User-facing templates (không phải AI-triggered)
- Có thể nhận dynamic parameters
- Encode domain expertise vào reusable form

Ví dụ Prompts:
- "code-review" template với context về coding standards
- "incident-analysis" template cho production incidents  
- "migration-review" template cho Flyway migrations
- "performance-analysis" template cho slow queries
```

Java analogy: Prompts giống như **Velocity/Thymeleaf templates** — parameterized, reusable, encode structure.

```java
@McpPrompt(name = "analyze-slow-query",
           description = "Template for analyzing a slow database query")
public PromptMessage analyzeSlowQuery(
    @PromptParam("query") String sql,
    @PromptParam("execution_time_ms") long executionTime
) {
    return PromptMessage.user("""
        Analyze this slow query (took %d ms):
        ```sql
        %s
        ```
        Check for: missing indexes, N+1 patterns, unnecessary JOINs.
        """.formatted(executionTime, sql));
}
```

### Khi nào dùng cái nào?

| Use Case | Primitive | Lý do |
|----------|-----------|-------|
| Query database | Tool | Có input (SQL), trả về kết quả |
| Show DB schema | Resource | Static data, AI read để có context |
| Restart service | Tool | Side effect, cần explicit invocation |
| Application config | Resource | Reference data, read-only |
| Analyze incident | Prompt | Encode domain expertise, user-triggered |
| Create GitHub issue | Tool | Write operation, có side effect |
| Recent commits | Resource | Data source cho context |

---

## 5. Transport Types

MCP supports hai transport mechanisms. Hiểu cái này quan trọng cho deployment decisions.

### 5.1 stdio Transport — Local Servers

```
Claude Code ─── stdin/stdout ──→ MCP Server Process
```

**Cách hoạt động**: Claude Code spawn MCP server như một subprocess. Communication qua stdin (Claude → Server) và stdout (Server → Claude).

**Khi dùng**:
- MCP server chạy trên cùng machine với AI client
- Development và local tooling
- Servers cần access local filesystem
- Security-sensitive servers (không expose network)

**Config example**:
```json
{
  "mcpServers": {
    "my-java-server": {
      "command": "java",
      "args": ["-jar", "/path/to/mcp-server.jar"],
      "env": {
        "DATABASE_URL": "jdbc:postgresql://localhost/mydb"
      }
    }
  }
}
```

**Lifecycle**: Server process start khi Claude Code start, stop khi Claude Code stop. Nếu server crash → tự restart.

### 5.2 SSE Transport — Remote Servers

```
Claude Code ─── HTTP/SSE ──→ Remote MCP Server
```

**Cách hoạt động**: Server chạy như HTTP server. Client kết nối qua HTTP, nhận events qua Server-Sent Events.

**Khi dùng**:
- Shared MCP servers cho nhiều developers
- Servers cần chạy với elevated permissions trên separate machine
- Cloud-hosted tools
- Servers cần persistent state giữa các sessions

**Config example**:
```json
{
  "mcpServers": {
    "team-shared-server": {
      "url": "https://mcp-server.internal.company.com/sse",
      "headers": {
        "Authorization": "Bearer ${TEAM_MCP_TOKEN}"
      }
    }
  }
}
```

**Java engineer note**: SSE transport rất familiar nếu bạn đã dùng Spring WebFlux với `SseEmitter` hoặc reactive streams. Same concept, khác use case.

### stdio vs SSE — Decision Matrix

| Factor | stdio | SSE |
|--------|-------|-----|
| Setup complexity | Thấp | Cao hơn |
| Security | Cao (local only) | Cần auth/TLS |
| Sharing với team | Khó | Dễ |
| Persistent state | Không | Có thể |
| Debugging | Dễ | Phức tạp hơn |
| Production use | Local dev | Team/org deployment |

**Recommendation cho Java project**: Bắt đầu với stdio. Khi cần share với team thì migrate sang SSE.

---

## 6. MCP Ecosystem — Tháng 6/2026

### Tăng trưởng

MCP được Anthropic announce vào tháng 11/2024. Đến tháng 6/2026:

- **10,000+ public MCP servers** trên GitHub và MCP registry
- **97 million monthly SDK downloads** (npm + PyPI + Maven Central)
- **Adoption**: OpenAI, Google DeepMind, Microsoft Copilot đều support MCP
- **Enterprise**: Fortune 500 companies deploying internal MCP servers

Đây không còn là "Anthropic-specific thing". MCP đã trở thành **industry standard** cho AI-tool integration, tương tự JDBC trong Java world.

### Popular MCP Servers (Official)

```
@modelcontextprotocol/server-filesystem
  → Read/write local files, directory browsing
  → Use case: Claude browse project beyond codebase

@modelcontextprotocol/server-postgresql  
  → Query Postgres, describe schema, analyze data
  → Use case: Database debugging, query generation

@modelcontextprotocol/server-github
  → Read issues, PRs, code, commits
  → Use case: Code review context, issue management

@modelcontextprotocol/server-slack
  → Read/send messages, search channels
  → Use case: Incident response, team communication

@modelcontextprotocol/server-google-drive
  → Read Google Docs, Sheets, Drive files
  → Use case: Documentation access

@modelcontextprotocol/server-git
  → Git log, diff, blame operations
  → Use case: Code history analysis
```

### Community MCP Servers (Java/Spring relevant)

```
mcp-server-jira        → Jira issue management
mcp-server-confluence  → Confluence documentation  
mcp-server-sonarqube   → Code quality metrics
mcp-server-grafana     → Metrics and dashboards
mcp-server-kafka       → Kafka topic inspection
mcp-server-redis       → Redis operations
```

### Java/Spring AI MCP SDK

Spring AI có first-class MCP support từ version 1.0.0 (released Q1 2025):

```xml
<!-- MCP Server -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-mcp-server-spring-boot-starter</artifactId>
    <version>1.0.0</version>
</dependency>

<!-- MCP Client (nếu build custom AI app) -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-mcp-client-spring-boot-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

Maven downloads của Spring AI MCP artifacts tăng 340% từ Q1 2025 đến Q1 2026 theo Maven Central stats.

---

## 7. Security Model

Đây là câu hỏi đầu tiên mà security-conscious Java engineers thường hỏi: *"Nếu Claude có thể call tools thực sự, security model là gì?"*

### MCP Security Principles

**1. Explicit Authorization**

User (không phải AI) configure MCP servers. Claude không thể tự thêm servers. Mỗi server được explicitly listed trong config file.

**2. Tool Disclosure**

Khi Claude Code connect tới MCP server, server announce tất cả available tools. User thấy chính xác tools nào Claude có access.

**3. Local Execution**

stdio servers chạy trên local machine của user, dưới user's own permissions. Không có elevated privileges.

**4. No Network Exposure (stdio)**

stdio MCP servers không listen trên bất kỳ port nào. Zero network attack surface.

**5. Environment Variable Isolation**

Secrets (database passwords, API keys) passed qua environment variables trong config, không hardcoded trong server code.

### Security Best Practices cho Java MCP Servers

```java
// 1. Validate all tool inputs
@Tool(description = "Execute SQL query")
public String executeQuery(
    @ToolParam(description = "SQL SELECT query") String sql
) {
    // Validate: only allow SELECT statements
    if (!sql.trim().toUpperCase().startsWith("SELECT")) {
        throw new McpToolException("Only SELECT queries are allowed");
    }
    
    // Use parameterized queries, never string concatenation
    // Never allow: DROP, DELETE, UPDATE, INSERT, TRUNCATE, ALTER
    return jdbcTemplate.queryForList(sql).toString();
}

// 2. Scope permissions appropriately
// DB user cho MCP server chỉ có SELECT permissions
// Không dùng superuser hoặc application user với write permissions

// 3. Audit logging
@Tool(description = "Get sensitive data")
public String getSensitiveData(...) {
    log.info("MCP tool called: getSensitiveData, by session: {}", 
             mcpContext.getSessionId());
    // ... implementation
}
```

---

## 8. MCP vs Tool Use (Function Calling)

Câu hỏi thường gặp: *"MCP với Claude API Tool Use khác nhau chỗ nào?"*

| Aspect | Claude API Tool Use | MCP |
|--------|---------------------|-----|
| Scope | Single API call | Persistent session |
| Tool definition | Inline trong API request | Declared bởi MCP server |
| Reusability | Per-application | Cross-application |
| Discovery | Manual (bạn define) | Automatic (server announces) |
| Transport | HTTPS (Anthropic API) | stdio hoặc SSE |
| Ecosystem | Custom | 10,000+ ready-made servers |
| Java SDK | Anthropic Java SDK | Spring AI MCP |

**Khi dùng Tool Use**: Build custom AI application cần specific tools cho use case của bạn.

**Khi dùng MCP**: Connect AI với existing systems và tools, đặc biệt khi muốn share across multiple AI applications.

**Trong thực tế**: Hai thứ complement nhau. Tool Use cho application-specific logic, MCP cho infrastructure-level integrations.

---

## 9. Hands-on Mental Exercise

Trước khi làm bài tập, hãy mentally walkthrough scenario này:

Bạn là Java backend engineer tại một e-commerce company. Team đang debug một production issue: order processing chậm đột ngột sau deploy mới.

**Với MCP setup**, Claude Code có thể:

1. Dùng **postgres MCP server** → Query slow query log, identify bottleneck queries
2. Dùng **github MCP server** → Compare schema changes giữa 2 versions, tìm missing index
3. Dùng **filesystem MCP server** → Read Flyway migration files liên quan
4. Dùng **custom actuator MCP server** (bạn sẽ build trong Bài 03) → Check JVM metrics, connection pool status, thread dump

Tất cả trong một Claude Code session, không cần copy-paste data giữa tools.

**Không có MCP**: Bạn phải manually query database, check GitHub, đọc files, và copy tất cả vào Claude prompt. Hoặc viết custom integration script.

---

## 10. Exercise

### Exercise 1.1 — Use Case Identification

Liệt kê **5 MCP use cases cụ thể** cho Java enterprise context của bạn. Với mỗi use case, trả lời:

```
Use case: [mô tả ngắn]
External system: [hệ thống gì — DB, API, file, etc.]
Primitive type: [Tool / Resource / Prompt]
Transport: [stdio / SSE]
Business value: [tại sao điều này có ích]
Security consideration: [risk gì và cách mitigate]
```

**Gợi ý để bắt đầu suy nghĩ**:
- Debugging production issues (logs, metrics, DB state)
- Code review assistance (GitHub, SonarQube, test coverage)
- Database management (schema exploration, query generation)
- CI/CD pipeline visibility (build status, deployment history)
- Documentation access (Confluence, internal wikis)

### Exercise 1.2 — JDBC to MCP Mapping

Fill vào bảng này với kiến thức từ bài học:

| JDBC Concept | MCP Equivalent | Ví dụ cụ thể |
|--------------|----------------|--------------|
| `DriverManager.getConnection()` | ? | ? |
| `PreparedStatement` | ? | ? |
| `ResultSet` | ? | ? |
| JDBC URL (`jdbc:postgresql://...`) | ? | ? |
| Database Driver JAR | ? | ? |
| Connection Pool | ? | ? |

### Exercise 1.3 — Architecture Diagram

Vẽ (hoặc mô tả) MCP architecture cho scenario sau:

> Spring Boot microservice với PostgreSQL database, deployed trên AWS. Team muốn Claude Code có thể query production DB (read-only), xem application logs trên CloudWatch, và browse code trên GitHub.

Chú ý: Xác định transport type phù hợp cho từng MCP server và lý do.

---

## Tóm tắt bài học

| Concept | Key Point |
|---------|-----------|
| MCP là gì | Open standard protocol kết nối AI với external tools/data |
| JDBC analogy | JDBC: Java→DB :: MCP: AI→Tools. Same driver pattern, different domain |
| Architecture | Host (Claude Code) → Client → Server → External System |
| Tools | Functions AI call, có input/output, có thể có side effects |
| Resources | Data AI reads, addressed by URI, thường read-only |
| Prompts | Reusable instruction templates, user-triggered |
| stdio transport | Local process, high security, dev/local use |
| SSE transport | Remote HTTP, shareable, team deployment |
| Ecosystem | 10,000+ servers, 97M monthly downloads, industry standard |
| Security | Explicit auth, local execution, env var secrets |

---

## Bài tiếp theo

**Bài 02: Consuming Existing MCP Servers** — Hands-on config và sử dụng postgres, github, và filesystem MCP servers trong Spring Boot project context. Bạn sẽ connect Claude Code trực tiếp tới database và GitHub repo của mình.

---

*Bài 01/03 — Module 03: Model Context Protocol*
