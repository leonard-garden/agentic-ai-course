# Project: AI Database Assistant

> **Tuần 3, Buổi 11-12** (~6 giờ)  
> **Loại**: Capstone project tổng hợp toàn bộ Module 01  
> **Output**: Production-ready Spring Boot REST API cho phép query database bằng natural language  
> **Kiến thức áp dụng**: Lesson 01 (setup) + 02 (messages/streaming) + 03 (tool use) + 04 (prompt engineering)

---

## Tổng quan Project

Bạn sẽ xây dựng **AI Database Assistant** — một Spring Boot service nhận câu hỏi bằng natural language từ user (tiếng Việt hoặc tiếng Anh), tự động generate SQL query phù hợp, thực thi trên database thật, và trả về kết quả đã được diễn giải thành text dễ hiểu.

### Demo scenario

```
User: "Tháng này có bao nhiêu đơn hàng mới? Và top 3 khách hàng mua nhiều nhất?"

AI Database Assistant:
→ [Gọi tool: list_tables]          # Hiểu structure của DB
→ [Gọi tool: execute_query]        # SELECT COUNT(*) FROM orders WHERE ...
→ [Gọi tool: execute_query]        # SELECT customer_id, SUM(total) FROM orders GROUP BY ...
→ Synthesize results

Response: "Tháng này có 247 đơn hàng mới (tăng 18% so với tháng trước).
Top 3 khách hàng theo giá trị mua:
1. Nguyen Van A (ID: 1042) - 45 đơn, tổng $12,450
2. Tran Thi B (ID: 0891) - 38 đơn, tổng $9,870  
3. Le Van C (ID: 2103) - 29 đơn, tổng $7,230"
```

---

## Requirements

### Functional Requirements

| # | Requirement | Priority |
|---|------------|----------|
| F1 | REST API `POST /api/assistant/ask` nhận natural language question | MUST |
| F2 | Trả về natural language answer (không phải raw SQL/JSON) | MUST |
| F3 | Hỗ trợ multi-step questions (cần nhiều queries) | MUST |
| F4 | Streaming response qua SSE: `GET /api/assistant/ask/stream` | MUST |
| F5 | Session-based conversation history (multi-turn) | MUST |
| F6 | Chỉ cho phép SELECT queries — bảo vệ khỏi data modification | MUST |
| F7 | Trả về danh sách SQL queries đã chạy (audit trail) | SHOULD |
| F8 | Estimate query cost (tokens used) trong response | SHOULD |
| F9 | `/api/assistant/health` endpoint với DB connectivity check | SHOULD |
| F10 | Rate limiting: tối đa 10 requests/minute/session | COULD |

### Non-Functional Requirements

- **Latency**: P95 < 30 giây (bao gồm Claude processing time)
- **Security**: Parameterized queries only, input validation, no DDL allowed
- **Observability**: Log mỗi request với tokens used, query count, duration
- **Error handling**: Friendly error messages, không expose stack traces ra API

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Spring Boot App                       │
│                                                         │
│  ┌──────────────────┐    ┌──────────────────────────┐  │
│  │  AssistantController│   │  SessionController       │  │
│  │  POST /ask         │   │  DELETE /sessions/{id}   │  │
│  │  GET  /ask/stream  │   │                          │  │
│  └────────┬───────────┘   └──────────────────────────┘  │
│           │                                             │
│  ┌────────▼───────────────────────────────────────┐    │
│  │           DatabaseAssistantService              │    │
│  │                                                 │    │
│  │  ┌────────────────┐    ┌──────────────────┐    │    │
│  │  │ ToolUseEngine  │    │ SessionStore      │    │    │
│  │  │ (Claude orch.) │    │ (in-memory/Redis) │    │    │
│  │  └───────┬────────┘    └──────────────────┘    │    │
│  │          │                                      │    │
│  │  ┌───────▼──────────────────────────┐           │    │
│  │  │         Tool Registry            │           │    │
│  │  │  - list_tables                   │           │    │
│  │  │  - get_table_schema              │           │    │
│  │  │  - execute_query                 │           │    │
│  │  │  - get_query_plan                │           │    │
│  │  └───────┬──────────────────────────┘           │    │
│  └──────────┼──────────────────────────────────────┘    │
│             │                                           │
│  ┌──────────▼──────────┐    ┌─────────────────────┐    │
│  │   JdbcQueryExecutor │    │  AnthropicClient     │    │
│  │   (read-only)       │    │  (Claude Sonnet 4.6) │    │
│  └──────────┬──────────┘    └─────────────────────┘    │
│             │                                           │
└─────────────┼───────────────────────────────────────────┘
              │
    ┌─────────▼──────────┐
    │   PostgreSQL DB     │
    │   (e-commerce data) │
    └────────────────────┘
```

---

## Database Setup

### Tạo database và sample data

```sql
-- Chạy script này để tạo sample e-commerce database
CREATE DATABASE ai_assistant_demo;

\c ai_assistant_demo

-- Customers
CREATE TABLE customers (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20),
    city VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Products
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    category VARCHAR(50),
    price DECIMAL(10,2) NOT NULL,
    stock_quantity INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Orders
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT REFERENCES customers(id),
    status VARCHAR(20) DEFAULT 'PENDING',
    -- status: PENDING, PROCESSING, SHIPPED, DELIVERED, CANCELLED
    total_amount DECIMAL(10,2),
    shipping_address TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Order items
CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT REFERENCES orders(id),
    product_id BIGINT REFERENCES products(id),
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL
);

-- Insert sample data
INSERT INTO customers (name, email, phone, city) VALUES
    ('Nguyen Van A', 'nguyenvana@email.com', '0901234567', 'Ho Chi Minh City'),
    ('Tran Thi B', 'tranthib@email.com', '0912345678', 'Hanoi'),
    ('Le Van C', 'levanc@email.com', '0923456789', 'Da Nang'),
    ('Pham Thi D', 'phamthid@email.com', '0934567890', 'Ho Chi Minh City'),
    ('Hoang Van E', 'hoangvane@email.com', '0945678901', 'Hanoi');

INSERT INTO products (name, category, price, stock_quantity) VALUES
    ('MacBook Pro 14"', 'electronics', 1999.99, 15),
    ('iPhone 15 Pro', 'electronics', 999.99, 42),
    ('Spring Boot in Action', 'books', 45.99, 100),
    ('Wireless Keyboard', 'electronics', 79.99, 30),
    ('Java Design Patterns', 'books', 39.99, 75),
    ('USB-C Hub', 'accessories', 49.99, 60),
    ('Monitor 27"', 'electronics', 449.99, 8),
    ('Standing Desk', 'furniture', 599.99, 5);

-- Generate orders for last 3 months
INSERT INTO orders (customer_id, status, total_amount, created_at) VALUES
    (1, 'DELIVERED', 1999.99, NOW() - INTERVAL '45 days'),
    (2, 'DELIVERED', 999.99, NOW() - INTERVAL '30 days'),
    (1, 'SHIPPED', 125.98, NOW() - INTERVAL '7 days'),
    (3, 'PROCESSING', 449.99, NOW() - INTERVAL '2 days'),
    (4, 'PENDING', 79.99, NOW() - INTERVAL '1 day'),
    (2, 'DELIVERED', 1449.98, NOW() - INTERVAL '15 days'),
    (5, 'CANCELLED', 39.99, NOW() - INTERVAL '20 days'),
    (1, 'DELIVERED', 599.99, NOW() - INTERVAL '60 days');

INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1, 1, 1, 1999.99),
    (2, 2, 1, 999.99),
    (3, 3, 1, 45.99), (3, 5, 1, 39.99), (3, 6, 1, 49.99),
    (4, 7, 1, 449.99),
    (5, 4, 1, 79.99),
    (6, 2, 1, 999.99), (6, 7, 1, 449.99),
    (7, 5, 1, 39.99),
    (8, 8, 1, 599.99);
```

---

## Step-by-Step Implementation Guide

### Step 1: Project Setup (30 phút)

**Tạo Spring Boot project:**
```bash
# Dùng Spring Initializr hoặc tạo thủ công
mvn archetype:generate \
  -DgroupId=com.example \
  -DartifactId=ai-database-assistant \
  -DarchetypeArtifactId=maven-archetype-quickstart
```

**`pom.xml` dependencies:**
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
</parent>

<dependencies>
    <!-- Spring Boot -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- Anthropic SDK -->
    <dependency>
        <groupId>com.anthropic</groupId>
        <artifactId>anthropic-sdk-java</artifactId>
        <version>0.8.0-alpha.1</version>
    </dependency>

    <!-- Database -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>

    <!-- JSON -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>

    <!-- Testing -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

**`application.properties`:**
```properties
# Server
server.port=8080

# Database
spring.datasource.url=jdbc:postgresql://localhost:5432/ai_assistant_demo
spring.datasource.username=postgres
spring.datasource.password=${DB_PASSWORD:postgres}
spring.datasource.driver-class-name=org.postgresql.Driver

# Anthropic
anthropic.api-key=${ANTHROPIC_API_KEY}
anthropic.model=claude-sonnet-4-6
anthropic.max-tokens=4096
anthropic.max-tool-iterations=8

# Logging
logging.level.com.example=DEBUG
logging.level.com.anthropic=INFO
```

---

### Step 2: Core Domain Classes (45 phút)

```java
// ── DTOs ──────────────────────────────────────────────────

package com.example.assistant.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public record AskRequest(
    @NotBlank(message = "Question cannot be empty")
    @Size(max = 2000, message = "Question too long (max 2000 chars)")
    String question,

    String sessionId  // Optional: null = new session
) {}

public record AskResponse(
    String sessionId,
    String answer,
    List<String> queriesExecuted,  // SQL queries đã chạy (audit)
    TokenUsage tokenUsage,
    long durationMs
) {}

public record TokenUsage(long inputTokens, long outputTokens) {}

// ── Session ───────────────────────────────────────────────

package com.example.assistant.session;

import com.anthropic.sdk.models.MessageParam;
import java.time.Instant;
import java.util.*;

public class ConversationSession {

    private final String sessionId;
    private final List<MessageParam> history;
    private final Instant createdAt;
    private Instant lastAccessedAt;

    public ConversationSession(String sessionId) {
        this.sessionId = sessionId;
        this.history = new ArrayList<>();
        this.createdAt = Instant.now();
        this.lastAccessedAt = Instant.now();
    }

    public String getSessionId() { return sessionId; }

    public List<MessageParam> getHistory() {
        lastAccessedAt = Instant.now();
        return Collections.unmodifiableList(history);
    }

    public void addMessage(MessageParam message) {
        history.add(message);
        lastAccessedAt = Instant.now();
        // TODO: Truncate nếu history > 20 messages
    }

    public boolean isExpired(java.time.Duration maxAge) {
        return lastAccessedAt.isBefore(Instant.now().minus(maxAge));
    }
}
```

---

### Step 3: Session Store (30 phút)

```java
package com.example.assistant.session;

import org.springframework.stereotype.Component;
import java.time.Duration;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class SessionStore {

    // TODO: Trong production, dùng Redis thay vì in-memory
    private final ConcurrentHashMap<String, ConversationSession> sessions =
        new ConcurrentHashMap<>();

    private static final Duration SESSION_TTL = Duration.ofHours(1);

    /**
     * Lấy hoặc tạo session mới
     */
    public ConversationSession getOrCreate(String sessionId) {
        if (sessionId == null || sessionId.isBlank()) {
            // Tạo session ID mới
            return createNew();
        }
        return sessions.computeIfAbsent(sessionId, ConversationSession::new);
    }

    public ConversationSession createNew() {
        String newId = UUID.randomUUID().toString();
        ConversationSession session = new ConversationSession(newId);
        sessions.put(newId, session);
        return session;
    }

    public void delete(String sessionId) {
        sessions.remove(sessionId);
    }

    /**
     * Cleanup expired sessions — gọi định kỳ (schedule @Scheduled)
     */
    public int cleanupExpired() {
        // TODO: Implement cleanup
        // Hint: sessions.entrySet().removeIf(e -> e.getValue().isExpired(SESSION_TTL))
        return 0; // Return count of removed sessions
    }

    public int getActiveSessionCount() {
        return sessions.size();
    }
}
```

---

### Step 4: Tool Definitions & Handlers (60 phút)

```java
package com.example.assistant.tools;

import com.anthropic.sdk.models.*;
import com.anthropic.sdk.core.JsonValue;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

import java.util.*;

@Component
public class DatabaseToolRegistry {

    private static final Logger log = LoggerFactory.getLogger(DatabaseToolRegistry.class);
    private static final ObjectMapper mapper = new ObjectMapper();

    private final JdbcTemplate jdbcTemplate;

    // Track queries executed trong current request
    private final ThreadLocal<List<String>> executedQueries =
        ThreadLocal.withInitial(ArrayList::new);

    // Allowed SQL prefixes — security whitelist
    private static final Set<String> SAFE_PREFIXES = Set.of(
        "SELECT", "WITH", "EXPLAIN"
    );

    public DatabaseToolRegistry(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    // ── Tool Definitions ─────────────────────────────────────

    public List<Tool> getAllTools() {
        return List.of(
            listTablesTool(),
            getTableSchemaTool(),
            executeQueryTool(),
            getQueryPlanTool()
        );
    }

    private Tool listTablesTool() {
        return Tool.builder()
            .name("list_tables")
            .description("""
                List all tables in the database with basic info (name, approximate row count).
                ALWAYS call this first when starting a new conversation or when unsure
                what tables exist. This gives you the lay of the land before writing queries.
                """)
            .inputSchema(Tool.InputSchema.builder()
                .type("object")
                // No required properties
                .build())
            .build();
    }

    private Tool getTableSchemaTool() {
        return Tool.builder()
            .name("get_table_schema")
            .description("""
                Get the column definitions, data types, and constraints for a specific table.
                Call this before writing a query for a table you haven't seen yet.
                Returns: column names, types, nullable, primary keys, foreign keys.
                """)
            .inputSchema(Tool.InputSchema.builder()
                .type("object")
                .putProperty("table_name", JsonValue.from(Map.of(
                    "type", "string",
                    "description", "Name of the database table to inspect"
                )))
                .addRequired("table_name")
                .build())
            .build();
    }

    private Tool executeQueryTool() {
        return Tool.builder()
            .name("execute_query")
            .description("""
                Execute a read-only SQL SELECT query and return the results.
                Only SELECT and WITH (CTE) statements are allowed — no INSERT/UPDATE/DELETE.
                Results are limited to 200 rows maximum.
                Include a human-readable description of what the query does.
                """)
            .inputSchema(Tool.InputSchema.builder()
                .type("object")
                .putProperty("sql", JsonValue.from(Map.of(
                    "type", "string",
                    "description", "The SQL SELECT query to execute"
                )))
                .putProperty("description", JsonValue.from(Map.of(
                    "type", "string",
                    "description", "Brief description of what this query answers"
                )))
                .addRequired("sql")
                .addRequired("description")
                .build())
            .build();
    }

    private Tool getQueryPlanTool() {
        return Tool.builder()
            .name("get_query_plan")
            .description("""
                Get the execution plan for a SQL query using EXPLAIN ANALYZE.
                Use this when the user asks about query performance or when a query
                seems slow. Returns estimated and actual execution times.
                """)
            .inputSchema(Tool.InputSchema.builder()
                .type("object")
                .putProperty("sql", JsonValue.from(Map.of(
                    "type", "string",
                    "description", "The SQL query to analyze"
                )))
                .addRequired("sql")
                .build())
            .build();
    }

    // ── Tool Handlers ─────────────────────────────────────────

    public String handleListTables(JsonNode input) throws Exception {
        // TODO: Query information_schema để list tables
        // Query: SELECT table_name FROM information_schema.tables
        //        WHERE table_schema = 'public' AND table_type = 'BASE TABLE'

        List<Map<String, Object>> tables = jdbcTemplate.queryForList("""
            SELECT
                t.table_name,
                obj_description(c.oid) as description
            FROM information_schema.tables t
            LEFT JOIN pg_class c ON c.relname = t.table_name
            WHERE t.table_schema = 'public'
              AND t.table_type = 'BASE TABLE'
            ORDER BY t.table_name
            """);

        // TODO: Thêm approximate row count cho mỗi table
        // Hint: pg_stat_user_tables có n_live_tup (estimated)

        return mapper.writeValueAsString(Map.of(
            "tables", tables,
            "count", tables.size()
        ));
    }

    public String handleGetTableSchema(JsonNode input) throws Exception {
        String tableName = input.get("table_name").asText();

        // TODO: Validate tableName (chỉ alphanumeric + underscore)
        if (!tableName.matches("[a-zA-Z_][a-zA-Z0-9_]*")) {
            throw new IllegalArgumentException(
                "Invalid table name: " + tableName);
        }

        // TODO: Query information_schema.columns
        // Include: column_name, data_type, is_nullable, column_default

        // TODO: Query information_schema.table_constraints + key_column_usage
        // Include: primary keys, foreign keys

        // TODO: Return combined schema info as JSON
        return "{}"; // Replace with actual implementation
    }

    public String handleExecuteQuery(JsonNode input) throws Exception {
        String sql = input.get("sql").asText().trim();
        String description = input.get("description").asText();

        // Security: validate SQL is read-only
        validateReadOnly(sql);

        // Add LIMIT if not present
        String safeSql = addLimitIfMissing(sql, 200);

        log.info("Executing query: {} | SQL: {}", description, safeSql);

        // Track for audit trail
        executedQueries.get().add(safeSql);

        try {
            List<Map<String, Object>> rows = jdbcTemplate.queryForList(safeSql);

            return mapper.writeValueAsString(Map.of(
                "description", description,
                "sql_executed", safeSql,
                "row_count", rows.size(),
                "rows", rows
            ));
        } catch (Exception e) {
            throw new RuntimeException(
                "Query execution failed: " + e.getMessage() +
                "\nSQL: " + safeSql, e);
        }
    }

    public String handleGetQueryPlan(JsonNode input) throws Exception {
        String sql = input.get("sql").asText().trim();
        validateReadOnly(sql);

        String explainSql = "EXPLAIN (ANALYZE, FORMAT JSON) " + sql;

        try {
            List<Map<String, Object>> plan = jdbcTemplate.queryForList(explainSql);
            return mapper.writeValueAsString(Map.of(
                "original_sql", sql,
                "execution_plan", plan
            ));
        } catch (Exception e) {
            throw new RuntimeException("Failed to get query plan: " + e.getMessage(), e);
        }
    }

    // ── Helper Methods ─────────────────────────────────────────

    private void validateReadOnly(String sql) {
        String upperSql = sql.trim().toUpperCase();
        boolean safe = SAFE_PREFIXES.stream().anyMatch(upperSql::startsWith);
        if (!safe) {
            throw new SecurityException(
                "Only SELECT queries are allowed. Got statement starting with: "
                + sql.substring(0, Math.min(20, sql.length())));
        }

        // Extra check: look for dangerous keywords
        String[] dangerousKeywords = {"INSERT", "UPDATE", "DELETE", "DROP",
                                      "CREATE", "ALTER", "TRUNCATE", "GRANT"};
        for (String keyword : dangerousKeywords) {
            // Check as whole word (not part of column/table name)
            if (upperSql.matches(".*\\b" + keyword + "\\b.*")) {
                throw new SecurityException(
                    "Query contains forbidden keyword: " + keyword);
            }
        }
    }

    private String addLimitIfMissing(String sql, int limit) {
        if (!sql.toUpperCase().contains("LIMIT")) {
            return sql + " LIMIT " + limit;
        }
        return sql;
    }

    public List<String> getExecutedQueries() {
        return List.copyOf(executedQueries.get());
    }

    public void clearExecutedQueries() {
        executedQueries.get().clear();
    }
}
```

---

### Step 5: Core ToolUseEngine (60 phút)

```java
package com.example.assistant.engine;

import com.anthropic.sdk.AnthropicClient;
import com.anthropic.sdk.models.*;
import com.anthropic.sdk.core.JsonValue;
import com.example.assistant.tools.DatabaseToolRegistry;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.stereotype.Component;

import java.util.*;
import java.util.function.Consumer;

@Component
public class AssistantEngine {

    private static final Logger log = LoggerFactory.getLogger(AssistantEngine.class);
    private static final ObjectMapper mapper = new ObjectMapper();

    private final AnthropicClient anthropicClient;
    private final DatabaseToolRegistry toolRegistry;

    @Value("${anthropic.model}")
    private String model;

    @Value("${anthropic.max-tokens}")
    private int maxTokens;

    @Value("${anthropic.max-tool-iterations:8}")
    private int maxToolIterations;

    public AssistantEngine(AnthropicClient anthropicClient,
                           DatabaseToolRegistry toolRegistry) {
        this.anthropicClient = anthropicClient;
        this.toolRegistry = toolRegistry;
    }

    /**
     * Run agentic loop với conversation history.
     * @param messages Full conversation history (sẽ được mutate)
     * @param systemPrompt System prompt cho assistant
     * @return Final text response từ Claude
     */
    public String run(List<MessageParam> messages, String systemPrompt) {
        toolRegistry.clearExecutedQueries();

        int iteration = 0;
        while (iteration < maxToolIterations) {
            iteration++;
            log.debug("Iteration {}/{}, messages in history: {}",
                iteration, maxToolIterations, messages.size());

            Message response = callClaude(messages, systemPrompt);

            // Append Claude response to history
            messages.add(MessageParam.builder()
                .role(MessageParam.Role.ASSISTANT)
                .content(response.content())
                .build());

            String stopReason = response.stopReason().toString();

            if ("end_turn".equals(stopReason)) {
                return extractText(response);
            }

            if ("tool_use".equals(stopReason)) {
                List<ContentBlockParam> toolResults = processToolCalls(response);
                messages.add(MessageParam.builder()
                    .role(MessageParam.Role.USER)
                    .content(toolResults)
                    .build());
            } else {
                log.warn("Unexpected stop_reason: {}", stopReason);
                return extractText(response);
            }
        }

        return "Xin lỗi, tôi đã xử lý quá nhiều bước và không thể hoàn thành. " +
               "Vui lòng thử câu hỏi đơn giản hơn.";
    }

    /**
     * Run với streaming — gọi onToken callback mỗi khi nhận token text.
     * Tool use steps vẫn xử lý normally (không stream).
     */
    public String runStreaming(List<MessageParam> messages,
                               String systemPrompt,
                               Consumer<String> onToken) {
        // TODO: Implement streaming variant
        // Hint: Dùng client.messages().stream() cho final response
        //       Dùng client.messages().create() cho intermediate tool calls
        // Tool calls không cần stream — chỉ final answer cần stream
        return run(messages, systemPrompt); // Fallback to non-streaming for now
    }

    private Message callClaude(List<MessageParam> messages, String systemPrompt) {
        return anthropicClient.messages().create(
            MessageCreateParams.builder()
                .model(Model.valueOf(model.toUpperCase().replace("-", "_")))
                .maxTokens(maxTokens)
                .system(systemPrompt)
                .tools(toolRegistry.getAllTools())
                .messages(messages)
                .build()
        );
    }

    private List<ContentBlockParam> processToolCalls(Message response) {
        List<ContentBlockParam> results = new ArrayList<>();

        for (ContentBlock block : response.content()) {
            if (!(block instanceof ContentBlock.ToolUseBlock toolUse)) continue;

            String toolName = toolUse.name();
            String toolUseId = toolUse.id();
            log.info("Executing tool: {}", toolName);

            String resultContent;
            boolean isError = false;

            try {
                JsonNode input = mapper.readTree(toolUse.input().toString());
                resultContent = dispatchTool(toolName, input);
                log.debug("Tool {} succeeded", toolName);
            } catch (SecurityException e) {
                log.warn("Tool {} security violation: {}", toolName, e.getMessage());
                resultContent = mapper.createObjectNode()
                    .put("error", "security_violation")
                    .put("message", e.getMessage())
                    .toString();
                isError = true;
            } catch (Exception e) {
                log.error("Tool {} failed: {}", toolName, e.getMessage(), e);
                resultContent = mapper.createObjectNode()
                    .put("error", "execution_failed")
                    .put("message", e.getMessage())
                    .toString();
                isError = true;
            }

            ToolResultBlockParam.Builder builder = ToolResultBlockParam.builder()
                .toolUseId(toolUseId)
                .content(resultContent);
            if (isError) builder.isError(true);

            results.add(ContentBlockParam.ofToolResult(builder.build()));
        }

        return results;
    }

    private String dispatchTool(String toolName, JsonNode input) throws Exception {
        return switch (toolName) {
            case "list_tables"       -> toolRegistry.handleListTables(input);
            case "get_table_schema"  -> toolRegistry.handleGetTableSchema(input);
            case "execute_query"     -> toolRegistry.handleExecuteQuery(input);
            case "get_query_plan"    -> toolRegistry.handleGetQueryPlan(input);
            default -> throw new IllegalArgumentException("Unknown tool: " + toolName);
        };
    }

    private String extractText(Message message) {
        return message.content().stream()
            .filter(b -> b instanceof ContentBlock.TextBlock)
            .map(b -> ((ContentBlock.TextBlock) b).text())
            .findFirst()
            .orElse("");
    }
}
```

---

### Step 6: Main Service & System Prompt (45 phút)

```java
package com.example.assistant.service;

import com.example.assistant.dto.*;
import com.example.assistant.engine.AssistantEngine;
import com.example.assistant.session.*;
import com.example.assistant.tools.DatabaseToolRegistry;
import org.springframework.stereotype.Service;

@Service
public class DatabaseAssistantService {

    private final AssistantEngine engine;
    private final SessionStore sessionStore;
    private final DatabaseToolRegistry toolRegistry;

    // TODO: Inject prompt từ config hoặc file
    private static final String SYSTEM_PROMPT = """
        ## Role
        Bạn là AI Database Assistant cho hệ thống e-commerce.
        Bạn giúp users query và phân tích dữ liệu bằng natural language.

        ## Approach
        1. Khi bắt đầu conversation mới hoặc không chắc về schema:
           LUÔN gọi list_tables trước để hiểu cấu trúc database.
        2. Trước khi viết query cho một table lần đầu:
           Gọi get_table_schema để verify column names và types.
        3. Viết query chính xác, sau đó execute_query.
        4. Nếu cần nhiều queries, thực hiện tuần tự và tổng hợp kết quả.

        ## Response Style
        - Trả lời bằng ngôn ngữ user dùng (Vietnamese hoặc English)
        - Diễn giải kết quả thành ngôn ngữ tự nhiên, không phải raw data
        - Khi có số liệu, format rõ ràng: dùng dấu phẩy ngàn, ký hiệu tiền tệ
        - Nếu kết quả trống (0 rows), giải thích rõ ràng
        - Highlight insights thú vị nếu thấy trong data

        ## Constraints
        - Chỉ đọc data, không bao giờ modify
        - Nếu câu hỏi không liên quan đến database, lịch sự từ chối
        - Nếu không thể answer với data có sẵn, nói rõ lý do
        - Không expose raw SQL trong response (trừ khi user hỏi)
        """;

    public DatabaseAssistantService(AssistantEngine engine,
                                    SessionStore sessionStore,
                                    DatabaseToolRegistry toolRegistry) {
        this.engine = engine;
        this.sessionStore = sessionStore;
        this.toolRegistry = toolRegistry;
    }

    public AskResponse ask(AskRequest request) {
        long startTime = System.currentTimeMillis();

        // Get or create session
        ConversationSession session = sessionStore.getOrCreate(request.sessionId());

        // Add user message to history
        session.addMessage(MessageParam.builder()
            .role(MessageParam.Role.USER)
            .content(request.question())
            .build());

        // Get mutable copy of history for this request
        List<MessageParam> messages = new ArrayList<>(session.getHistory());
        // Remove last message — engine sẽ re-add sau khi process
        messages.remove(messages.size() - 1);
        messages.add(MessageParam.builder()
            .role(MessageParam.Role.USER)
            .content(request.question())
            .build());

        // Run agentic loop
        String answer = engine.run(messages, SYSTEM_PROMPT);

        // Save assistant response to session
        session.addMessage(MessageParam.builder()
            .role(MessageParam.Role.ASSISTANT)
            .content(answer)
            .build());

        long duration = System.currentTimeMillis() - startTime;

        return new AskResponse(
            session.getSessionId(),
            answer,
            toolRegistry.getExecutedQueries(),
            new TokenUsage(0, 0), // TODO: Track actual tokens from engine
            duration
        );
    }

    public void clearSession(String sessionId) {
        sessionStore.delete(sessionId);
    }
}
```

---

### Step 7: REST Controller (30 phút)

```java
package com.example.assistant.controller;

import com.example.assistant.dto.*;
import com.example.assistant.service.DatabaseAssistantService;
import jakarta.validation.Valid;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;

import java.util.Map;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

@RestController
@RequestMapping("/api/assistant")
public class AssistantController {

    private final DatabaseAssistantService assistantService;
    private final ExecutorService sseExecutor = Executors.newCachedThreadPool();

    public AssistantController(DatabaseAssistantService assistantService) {
        this.assistantService = assistantService;
    }

    /**
     * Standard (non-streaming) endpoint
     */
    @PostMapping("/ask")
    public ResponseEntity<AskResponse> ask(@Valid @RequestBody AskRequest request) {
        AskResponse response = assistantService.ask(request);
        return ResponseEntity.ok(response);
    }

    /**
     * Streaming endpoint via Server-Sent Events
     */
    @GetMapping(value = "/ask/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter askStream(
            @RequestParam String question,
            @RequestParam(required = false) String sessionId) {

        SseEmitter emitter = new SseEmitter(120_000L); // 2 min timeout

        sseExecutor.execute(() -> {
            try {
                // TODO: Implement streaming
                // 1. Gọi assistantService.askStreaming() với onToken callback
                // 2. Mỗi token: emitter.send(SseEmitter.event().data(token))
                // 3. Khi xong: emitter.send(SseEmitter.event().data("[DONE]"))
                // 4. emitter.complete()

                // Placeholder:
                AskResponse response = assistantService.ask(
                    new AskRequest(question, sessionId));

                // Send full response as single event
                emitter.send(SseEmitter.event()
                    .name("message")
                    .data(response.answer()));
                emitter.send(SseEmitter.event()
                    .name("done")
                    .data("[DONE]"));
                emitter.complete();

            } catch (Exception e) {
                try {
                    emitter.send(SseEmitter.event()
                        .name("error")
                        .data("Error: " + e.getMessage()));
                } catch (Exception ignored) {}
                emitter.completeWithError(e);
            }
        });

        return emitter;
    }

    /**
     * Clear session history
     */
    @DeleteMapping("/sessions/{sessionId}")
    public ResponseEntity<Map<String, String>> clearSession(
            @PathVariable String sessionId) {
        assistantService.clearSession(sessionId);
        return ResponseEntity.ok(Map.of(
            "message", "Session cleared",
            "sessionId", sessionId
        ));
    }

    /**
     * Health check
     */
    @GetMapping("/health")
    public ResponseEntity<Map<String, Object>> health() {
        // TODO: Check DB connectivity
        // TODO: Check Claude API reachability (optional)
        return ResponseEntity.ok(Map.of(
            "status", "UP",
            "database", "UP", // TODO: actual check
            "timestamp", System.currentTimeMillis()
        ));
    }
}
```

---

## Testing Checklist

### Unit Tests

```java
@SpringBootTest
class DatabaseToolRegistryTest {

    @Autowired
    private DatabaseToolRegistry registry;

    @Test
    void listTables_shouldReturnAllTables() throws Exception {
        // TODO: Verify returns customers, orders, order_items, products tables
    }

    @Test
    void executeQuery_shouldBlockInsertStatements() {
        // TODO: Verify SecurityException thrown for INSERT
        assertThrows(SecurityException.class, () ->
            registry.handleExecuteQuery(
                makeInput("INSERT INTO customers VALUES (1, 'test')", "insert test")));
    }

    @Test
    void executeQuery_shouldAddLimitIfMissing() throws Exception {
        // TODO: Verify LIMIT is added to query without LIMIT
    }

    @Test
    void executeQuery_shouldRespectExistingLimit() throws Exception {
        // TODO: Verify existing LIMIT is not replaced
    }
}
```

### Integration Tests — Manual

Sau khi start application, test với curl:

```bash
BASE="http://localhost:8080/api/assistant"

# Test 1: Basic query
echo "=== Test 1: Count customers ==="
curl -s -X POST "$BASE/ask" \
  -H "Content-Type: application/json" \
  -d '{"question": "How many customers do we have?"}' | jq .

# Test 2: Multi-turn với session
echo "=== Test 2: Multi-turn conversation ==="
SESSION_ID=$(curl -s -X POST "$BASE/ask" \
  -H "Content-Type: application/json" \
  -d '{"question": "What tables are available?"}' | jq -r '.sessionId')

echo "Session ID: $SESSION_ID"

curl -s -X POST "$BASE/ask" \
  -H "Content-Type: application/json" \
  -d "{\"question\": \"Now show me the orders table structure\", \"sessionId\": \"$SESSION_ID\"}" | jq .

# Test 3: Complex analytical query
echo "=== Test 3: Analytics ==="
curl -s -X POST "$BASE/ask" \
  -H "Content-Type: application/json" \
  -d '{"question": "Tháng này có bao nhiêu đơn hàng? Top 3 sản phẩm bán chạy?"}' | jq .

# Test 4: Security — should be blocked
echo "=== Test 4: Security test ==="
curl -s -X POST "$BASE/ask" \
  -H "Content-Type: application/json" \
  -d '{"question": "Delete all orders from the database"}' | jq .

# Test 5: Streaming
echo "=== Test 5: Streaming ==="
curl -N "$BASE/ask/stream?question=How+many+products+do+we+have"

# Test 6: Clear session
echo "=== Test 6: Clear session ==="
curl -s -X DELETE "$BASE/sessions/$SESSION_ID" | jq .
```

### Expected Test Results

| Test | Expected Behavior |
|------|------------------|
| Count customers | Trả về số customers chính xác (5 trong sample data) |
| Multi-turn | Lần 2 nhớ context từ lần 1 về tables |
| Analytics | Gọi ít nhất 2 queries, tổng hợp kết quả |
| Security | Claude từ chối modify data, giải thích lý do |
| Streaming | Tokens xuất hiện từ từ, kết thúc bằng [DONE] |
| Clear session | 200 OK, session deleted |

---

## Extension Ideas

Sau khi hoàn thành requirements cơ bản, thử thêm:

### Level 1: Polish
1. **Proper token tracking**: Accumulate tokens across all Claude calls trong một request
2. **Streaming implementation**: Hoàn thiện SSE streaming cho final answer
3. **Session cleanup**: `@Scheduled` job clear expired sessions mỗi 30 phút
4. **Input sanitization**: Strip SQL injection attempts từ natural language question

### Level 2: Features
5. **Query caching**: Cache results của identical queries trong 5 phút (Caffeine/Redis)
6. **Export to CSV**: Tool mới cho phép export query results thành CSV file
7. **Query history**: Lưu lịch sử queries của mỗi session, cho phép re-run
8. **Explain mode**: `?explain=true` parameter → Claude giải thích SQL query nó viết

### Level 3: Production-ready
9. **Redis session store**: Thay ConcurrentHashMap bằng Redis để support multiple instances
10. **Rate limiting**: Dùng Bucket4j hoặc Redis để limit requests per session
11. **Metrics**: Micrometer metrics cho token usage, query count, latency
12. **Auth**: JWT authentication, user-specific sessions
13. **Multi-database**: Support MySQL/SQLite bên cạnh PostgreSQL

---

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| F1-F3 Functional | 30 | POST ask, natural language answer, multi-step |
| F4 Streaming | 15 | SSE endpoint hoạt động |
| F5 Sessions | 15 | Multi-turn conversation với history |
| F6 Security | 20 | SELECT only, input validation, error handling |
| Code Quality | 10 | Clean code, proper logging, error messages |
| Tests | 10 | Ít nhất 5 unit tests pass |
| **Total** | **100** | Pass threshold: 70 |

---

## Submission

1. Push code lên GitHub repository
2. Include `README.md` với:
   - Setup instructions (database, env vars)
   - API documentation (request/response examples)
   - Architecture notes (decisions bạn đã làm)
3. Demo video (5 phút): chạy ít nhất 3 test queries khác nhau

---

*Hoàn thành project này, bạn đã xây dựng một production-grade AI application tổng hợp: REST API + streaming + sessions + tool use + prompt engineering. Đây là foundation cho Module 02 — Agentic Systems.*
