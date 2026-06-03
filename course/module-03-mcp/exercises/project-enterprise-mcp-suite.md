# Capstone Project: JavaOps MCP Suite

> **Module**: 03 — Model Context Protocol  
> **Type**: Capstone Project (tuần 3)  
> **Thời lượng**: 8-12 giờ implementation  
> **Deliverable**: Production-ready MCP server deploy được cho team

---

## Tổng quan

**JavaOps MCP Suite** là một Spring Boot application expose 7 MCP tools để Claude Code có thể interact trực tiếp với Spring Boot application internals — từ bean graph, database schema, migration status, đến log analysis và code search.

Đây không phải là toy project. Mục tiêu là build một server bạn **thực sự dùng** trong daily development workflow.

---

## Kiến trúc tổng thể

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Claude Code                                   │
│                    (Developer's machine)                             │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ JSON-RPC over stdio
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   JavaOps MCP Suite                                  │
│               (Spring Boot, runs locally)                            │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Tool Registry                             │   │
│  │  get_beans │ check_migrations │ list_endpoints │ analyze_logs│   │
│  │  get_db_schema │ run_health_check │ search_code             │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                           │                                         │
│          ┌────────────────┼────────────────┐                        │
│          ▼                ▼                ▼                        │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐               │
│  │ ActuatorClient│ │  JdbcTemplate│ │  FileSystem  │               │
│  │ (HTTP calls) │ │  (DB queries)│ │  (Log files) │               │
│  └──────┬───────┘ └──────┬───────┘ └──────┬───────┘               │
└─────────┼────────────────┼────────────────┼───────────────────────-┘
          │                │                │
          ▼                ▼                ▼
┌──────────────────┐ ┌──────────────┐ ┌──────────────────────────────┐
│  Your Spring Boot│ │  PostgreSQL  │ │  Project Source Code         │
│  Application     │ │  Database    │ │  (src/, pom.xml, etc.)       │
│  :8080/actuator  │ │  (dev env)   │ │                              │
└──────────────────┘ └──────────────┘ └──────────────────────────────┘
```

**Điểm quan trọng về kiến trúc**:

- JavaOps MCP Suite là **separate process** — không phải library trong app của bạn
- Nó connect tới running application qua Actuator HTTP endpoints
- Nó connect tới database qua JDBC (read-only user)
- Nó đọc source files qua filesystem
- Claude Code spawn nó như subprocess, communicate qua stdin/stdout

---

## 7 Tools — Specification Chi Tiết

### Tool 1: `get_beans`

**Mục đích**: Cho Claude thấy toàn bộ Spring Application Context — beans nào đang chạy, type của chúng, và dependency graph.

**Input**: Không có (hoặc optional filter)

```java
@Tool(description = """
    List all Spring beans in the application context.
    Returns: bean name, class type, scope, and direct dependencies.
    Optionally filter by package prefix or bean name pattern.
    Use this to: understand application structure, find missing beans,
    debug autowiring issues, check if a component is registered.
    """)
public String getBeans(
    @ToolParam(description = "Optional: filter beans by package prefix, e.g. 'com.mycompany'. " +
                             "Leave null to get all beans.")
    String packageFilter
)
```

**Output format**:
```
Spring Application Context — 247 beans total
Filtered by: com.mycompany (showing 34 beans)

BEAN: orderService
  Class: com.mycompany.order.OrderService
  Scope: singleton
  Dependencies:
    - orderRepository (com.mycompany.order.OrderRepository)
    - paymentService (com.mycompany.payment.PaymentService)
    - eventPublisher (org.springframework.context.ApplicationEventPublisher)

BEAN: orderRepository
  Class: com.mycompany.order.OrderRepository
  Scope: singleton
  Dependencies:
    - entityManagerFactory
    - transactionManager
...
```

**Implementation approach**:
- Call `/actuator/beans` endpoint
- Parse JSON, apply package filter nếu có
- Format thành human-readable text
- Sort beans alphabetically

---

### Tool 2: `check_migrations`

**Mục đích**: Xem trạng thái Flyway migrations — version nào đã apply, cái nào fail, có pending migrations không.

**Input**: Không có parameters

```java
@Tool(description = """
    Check Flyway database migration status.
    Returns: all migrations with version, description, execution status,
    checksum, and installation timestamp.
    Status values: SUCCESS, FAILED, PENDING, IGNORED, MISSING.
    Use this to: verify deployment ran migrations correctly,
    find failed migrations, check migration history.
    """)
public String checkMigrations()
```

**Output format**:
```
Flyway Migration Status
=======================
Schema version: 24
Total migrations: 24 (23 applied, 0 failed, 1 pending)

Applied migrations:
  [SUCCESS] V1__create_users_table.sql
            Applied: 2025-01-15 09:23:11
            Duration: 45ms

  [SUCCESS] V2__add_email_index.sql
            Applied: 2025-01-15 09:23:12
            Duration: 120ms

  ...

  [SUCCESS] V23__add_payment_intent_id.sql
            Applied: 2026-05-28 14:35:02
            Duration: 233ms

Pending migrations:
  [PENDING] V24__add_refund_status_column.sql
            Found in: classpath:db/migration/

⚠ Warning: 1 pending migration found. Run application to apply.
```

**Implementation approach**:
- Call `/actuator/flyway`
- Nếu không available: query `flyway_schema_history` table trực tiếp qua JDBC
- Highlight FAILED và PENDING migrations
- Show summary counts

---

### Tool 3: `list_endpoints`

**Mục đích**: List tất cả REST endpoints với HTTP method, URL pattern, handler class, và authentication requirements.

**Input**: Optional filter

```java
@Tool(description = """
    List all REST API endpoints in the application.
    Returns: HTTP method, URL pattern, handler class, handler method,
    and any security annotations (@PreAuthorize, @Secured).
    Optionally filter by HTTP method or URL prefix.
    Use this to: find endpoints, check API coverage, 
    audit security annotations, generate API documentation.
    """)
public String listEndpoints(
    @ToolParam(description = "Optional: filter by HTTP method (GET, POST, PUT, DELETE, PATCH). " +
                             "Leave null for all methods.")
    String httpMethod,
    
    @ToolParam(description = "Optional: filter by URL prefix, e.g. '/api/orders'. " +
                             "Leave null for all URLs.")
    String urlPrefix
)
```

**Output format**:
```
REST API Endpoints — 47 total (showing 12 matching filter)

GET     /api/users                    UserController.getUsers()
GET     /api/users/{id}               UserController.getUserById()
POST    /api/users                    UserController.createUser()
        @PreAuthorize("hasRole('ADMIN')")
PUT     /api/users/{id}               UserController.updateUser()
DELETE  /api/users/{id}               UserController.deleteUser()
        @PreAuthorize("hasRole('ADMIN')")

GET     /api/orders                   OrderController.getOrders()
GET     /api/orders/{id}              OrderController.getOrderById()
POST    /api/orders                   OrderController.createOrder()
PUT     /api/orders/{id}/cancel       OrderController.cancelOrder()

[Actuator endpoints: 15 (excluded from list)]
```

**Implementation approach**:
- Call `/actuator/mappings`
- Parse `dispatcherServlets` section
- Filter out actuator endpoints (chứa `/actuator/`)
- Apply HTTP method và URL prefix filter nếu có
- Detect security annotations từ handler method reflection (nếu accessible)

---

### Tool 4: `analyze_logs`

**Mục đích**: Parse application logs và tìm patterns, errors, hay specific events.

**Input**: Search parameters

```java
@Tool(description = """
    Analyze application log file. Search for patterns, errors, or specific events.
    Can filter by: log level, time range, search term, logger name.
    Returns matching log lines with context (surrounding lines).
    
    Use this to: investigate errors, trace request flows, 
    find performance issues, understand what happened at a specific time.
    """)
public String analyzeLogs(
    @ToolParam(description = "Log level filter: ERROR, WARN, INFO, DEBUG. Null for all levels.")
    String level,
    
    @ToolParam(description = "Search term to look for in log messages. " +
                             "Case-insensitive substring match. Null to skip text search.")
    String searchTerm,
    
    @ToolParam(description = "Number of lines of context to include around each match. " +
                             "0 = match only, 2 = 2 lines before and after. Max: 5.")
    int contextLines,
    
    @ToolParam(description = "Maximum number of matching entries to return. Max: 200.")
    int maxResults
)
```

**Output format**:
```
Log Analysis Results
====================
Filters: level=ERROR, searchTerm="payment", contextLines=2, maxResults=50
Found: 7 matching entries (showing all 7)

---
2026-06-03 14:23:45.123 INFO  [http-nio-8080-exec-3] PaymentService - Processing payment for order #10234
2026-06-03 14:23:45.234 INFO  [http-nio-8080-exec-3] StripeClient - Calling Stripe charge API
2026-06-03 14:23:45.891 ERROR [http-nio-8080-exec-3] PaymentService - Payment failed for order #10234
                               com.stripe.exception.CardException: Your card has insufficient funds.
                               at com.example.payment.StripeClient.charge(StripeClient.java:87)
2026-06-03 14:23:45.892 WARN  [http-nio-8080-exec-3] OrderService - Reverting order #10234 to PENDING

---
[6 more entries...]
```

**Implementation approach**:
- Đọc log file được config qua `logging.file.name` hoặc default location
- Parse log lines theo Spring Boot default format: `TIMESTAMP LEVEL [THREAD] LOGGER - MESSAGE`
- Filter theo level, searchTerm
- Collect context lines (N lines before/after match)
- Return formatted results với separator giữa các matches

---

### Tool 5: `get_db_schema`

**Mục đích**: Full database schema với tables, columns, constraints, indexes, và foreign keys.

**Input**: Optional table filter

```java
@Tool(description = """
    Get complete database schema description.
    Returns: all tables with columns (name, type, nullable, default),
    primary keys, foreign keys, unique constraints, and indexes.
    Optionally filter to specific tables.
    
    Use this to: understand data model, write correct SQL queries,
    find foreign key relationships, check index coverage.
    Always call this before writing complex SQL queries.
    """)
public String getDbSchema(
    @ToolParam(description = "Optional: comma-separated list of table names to describe. " +
                             "E.g. 'users,orders,order_items'. Null for all tables.")
    String tableFilter
)
```

**Output format**:
```
Database Schema — PostgreSQL
============================
Tables: 12 | Views: 3 | Total columns: 87

TABLE: users
  Columns:
    id               BIGSERIAL        NOT NULL  PK
    email            VARCHAR(255)     NOT NULL  UNIQUE
    name             VARCHAR(255)     NOT NULL
    created_at       TIMESTAMPTZ      NOT NULL  DEFAULT now()
    plan_id          BIGINT           NULL      FK→plans.id
    deleted_at       TIMESTAMPTZ      NULL

  Indexes:
    users_pkey           UNIQUE  (id)
    users_email_key      UNIQUE  (email)
    idx_users_created    btree   (created_at DESC)

  Foreign Keys:
    users_plan_id_fkey: plan_id → plans(id) ON DELETE SET NULL

TABLE: orders
  Columns:
    id               BIGSERIAL        NOT NULL  PK
    user_id          BIGINT           NOT NULL  FK→users.id
    status           VARCHAR(50)      NOT NULL  DEFAULT 'PENDING'
    total            NUMERIC(10,2)    NOT NULL
    created_at       TIMESTAMPTZ      NOT NULL  DEFAULT now()
  ...
```

**Implementation approach**:
- Query `information_schema.tables`, `information_schema.columns`
- Query `information_schema.table_constraints`, `information_schema.key_column_usage`
- Query `pg_indexes` cho index information
- Apply table filter nếu có
- Format với clear visual hierarchy

---

### Tool 6: `run_health_check`

**Mục đích**: Comprehensive health check với actionable insights, không chỉ UP/DOWN.

**Input**: Optional specific component

```java
@Tool(description = """
    Run comprehensive application health check.
    Returns: overall status and per-component details including
    database connectivity, disk space, external service dependencies,
    connection pool status, cache health.
    Includes actionable recommendations for any unhealthy components.
    
    Use this to: check deployment health, investigate slowdowns,
    verify all dependencies are operational, monitor resource usage.
    """)
public String runHealthCheck(
    @ToolParam(description = "Optional: specific component to check, e.g. 'db', 'diskSpace', 'redis'. " +
                             "Null for full health check.")
    String component
)
```

**Output format**:
```
Application Health Check
========================
Overall Status: DEGRADED ⚠
Timestamp: 2026-06-03T14:30:00Z

Components:
  ✓ db             UP      Response: 3ms
                           Database: PostgreSQL 15.2
                           
  ✓ diskSpace      UP      Free: 45.2 GB / 100 GB (45.2% free)
  
  ⚠ redis          DOWN    Error: Connection refused to localhost:6379
                           Impact: Session caching disabled, 
                           → All requests hitting database
                           Recommendation: Start Redis or check connection config
                           
  ✓ ping           UP
  
  ✓ livenessState  CORRECT
  ⚠ readinessState ACCEPTING_TRAFFIC  (degraded due to redis)

HikariCP Connection Pool:
  Active: 8 / 20 (40%)
  Idle: 12
  Awaiting connection: 0
  Total acquired: 15,234
  
JVM Memory:
  Heap used: 312 MB / 512 MB (60.9%)
  Non-heap: 87 MB
  GC pauses (last 5min): 3 pauses, total 45ms

Recommendations:
  1. [HIGH] Redis is down — investigate and restart
  2. [LOW] Heap usage at 60%, monitor if workload increases
```

**Implementation approach**:
- Call `/actuator/health` với full detail
- Call `/actuator/metrics/hikaricp.connections.*` cho connection pool stats
- Call `/actuator/metrics/jvm.memory.*` cho memory stats
- Call `/actuator/metrics/jvm.gc.*` cho GC stats
- Synthesize thành comprehensive report
- Add actionable recommendations dựa trên findings

---

### Tool 7: `search_code`

**Mục đích**: Tìm kiếm trong source code theo text, annotation, class name, hoặc method name.

**Input**: Search parameters

```java
@Tool(description = """
    Search project source code for patterns, class names, method names, or annotations.
    Searches Java source files, configuration files, and SQL migrations.
    Returns: file path, line number, matching line, and surrounding context.
    
    Use this to: find where a class is used, locate specific annotations,
    find all places a method is called, search for TODO/FIXME comments,
    find configuration properties usage.
    """)
public String searchCode(
    @ToolParam(description = "Search term: text, class name, annotation (@Transactional), " +
                             "or method name. Case-insensitive.")
    String searchTerm,
    
    @ToolParam(description = "File type filter: 'java', 'sql', 'yml', 'xml', or 'all'. Default: 'java'")
    String fileType,
    
    @ToolParam(description = "Maximum number of matches to return. Max: 50.")
    int maxResults
)
```

**Output format**:
```
Code Search Results
===================
Query: "@Transactional" | File type: java | Max: 50
Found: 23 matches in 8 files

src/main/java/com/example/order/OrderService.java:45
  @Service
  public class OrderService {
→     @Transactional
      public Order createOrder(CreateOrderRequest request) {

src/main/java/com/example/order/OrderService.java:89
  
→     @Transactional(readOnly = true)
      public Page<Order> findOrders(Pageable pageable) {

src/main/java/com/example/payment/PaymentService.java:34
  @Service
  public class PaymentService {
→     @Transactional(rollbackFor = PaymentException.class)
      public PaymentResult processPayment(PaymentRequest request) {

[20 more matches in 6 files...]
```

**Implementation approach**:
- Đọc source root từ config (`app.source.root` property hoặc scan current directory)
- Walk file tree, filter theo `fileType`
- Search từng file theo `searchTerm` (case-insensitive)
- Collect matches với 2 lines context (before + after)
- Return với file path và line numbers
- Respect `maxResults` limit

---

## Project Structure

```
javaops-mcp-suite/
├── pom.xml
├── README.md
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/example/javaops/
│   │   │       ├── JavaOpsMcpApplication.java
│   │   │       ├── config/
│   │   │       │   ├── AppConfig.java          (RestTemplate, ObjectMapper beans)
│   │   │       │   └── SecurityConfig.java     (chỉ cho stdio, không expose HTTP)
│   │   │       ├── client/
│   │   │       │   └── ActuatorClient.java     (HTTP calls tới target app)
│   │   │       ├── tools/
│   │   │       │   ├── ActuatorTools.java      (get_beans, check_migrations, list_endpoints, run_health_check)
│   │   │       │   ├── DatabaseTools.java      (get_db_schema)
│   │   │       │   ├── LogAnalysisTools.java   (analyze_logs)
│   │   │       │   └── CodeSearchTools.java    (search_code)
│   │   │       ├── resources/
│   │   │       │   └── ApplicationResources.java (app://info, app://config)
│   │   │       └── prompts/
│   │   │           └── ApplicationPrompts.java   (analyze-issue, review-migration)
│   │   └── resources/
│   │       └── application.yml
│   └── test/
│       └── java/
│           └── com/example/javaops/
│               ├── tools/
│               │   ├── ActuatorToolsTest.java
│               │   ├── DatabaseToolsTest.java
│               │   ├── DatabaseToolsSecurityTest.java
│               │   ├── LogAnalysisToolsTest.java
│               │   └── CodeSearchToolsTest.java
│               └── integration/
│                   └── McpServerIntegrationTest.java
```

---

## Implementation Steps

### Step 1: Project Setup (30 phút)

```bash
# Tạo project từ Spring Initializr
curl https://start.spring.io/starter.zip \
  -d type=maven-project \
  -d language=java \
  -d bootVersion=3.3.0 \
  -d baseDir=javaops-mcp-suite \
  -d groupId=com.example \
  -d artifactId=javaops-mcp-suite \
  -d name=JavaOpsMcpSuite \
  -d dependencies=web,actuator,jdbc \
  -o javaops-mcp-suite.zip

unzip javaops-mcp-suite.zip && cd javaops-mcp-suite
```

Thêm Spring AI MCP dependency vào `pom.xml` (xem Bài 03).

Verify build:
```bash
mvn clean compile
```

### Step 2: Application Configuration (15 phút)

Tạo `application.yml` với:
- `spring.ai.mcp.server.transport: stdio`
- Actuator base URL config
- Database connection
- Logging ra file (không ra stdout)

### Step 3: ActuatorClient (45 phút)

Implement `ActuatorClient` với methods:
- `getBeans(String packageFilter)`
- `getFlywayStatus()`
- `getMappings(String httpMethod, String urlPrefix)`
- `getHealth(String component)`
- `getMetrics(String metricName)`
- `getLogfile()`

Viết unit tests với MockRestServiceServer.

### Step 4: ActuatorTools (1 giờ)

Implement 4 tools:
- `get_beans`
- `check_migrations`
- `list_endpoints`
- `run_health_check`

Mỗi tool cần:
- Proper `@Tool` description
- Input validation
- Error handling (graceful khi actuator không available)
- Audit logging

Viết unit tests.

### Step 5: DatabaseTools (45 phút)

Implement `get_db_schema` với full schema introspection.

**Security checklist**:
- [ ] Chỉ read từ `information_schema` và `pg_catalog`
- [ ] Không expose passwords hay connection strings
- [ ] Validate table name filter (chỉ alphanumeric + underscore)

Viết security tests.

### Step 6: LogAnalysisTools (45 phút)

Implement `analyze_logs`:
- Đọc log file path từ `logging.file.name` property
- Parse Spring Boot log format với regex
- Implement level filter
- Implement text search với context lines
- Respect maxResults limit

Viết tests với sample log files.

### Step 7: CodeSearchTools (1 giờ)

Implement `search_code`:
- Walk file tree từ configured source root
- Filter theo file extension
- Case-insensitive text search
- Collect 2 lines context
- Format với file path + line numbers

Edge cases cần handle:
- Binary files (skip)
- Very large files (stream, không load toàn bộ vào memory)
- Files với encoding issues
- Symlinks (follow hoặc skip — document decision)

Viết tests với test fixtures.

### Step 8: Resources và Prompts (30 phút)

Implement:
- `app://info` resource
- `app://config` resource (safe keys only)
- `analyze-spring-issue` prompt
- `review-migration` prompt

### Step 9: Integration Testing (1 giờ)

Test full MCP flow với MCP Inspector:

```bash
# Build JAR
mvn clean package

# Start your Spring Boot app (target app) on port 8080
# Then run MCP Inspector với JavaOps MCP Suite
mcp-inspector java -jar target/javaops-mcp-suite-1.0.0.jar
```

Test checklist:
- [ ] `/mcp` shows 7 tools connected
- [ ] `get_beans` returns bean list
- [ ] `check_migrations` returns migration history
- [ ] `list_endpoints` returns API endpoints
- [ ] `analyze_logs level=ERROR count=20` returns log entries
- [ ] `get_db_schema` returns full schema
- [ ] `run_health_check` returns health status
- [ ] `search_code term=@Transactional` returns code matches
- [ ] `get_db_schema` với invalid table name — returns error (not exception)
- [ ] `analyze_logs level=INVALID` — returns validation error

### Step 10: Real-world Validation (1 giờ)

Config server vào một Spring Boot project thực, chạy Claude Code session và:

1. "Describe architecture của application này dựa trên Spring beans"
2. "Có migration nào pending không? Show migration history"
3. "List tất cả POST endpoints và check xem chúng có security annotations không"
4. "Có ERROR logs trong 30 phút qua không? Nếu có, summarize"
5. "Generate ERD description từ database schema"
6. "Tìm tất cả nơi dùng @Transactional với readOnly=true"
7. "Application có healthy không? Có vấn đề gì cần attention không?"

Document kết quả — có tool nào không work như expect không? Fix.

---

## Testing Guide

### Unit Test Coverage Requirements

```
ActuatorTools:          ≥ 80% line coverage
DatabaseTools:          ≥ 85% line coverage (security critical)
LogAnalysisTools:       ≥ 75% line coverage
CodeSearchTools:        ≥ 75% line coverage
ActuatorClient:         ≥ 70% line coverage (mock HTTP)
```

### Mandatory Security Tests

```java
// DatabaseTools — SQL injection prevention
@Test void rejectsNonSelectQuery()        // DROP TABLE
@Test void rejectsDeleteQuery()           // DELETE FROM
@Test void rejectsInsertQuery()           // INSERT INTO
@Test void rejectsDropInSubquery()        // SELECT 1; DROP TABLE
@Test void rejectsUnionBasedInjection()   // SELECT ... UNION SELECT ...

// Input validation
@Test void rejectsTableNameWithSpecialChars()  // ; DROP TABLE--
@Test void rejectsExcessivelyLongInput()
@Test void handlesNullInputGracefully()
@Test void handlesBlankInputGracefully()

// Log analysis
@Test void rejectsInvalidLogLevel()
@Test void rejectsExcessiveLineCount()
@Test void rejectsExcessiveContextLines()
```

### Integration Test với Testcontainers

```java
@SpringBootTest
@Testcontainers
class McpServerIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @Container
    static GenericContainer<?> targetApp = new GenericContainer<>("your-spring-app:latest")
        .withExposedPorts(8080);
    
    @Test
    void getDbSchema_returnsSchemaForTestDatabase() {
        // ...
    }
    
    @Test
    void checkMigrations_returnsAppliedMigrations() {
        // ...
    }
}
```

---

## Deployment Instructions

### Option A: Developer Local Deployment

```bash
# 1. Build
mvn clean package -DskipTests

# 2. Copy to tools directory
mkdir -p ~/.local/share/mcp-servers
cp target/javaops-mcp-suite-1.0.0.jar ~/.local/share/mcp-servers/

# 3. Add to project's .claude/settings.json
cat > .claude/settings.json << 'EOF'
{
  "mcpServers": {
    "javaops": {
      "command": "java",
      "args": [
        "-jar",
        "/Users/YOURNAME/.local/share/mcp-servers/javaops-mcp-suite-1.0.0.jar"
      ],
      "env": {
        "DATABASE_URL": "postgresql://mcp_readonly:password@localhost:5432/myapp_dev",
        "ACTUATOR_BASE_URL": "http://localhost:8080/actuator",
        "APP_SOURCE_ROOT": "/Users/YOURNAME/projects/my-spring-app/src"
      }
    }
  }
}
EOF

# 4. Verify
claude
/mcp
# Should show: javaops ✓ Connected (7 tools available)
```

### Option B: Team Shared Deployment

Cho team sharing — deploy như SSE server:

```yaml
# application-sse.yml — override cho SSE transport
spring:
  ai:
    mcp:
      server:
        transport: sse
        port: 8090

server:
  port: 8090
```

```bash
# Deploy trên team server
java -jar javaops-mcp-suite-1.0.0.jar --spring.profiles.active=sse &

# Team members config:
{
  "mcpServers": {
    "javaops": {
      "url": "http://mcp-server.internal:8090/sse",
      "headers": {
        "X-Team-Token": "${TEAM_MCP_TOKEN}"
      }
    }
  }
}
```

### Option C: Docker Deployment

```dockerfile
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

COPY target/javaops-mcp-suite-1.0.0.jar app.jar

# Create non-root user
RUN addgroup -S mcpserver && adduser -S mcpserver -G mcpserver
USER mcpserver

ENTRYPOINT ["java", \
  "-Xmx256m", \
  "-jar", \
  "app.jar"]
```

```bash
# Build image
docker build -t javaops-mcp-suite:1.0.0 .

# Run
docker run -d \
  -e DATABASE_URL="postgresql://mcp_readonly:pass@host.docker.internal:5432/myapp_dev" \
  -e ACTUATOR_BASE_URL="http://host.docker.internal:8080/actuator" \
  -p 8090:8090 \
  javaops-mcp-suite:1.0.0
```

---

## Evaluation Rubric

### Functional Completeness (40 điểm)

| Criteria | Điểm |
|----------|------|
| Tất cả 7 tools implement và hoạt động | 20 |
| Tools trả về đúng format như spec | 10 |
| Resources (app://info, app://config) hoạt động | 5 |
| Prompts (2 templates) hoạt động | 5 |

### Code Quality (25 điểm)

| Criteria | Điểm |
|----------|------|
| Test coverage ≥ 80% overall | 10 |
| Tất cả security tests pass | 10 |
| Code follows immutability principles | 5 |

### Security (20 điểm)

| Criteria | Điểm |
|----------|------|
| SQL injection prevention | 8 |
| Input validation tất cả tools | 7 |
| Sensitive data không exposed | 5 |

### Usability (15 điểm)

| Criteria | Điểm |
|----------|------|
| Error messages rõ ràng và actionable | 5 |
| Tool descriptions đủ tốt để Claude dùng đúng | 5 |
| README với setup instructions | 5 |

### Bonus (10 điểm)

| Criteria | Điểm |
|----------|------|
| Docker deployment option | 3 |
| Team SSE deployment | 3 |
| Testcontainers integration tests | 4 |

---

## Submission

Khi hoàn thành, nộp:

1. **GitHub repository** với full source code
2. **Demo video** (5-10 phút): Chạy Claude Code với JavaOps MCP Suite, thực hiện ít nhất 5 trong 7 tools, giải thích từng tool
3. **REFLECTION.md**: Những gì học được, khó khăn gặp phải, cách sử dụng trong real project

---

## Tips từ Kinh Nghiệm Thực Tế

**Về stdio transport**: Đây là lỗi phổ biến nhất. Bất kỳ output nào ra stdout sẽ corrupt JSON-RPC communication. Kiểm tra kỹ: Spring Boot startup banner, System.out.println trong code, thư viện nào đó log ra stdout. Tắt hết, log tất cả vào file.

**Về tool descriptions**: Claude dùng description để quyết định khi nào gọi tool và với arguments gì. Một description tốt là: (1) rõ tool làm gì, (2) output format là gì, (3) khi nào nên dùng, (4) limitations là gì. Đầu tư thời gian vào đây.

**Về error handling**: Mỗi tool nên return error message hữu ích thay vì throw exception. Exception sẽ crash MCP session. Return "Error: PostgreSQL not available at localhost:5432. Is the database running?" thay vì stacktrace.

**Về performance**: MCP tool calls có timeout. Tránh queries không có LIMIT, tránh đọc toàn bộ log file vào memory. Design cho "fast enough for interactive use" — mục tiêu dưới 2 giây cho mỗi tool call.

**Về real-world testing**: Test với production-like data volume. Một schema với 200 tables sẽ expose performance issues mà test với 5 tables không thấy.

---

*Capstone Project — Module 03: Model Context Protocol*  
*AI Engineering for Java Backend Engineers*
