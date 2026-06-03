# Project 01: JavaOps MCP Suite ⭐

> **Tên project:** JavaOps MCP Suite
> **Tagline:** Connect Claude to any Spring Boot application internals
> **Độ khó:** Intermediate
> **Thời gian build:** 2-3 ngày
> **Stack:** Java 21, Spring Boot 3.x, MCP Java SDK
> **Portfolio value:** ⭐⭐⭐⭐⭐ — Showcase piece

---

## Tại sao đây là portfolio piece mạnh nhất?

Trước khi bắt đầu code, hãy hiểu **tại sao** project này đặc biệt quan trọng:

**1. Java-native differentiator**
Hầu hết MCP servers trong ecosystem được viết bằng Python hoặc TypeScript. Một MCP server viết bằng Java, tích hợp Spring Boot — đây là thứ mà 45% enterprise developers (những người dùng Java) đang cần và chưa có ai làm tốt.

**2. Real enterprise use case**
Mỗi công ty chạy Spring Boot đều có vấn đề: Claude Code không biết gì về application của bạn. JavaOps MCP Suite giải quyết đúng pain point đó — Claude có thể introspect trực tiếp vào running application.

**3. MCP là hot skill**
Anthropic's MCP ecosystem đang tăng trưởng từ 0 lên 10,000+ servers trong ~18 tháng. Engineers hiểu MCP protocol và biết build MCP servers đang được tuyển dụng tích cực.

**4. Demonstrable in 5 minutes**
Demo script rõ ràng: "Ask Claude to diagnose a production issue using only this MCP server." Hiring manager xem demo 5 phút là hiểu ngay giá trị.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Developer Workflow                    │
└─────────────────────────┬───────────────────────────────┘
                          │ asks question
                          ▼
┌─────────────────────────────────────────────────────────┐
│                    Claude Code / Claude                  │
│              (AI assistant với MCP support)             │
└─────────────────────────┬───────────────────────────────┘
                          │ MCP protocol calls
                          ▼
┌─────────────────────────────────────────────────────────┐
│              JavaOps MCP Server (Java 21)                │
│                                                         │
│  Tools:              Resources:                         │
│  ├── get_beans       ├── db://schema                   │
│  ├── check_migrations├── logs://recent                  │
│  ├── list_endpoints  │                                  │
│  ├── analyze_logs    │                                  │
│  ├── get_db_schema   │                                  │
│  ├── run_health_check│                                  │
│  └── search_errors   │                                  │
└─────────────────────────┬───────────────────────────────┘
                          │ HTTP / JMX / JDBC
                          ▼
┌─────────────────────────────────────────────────────────┐
│              Target Spring Boot Application             │
│                                                         │
│  ├── Spring Actuator endpoints                         │
│  ├── Flyway migration registry                         │
│  ├── PostgreSQL database                               │
│  └── Application logs (Logback/Log4j2)                 │
└─────────────────────────────────────────────────────────┘
```

**Flow khi dùng:**
1. Developer mở Claude Code, configure JavaOps MCP Server
2. Developer hỏi: "Why is the /api/orders endpoint slow?"
3. Claude calls `list_endpoints` → thấy `/api/orders` mapping
4. Claude calls `analyze_logs` với filter "orders" → thấy N+1 query warnings
5. Claude calls `get_db_schema` table "orders" → thấy missing index
6. Claude trả lời: "N+1 query problem ở Orders.getItems(), thiếu index trên `customer_id`. Đây là fix..."

---

## Maven Dependencies

```xml
<!-- pom.xml -->
<project>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.0</version>
    </parent>

    <properties>
        <java.version>21</java.version>
        <mcp.version>0.9.0</mcp.version>
    </properties>

    <dependencies>
        <!-- Spring Boot Web (cho MCP HTTP transport) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Actuator client — gọi target app's actuator endpoints -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- MCP Java SDK -->
        <dependency>
            <groupId>io.modelcontextprotocol.sdk</groupId>
            <artifactId>mcp</artifactId>
            <version>${mcp.version}</version>
        </dependency>

        <!-- Spring AI MCP Server starter (integrates Spring + MCP) -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-starter-mcp-server-webmvc</artifactId>
        </dependency>

        <!-- JDBC để query target database -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
        </dependency>

        <!-- HTTP client để gọi actuator -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-webflux</artifactId>
        </dependency>

        <!-- JSON processing -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
    </dependencies>
</project>
```

---

## application.yml Configuration

```yaml
# application.yml
spring:
  application:
    name: javaops-mcp-server

  # MCP Server configuration
  ai:
    mcp:
      server:
        name: "JavaOps MCP Suite"
        version: "1.0.0"
        transport: STDIO   # Claude Code dùng STDIO transport

  # Database connection tới target app's database
  datasource:
    url: ${TARGET_DB_URL:jdbc:postgresql://localhost:5432/myapp}
    username: ${TARGET_DB_USER:postgres}
    password: ${TARGET_DB_PASSWORD:}
    driver-class-name: org.postgresql.Driver

javaops:
  # URL của Spring Boot app muốn introspect
  target-app-url: ${TARGET_APP_URL:http://localhost:8080}
  # Path tới log file của target app
  log-file-path: ${TARGET_LOG_PATH:/var/log/app/application.log}
  # Max lines trả về khi analyze logs
  max-log-lines: 500

server:
  port: 3001   # MCP server port (cho HTTP transport nếu cần)
```

---

## Step-by-Step Implementation

### Step 1: Project Structure

```
javaops-mcp-suite/
├── pom.xml
├── src/main/java/com/javaops/mcp/
│   ├── JavaOpsMcpApplication.java
│   ├── config/
│   │   └── McpServerConfig.java
│   ├── tools/
│   │   ├── BeanInspectorTool.java
│   │   ├── MigrationCheckerTool.java
│   │   ├── EndpointListerTool.java
│   │   ├── LogAnalyzerTool.java
│   │   ├── DatabaseSchemaTool.java
│   │   ├── HealthCheckTool.java
│   │   └── ErrorSearchTool.java
│   ├── resources/
│   │   ├── DatabaseSchemaResource.java
│   │   └── RecentLogsResource.java
│   └── client/
│       └── ActuatorClient.java
└── src/main/resources/
    └── application.yml
```

### Step 2: Main Application

```java
// JavaOpsMcpApplication.java
@SpringBootApplication
public class JavaOpsMcpApplication {
    public static void main(String[] args) {
        SpringApplication.run(JavaOpsMcpApplication.class, args);
    }
}
```

### Step 3: Actuator Client

```java
// client/ActuatorClient.java
@Component
public class ActuatorClient {

    private final WebClient webClient;
    private final String targetAppUrl;

    public ActuatorClient(WebClient.Builder builder,
                          @Value("${javaops.target-app-url}") String targetAppUrl) {
        this.targetAppUrl = targetAppUrl;
        this.webClient = builder
            .baseUrl(targetAppUrl)
            .defaultHeader("Accept", "application/json")
            .build();
    }

    public Map<String, Object> getBeans() {
        return webClient.get()
            .uri("/actuator/beans")
            .retrieve()
            .bodyToMono(new ParameterizedTypeReference<Map<String, Object>>() {})
            .block(Duration.ofSeconds(10));
    }

    public Map<String, Object> getMappings() {
        return webClient.get()
            .uri("/actuator/mappings")
            .retrieve()
            .bodyToMono(new ParameterizedTypeReference<Map<String, Object>>() {})
            .block(Duration.ofSeconds(10));
    }

    public Map<String, Object> getHealth() {
        return webClient.get()
            .uri("/actuator/health")
            .retrieve()
            .bodyToMono(new ParameterizedTypeReference<Map<String, Object>>() {})
            .block(Duration.ofSeconds(10));
    }

    public Map<String, Object> getMetrics(String metricName) {
        return webClient.get()
            .uri("/actuator/metrics/{name}", metricName)
            .retrieve()
            .bodyToMono(new ParameterizedTypeReference<Map<String, Object>>() {})
            .block(Duration.ofSeconds(10));
    }

    public List<Map<String, Object>> getFlywayMigrations() {
        var response = webClient.get()
            .uri("/actuator/flyway")
            .retrieve()
            .bodyToMono(new ParameterizedTypeReference<Map<String, Object>>() {})
            .block(Duration.ofSeconds(10));

        // Parse Flyway response structure
        var contexts = (Map<?, ?>) response.get("contexts");
        var firstContext = contexts.values().iterator().next();
        var migrations = (Map<?, ?>) ((Map<?, ?>) firstContext).get("migrations");
        return (List<Map<String, Object>>) migrations;
    }
}
```

### Step 4: Tool 1 — get_beans

```java
// tools/BeanInspectorTool.java
@Component
public class BeanInspectorTool {

    private final ActuatorClient actuatorClient;

    public BeanInspectorTool(ActuatorClient actuatorClient) {
        this.actuatorClient = actuatorClient;
    }

    @Tool(description = """
        List all Spring beans in the target application.
        Useful for understanding what components are loaded,
        finding specific service/repository beans, and diagnosing
        bean configuration issues. Can filter by name pattern.
        """)
    public String getBeans(
        @ToolParam(description = "Optional filter pattern for bean names (e.g., 'Service', 'Repository')")
        String nameFilter
    ) {
        try {
            var beansData = actuatorClient.getBeans();
            var contexts = (Map<?, ?>) beansData.get("contexts");

            var result = new StringBuilder();
            result.append("=== Spring Beans ===\n\n");

            for (var contextEntry : contexts.entrySet()) {
                var context = (Map<?, ?>) contextEntry.getValue();
                var beans = (Map<?, ?>) context.get("beans");

                for (var beanEntry : beans.entrySet()) {
                    String beanName = beanEntry.getKey().toString();

                    if (nameFilter != null && !nameFilter.isBlank()
                        && !beanName.toLowerCase().contains(nameFilter.toLowerCase())) {
                        continue;
                    }

                    var beanInfo = (Map<?, ?>) beanEntry.getValue();
                    String type = beanInfo.getOrDefault("type", "unknown").toString();
                    List<?> dependencies = (List<?>) beanInfo.getOrDefault("dependencies", List.of());

                    result.append(String.format("Bean: %s\n", beanName));
                    result.append(String.format("  Type: %s\n", type));
                    if (!dependencies.isEmpty()) {
                        result.append(String.format("  Depends on: %s\n", dependencies));
                    }
                    result.append("\n");
                }
            }

            return result.toString();
        } catch (Exception e) {
            return "Error getting beans: " + e.getMessage()
                + "\nEnsure target app has spring-boot-actuator and 'beans' endpoint is exposed.";
        }
    }
}
```

### Step 5: Tool 2 — check_migrations

```java
// tools/MigrationCheckerTool.java
@Component
public class MigrationCheckerTool {

    private final ActuatorClient actuatorClient;

    @Tool(description = """
        Check Flyway database migration status.
        Shows all migrations with their version, description, and status (SUCCESS/FAILED/PENDING).
        Use this to diagnose database migration issues or understand the schema evolution history.
        """)
    public String checkMigrations(
        @ToolParam(description = "Filter by status: ALL, SUCCESS, FAILED, or PENDING. Default: ALL")
        String statusFilter
    ) {
        try {
            var migrations = actuatorClient.getFlywayMigrations();
            var filter = statusFilter == null ? "ALL" : statusFilter.toUpperCase();

            var result = new StringBuilder("=== Flyway Migrations ===\n\n");
            int total = 0, success = 0, failed = 0, pending = 0;

            for (var migration : migrations) {
                String state = migration.getOrDefault("state", "UNKNOWN").toString();

                if (!filter.equals("ALL") && !state.equals(filter)) continue;

                total++;
                switch (state) {
                    case "SUCCESS" -> success++;
                    case "FAILED" -> failed++;
                    case "PENDING" -> pending++;
                }

                result.append(String.format("Version: %s | State: %s\n",
                    migration.get("version"), state));
                result.append(String.format("  Description: %s\n",
                    migration.get("description")));
                result.append(String.format("  Script: %s\n",
                    migration.get("script")));
                result.append(String.format("  Executed: %s\n\n",
                    migration.getOrDefault("installedOn", "N/A")));
            }

            result.append(String.format("\nSummary: %d total | %d success | %d failed | %d pending",
                total, success, failed, pending));

            if (failed > 0) {
                result.append("\n⚠️  WARNING: Failed migrations detected! Database may be in inconsistent state.");
            }

            return result.toString();
        } catch (Exception e) {
            return "Error checking migrations: " + e.getMessage();
        }
    }
}
```

### Step 6: Tool 3 — list_endpoints

```java
// tools/EndpointListerTool.java
@Component
public class EndpointListerTool {

    private final ActuatorClient actuatorClient;

    @Tool(description = """
        List all HTTP endpoints (REST API routes) in the target Spring Boot application.
        Shows HTTP method, URL pattern, and handler class/method.
        Use this to understand the API surface, find specific endpoints,
        or diagnose routing issues.
        """)
    public String listEndpoints(
        @ToolParam(description = "Optional URL pattern filter (e.g., '/api/users', '/admin')")
        String urlFilter,
        @ToolParam(description = "Optional HTTP method filter (GET, POST, PUT, DELETE, etc.)")
        String methodFilter
    ) {
        try {
            var mappingsData = actuatorClient.getMappings();
            var contexts = (Map<?, ?>) mappingsData.get("contexts");

            var result = new StringBuilder("=== HTTP Endpoints ===\n\n");

            for (var contextEntry : contexts.entrySet()) {
                var context = (Map<?, ?>) contextEntry.getValue();
                var mappings = (Map<?, ?>) context.get("mappings");
                var dispatcherServlets = (Map<?, ?>) mappings.get("dispatcherServlets");

                if (dispatcherServlets == null) continue;

                for (var servletEntry : dispatcherServlets.entrySet()) {
                    var handlers = (List<?>) servletEntry.getValue();

                    for (var handlerObj : handlers) {
                        var handler = (Map<?, ?>) handlerObj;
                        var details = (Map<?, ?>) handler.get("details");
                        if (details == null) continue;

                        var requestMappingInfo = (Map<?, ?>) details.get("requestMappingConditions");
                        if (requestMappingInfo == null) continue;

                        var patterns = (List<?>) requestMappingInfo.get("patterns");
                        var methods = (List<?>) requestMappingInfo.get("methods");
                        String handlerMethod = handler.getOrDefault("handler", "").toString();

                        if (patterns == null || patterns.isEmpty()) continue;

                        String pattern = patterns.get(0).toString();
                        String method = methods != null && !methods.isEmpty()
                            ? methods.get(0).toString() : "ANY";

                        // Apply filters
                        if (urlFilter != null && !urlFilter.isBlank()
                            && !pattern.contains(urlFilter)) continue;
                        if (methodFilter != null && !methodFilter.isBlank()
                            && !method.equalsIgnoreCase(methodFilter)) continue;

                        result.append(String.format("[%s] %s\n", method, pattern));
                        result.append(String.format("  Handler: %s\n\n", handlerMethod));
                    }
                }
            }

            return result.toString();
        } catch (Exception e) {
            return "Error listing endpoints: " + e.getMessage();
        }
    }
}
```

### Step 7: Tool 4 — analyze_logs

```java
// tools/LogAnalyzerTool.java
@Component
public class LogAnalyzerTool {

    @Value("${javaops.log-file-path}")
    private String logFilePath;

    @Value("${javaops.max-log-lines:500}")
    private int maxLogLines;

    @Tool(description = """
        Analyze recent application logs. Can filter by level (ERROR, WARN, INFO),
        search for keywords, or look for specific time ranges.
        Returns relevant log entries with context. Use this to diagnose errors,
        trace request flows, or investigate performance issues.
        """)
    public String analyzeLogs(
        @ToolParam(description = "Log level filter: ERROR, WARN, INFO, or ALL")
        String levelFilter,
        @ToolParam(description = "Keyword to search for in log messages (e.g., 'OutOfMemory', 'timeout', class name)")
        String keyword,
        @ToolParam(description = "Number of recent lines to analyze (default: 200, max: 500)")
        Integer lineCount
    ) {
        try {
            int linesToRead = lineCount != null ? Math.min(lineCount, maxLogLines) : 200;
            Path logPath = Path.of(logFilePath);

            if (!Files.exists(logPath)) {
                return "Log file not found at: " + logFilePath
                    + "\nSet javaops.log-file-path in application.yml";
            }

            // Read last N lines efficiently
            List<String> lines = readLastNLines(logPath, linesToRead);

            // Filter lines
            String level = levelFilter != null ? levelFilter.toUpperCase() : "ALL";
            List<String> filtered = lines.stream()
                .filter(line -> {
                    boolean levelMatch = level.equals("ALL") || line.contains(level);
                    boolean keywordMatch = keyword == null || keyword.isBlank()
                        || line.toLowerCase().contains(keyword.toLowerCase());
                    return levelMatch && keywordMatch;
                })
                .collect(Collectors.toList());

            if (filtered.isEmpty()) {
                return String.format("No log entries found matching level=%s, keyword=%s in last %d lines",
                    level, keyword, linesToRead);
            }

            // Build summary
            var result = new StringBuilder();
            result.append(String.format("=== Log Analysis (last %d lines) ===\n", linesToRead));
            result.append(String.format("Filter: level=%s, keyword=%s\n", level, keyword));
            result.append(String.format("Matches found: %d\n\n", filtered.size()));

            // Count by level
            long errors = filtered.stream().filter(l -> l.contains("ERROR")).count();
            long warns = filtered.stream().filter(l -> l.contains("WARN")).count();
            result.append(String.format("Breakdown: %d ERROR, %d WARN, %d INFO\n\n",
                errors, warns, filtered.size() - errors - warns));

            // Show entries (limit to avoid overwhelming Claude's context)
            int showCount = Math.min(filtered.size(), 50);
            result.append("=== Log Entries ===\n");
            filtered.subList(filtered.size() - showCount, filtered.size())
                .forEach(line -> result.append(line).append("\n"));

            if (filtered.size() > showCount) {
                result.append(String.format("\n... và %d entries nữa. Dùng keyword filter để narrow down.",
                    filtered.size() - showCount));
            }

            return result.toString();
        } catch (Exception e) {
            return "Error analyzing logs: " + e.getMessage();
        }
    }

    private List<String> readLastNLines(Path path, int n) throws IOException {
        // Efficient reverse reading for large log files
        List<String> lines = new ArrayList<>();
        try (RandomAccessFile raf = new RandomAccessFile(path.toFile(), "r")) {
            long fileLength = raf.length();
            long pos = fileLength - 1;
            StringBuilder sb = new StringBuilder();
            int linesFound = 0;

            while (pos >= 0 && linesFound < n) {
                raf.seek(pos);
                char c = (char) raf.read();
                if (c == '\n' && sb.length() > 0) {
                    lines.add(0, sb.reverse().toString());
                    sb = new StringBuilder();
                    linesFound++;
                } else if (c != '\n') {
                    sb.append(c);
                }
                pos--;
            }
            if (sb.length() > 0) lines.add(0, sb.reverse().toString());
        }
        return lines;
    }
}
```

### Step 8: Tool 5 — get_db_schema

```java
// tools/DatabaseSchemaTool.java
@Component
public class DatabaseSchemaTool {

    private final JdbcTemplate jdbcTemplate;

    @Tool(description = """
        Get database schema information for the target application's database.
        Can list all tables, get columns for a specific table, or find indexes.
        Use this to understand the data model, find missing indexes, or
        diagnose schema-related performance issues.
        """)
    public String getDbSchema(
        @ToolParam(description = "Table name to get detailed schema for. Leave empty to list all tables.")
        String tableName,
        @ToolParam(description = "Include indexes in output? true/false")
        Boolean includeIndexes
    ) {
        try {
            if (tableName == null || tableName.isBlank()) {
                return listAllTables();
            } else {
                return getTableSchema(tableName, Boolean.TRUE.equals(includeIndexes));
            }
        } catch (Exception e) {
            return "Error getting schema: " + e.getMessage();
        }
    }

    private String listAllTables() {
        String sql = """
            SELECT table_name, pg_size_pretty(pg_total_relation_size(quote_ident(table_name))) as size
            FROM information_schema.tables
            WHERE table_schema = 'public'
            ORDER BY table_name
            """;

        var tables = jdbcTemplate.queryForList(sql);
        var result = new StringBuilder("=== Database Tables ===\n\n");

        for (var table : tables) {
            result.append(String.format("- %s (%s)\n",
                table.get("table_name"), table.get("size")));
        }

        result.append(String.format("\nTotal: %d tables. Use get_db_schema with table name for details.",
            tables.size()));
        return result.toString();
    }

    private String getTableSchema(String tableName, boolean includeIndexes) {
        // Columns
        String columnSql = """
            SELECT column_name, data_type, is_nullable, column_default
            FROM information_schema.columns
            WHERE table_schema = 'public' AND table_name = ?
            ORDER BY ordinal_position
            """;

        var columns = jdbcTemplate.queryForList(columnSql, tableName);
        var result = new StringBuilder(String.format("=== Table: %s ===\n\n", tableName));
        result.append("Columns:\n");

        for (var col : columns) {
            result.append(String.format("  %-25s %-20s nullable=%s default=%s\n",
                col.get("column_name"), col.get("data_type"),
                col.get("is_nullable"), col.get("column_default")));
        }

        if (includeIndexes) {
            String indexSql = """
                SELECT indexname, indexdef
                FROM pg_indexes
                WHERE schemaname = 'public' AND tablename = ?
                ORDER BY indexname
                """;

            var indexes = jdbcTemplate.queryForList(indexSql, tableName);
            result.append("\nIndexes:\n");

            for (var idx : indexes) {
                result.append(String.format("  %s\n    %s\n",
                    idx.get("indexname"), idx.get("indexdef")));
            }

            if (indexes.isEmpty()) {
                result.append("  (no indexes found — potential performance issue!)\n");
            }
        }

        // Row count estimate
        try {
            Long rowCount = jdbcTemplate.queryForObject(
                "SELECT reltuples::bigint FROM pg_class WHERE relname = ?",
                Long.class, tableName
            );
            result.append(String.format("\nApprox row count: %,d\n", rowCount));
        } catch (Exception ignored) {}

        return result.toString();
    }
}
```

### Step 9: Tool 6 & 7 — Health Check và Error Search

```java
// tools/HealthCheckTool.java
@Component
public class HealthCheckTool {

    private final ActuatorClient actuatorClient;

    @Tool(description = """
        Run a comprehensive health check on the target Spring Boot application.
        Returns status of all health indicators: database, disk space, cache,
        custom health indicators, and JVM metrics.
        """)
    public String runHealthCheck() {
        try {
            var health = actuatorClient.getHealth();
            var result = new StringBuilder("=== Health Check ===\n\n");

            String overallStatus = health.getOrDefault("status", "UNKNOWN").toString();
            result.append(String.format("Overall Status: %s\n\n", overallStatus));

            var components = (Map<?, ?>) health.get("components");
            if (components != null) {
                result.append("Components:\n");
                for (var entry : components.entrySet()) {
                    var component = (Map<?, ?>) entry.getValue();
                    String status = component.getOrDefault("status", "UNKNOWN").toString();
                    result.append(String.format("  %-20s: %s\n", entry.getKey(), status));

                    // Show details for non-UP components
                    if (!"UP".equals(status)) {
                        var details = component.get("details");
                        if (details != null) {
                            result.append(String.format("    Details: %s\n", details));
                        }
                    }
                }
            }

            // JVM metrics
            appendJvmMetrics(result);

            return result.toString();
        } catch (Exception e) {
            return "Error running health check: " + e.getMessage();
        }
    }

    private void appendJvmMetrics(StringBuilder result) {
        try {
            var heapUsed = actuatorClient.getMetrics("jvm.memory.used");
            var heapMax = actuatorClient.getMetrics("jvm.memory.max");

            result.append("\nJVM Metrics:\n");

            if (heapUsed != null) {
                var measurements = (List<?>) heapUsed.get("measurements");
                if (measurements != null && !measurements.isEmpty()) {
                    var value = ((Map<?, ?>) measurements.get(0)).get("value");
                    long bytes = ((Number) value).longValue();
                    result.append(String.format("  Heap Used: %s MB\n", bytes / 1024 / 1024));
                }
            }

            if (heapMax != null) {
                var measurements = (List<?>) heapMax.get("measurements");
                if (measurements != null && !measurements.isEmpty()) {
                    var value = ((Map<?, ?>) measurements.get(0)).get("value");
                    long bytes = ((Number) value).longValue();
                    result.append(String.format("  Heap Max:  %s MB\n", bytes / 1024 / 1024));
                }
            }
        } catch (Exception ignored) {
            result.append("  (JVM metrics unavailable)\n");
        }
    }
}

// tools/ErrorSearchTool.java
@Component
public class ErrorSearchTool {

    @Value("${javaops.log-file-path}")
    private String logFilePath;

    @Tool(description = """
        Search for error patterns in application logs.
        Finds recurring errors, groups similar exceptions, and counts frequency.
        Use this to identify the most common errors affecting the application.
        """)
    public String searchErrors(
        @ToolParam(description = "Search term or exception class name (e.g., 'NullPointerException', 'timeout')")
        String searchTerm,
        @ToolParam(description = "Number of hours to look back (default: 24)")
        Integer hoursBack
    ) {
        try {
            int hours = hoursBack != null ? hoursBack : 24;
            Path logPath = Path.of(logFilePath);

            if (!Files.exists(logPath)) {
                return "Log file not found: " + logFilePath;
            }

            List<String> allLines = Files.readAllLines(logPath);

            // Filter ERROR lines containing search term
            var errorGroups = new HashMap<String, Integer>();
            var recentErrors = new ArrayList<String>();
            LocalDateTime cutoff = LocalDateTime.now().minusHours(hours);

            for (String line : allLines) {
                if (!line.contains("ERROR")) continue;
                if (searchTerm != null && !line.toLowerCase().contains(searchTerm.toLowerCase())) continue;

                // Extract exception class as grouping key
                String key = extractExceptionKey(line);
                errorGroups.merge(key, 1, Integer::sum);
                recentErrors.add(line);
            }

            var result = new StringBuilder("=== Error Search Results ===\n\n");
            result.append(String.format("Search: '%s' | Last %d hours\n", searchTerm, hours));
            result.append(String.format("Total matches: %d\n\n", recentErrors.size()));

            // Top error types
            result.append("Top Error Types:\n");
            errorGroups.entrySet().stream()
                .sorted(Map.Entry.<String, Integer>comparingByValue().reversed())
                .limit(10)
                .forEach(entry ->
                    result.append(String.format("  %4d × %s\n", entry.getValue(), entry.getKey()))
                );

            // Recent 10 errors
            result.append("\nMost Recent Errors:\n");
            int startIdx = Math.max(0, recentErrors.size() - 10);
            recentErrors.subList(startIdx, recentErrors.size())
                .forEach(line -> result.append(line).append("\n"));

            return result.toString();
        } catch (Exception e) {
            return "Error searching logs: " + e.getMessage();
        }
    }

    private String extractExceptionKey(String line) {
        // Try to extract exception class name
        int exIdx = line.indexOf("Exception");
        if (exIdx > 0) {
            int start = line.lastIndexOf(' ', exIdx - 1) + 1;
            return line.substring(start, exIdx + "Exception".length());
        }
        return "UnknownError";
    }
}
```

### Step 10: MCP Resources

```java
// resources/DatabaseSchemaResource.java
@Component
public class DatabaseSchemaResource {

    private final DatabaseSchemaTool schemaTool;

    // MCP Resource: db://schema
    // Claude có thể access resource này bất kỳ lúc nào
    @McpResource(uri = "db://schema",
                 name = "Database Schema",
                 description = "Full database schema for all tables",
                 mimeType = "text/plain")
    public String getDatabaseSchema() {
        return schemaTool.getDbSchema(null, true);
    }
}

// resources/RecentLogsResource.java
@Component
public class RecentLogsResource {

    private final LogAnalyzerTool logAnalyzerTool;

    // MCP Resource: logs://recent
    @McpResource(uri = "logs://recent",
                 name = "Recent Logs",
                 description = "Recent application logs (last 200 lines)",
                 mimeType = "text/plain")
    public String getRecentLogs() {
        return logAnalyzerTool.analyzeLogs("ALL", null, 200);
    }
}
```

---

## Claude Code Configuration

Thêm vào `.claude/settings.json` của project muốn dùng MCP server:

```json
{
  "mcpServers": {
    "javaops": {
      "command": "java",
      "args": [
        "-jar",
        "/path/to/javaops-mcp-suite-1.0.0.jar"
      ],
      "env": {
        "TARGET_APP_URL": "http://localhost:8080",
        "TARGET_DB_URL": "jdbc:postgresql://localhost:5432/myapp",
        "TARGET_DB_USER": "postgres",
        "TARGET_DB_PASSWORD": "your-password",
        "TARGET_LOG_PATH": "/var/log/myapp/application.log"
      }
    }
  }
}
```

---

## Testing Guide

### Dùng MCP Inspector

```bash
# Install MCP Inspector
npx @modelcontextprotocol/inspector

# Chạy với STDIO transport
npx @modelcontextprotocol/inspector java -jar javaops-mcp-suite-1.0.0.jar
```

MCP Inspector cho phép bạn:
- List all tools và resources
- Call từng tool thủ công với custom inputs
- Xem raw JSON responses
- Debug transport issues

### Test Cases

```bash
# Test 1: List all beans
Tool: get_beans
Input: { "nameFilter": "Service" }
Expected: List of all @Service beans

# Test 2: Check migrations
Tool: check_migrations
Input: { "statusFilter": "FAILED" }
Expected: Any failed Flyway migrations (hopefully empty list)

# Test 3: Find slow endpoints
Tool: list_endpoints
Input: { "urlFilter": "/api" }
Expected: All /api/* routes với handler info

# Test 4: Database schema
Tool: get_db_schema
Input: { "tableName": "users", "includeIndexes": true }
Expected: Columns, types, indexes for users table

# Test 5: Recent errors
Tool: search_errors
Input: { "searchTerm": "Exception", "hoursBack": 1 }
Expected: Recent exceptions grouped by type
```

---

## Demo Script

**Setup:** Chạy một Spring Boot app với actuator enabled và một số intentional issues.

```
User: "The /api/orders endpoint is returning 500 errors. What's wrong?"

Claude calls:
1. list_endpoints (filter: "orders") → finds GET /api/orders handler
2. search_errors (searchTerm: "orders") → finds NullPointerException in OrderService
3. analyze_logs (level: "ERROR", keyword: "OrderService") → full stack trace
4. get_db_schema (tableName: "orders", includeIndexes: true) → sees missing FK index

Claude responds:
"OrderService.getOrdersWithItems() đang throw NullPointerException ở line 47.
Stack trace cho thấy item list có thể null khi order chưa có items.
Database schema cũng cho thấy thiếu index trên orders.customer_id,
có thể gây slow query khi load orders cho customer cụ thể.

Fix đề nghị:
1. Add null check: if (order.getItems() != null) ...
2. Add database index: CREATE INDEX idx_orders_customer_id ON orders(customer_id);
"
```

---

## GitHub README Template

```markdown
# JavaOps MCP Suite

> Connect Claude Code to any Spring Boot application internals

## What it does

JavaOps MCP Suite is an MCP server that gives Claude direct access to your
Spring Boot application's internals — beans, endpoints, database schema, logs,
and health metrics.

**Ask Claude things like:**
- "Why is the /api/orders endpoint slow?"
- "Which Flyway migrations have failed?"
- "What Spring beans depend on UserService?"
- "Show me recent ERROR logs related to authentication"

## Tools

| Tool | Description |
|------|-------------|
| `get_beans` | List Spring beans with dependency info |
| `check_migrations` | Flyway migration status |
| `list_endpoints` | All HTTP endpoints and handlers |
| `analyze_logs` | Filter and analyze application logs |
| `get_db_schema` | Database tables, columns, indexes |
| `run_health_check` | Health indicators and JVM metrics |
| `search_errors` | Find and group error patterns |

## Quick Start

\`\`\`bash
git clone https://github.com/yourname/javaops-mcp-suite
cd javaops-mcp-suite
mvn package -DskipTests
\`\`\`

Add to `.claude/settings.json`:
\`\`\`json
{
  "mcpServers": {
    "javaops": {
      "command": "java",
      "args": ["-jar", "target/javaops-mcp-suite-1.0.0.jar"],
      "env": { "TARGET_APP_URL": "http://localhost:8080" }
    }
  }
}
\`\`\`

## Requirements

- Java 21+
- Spring Boot application với Actuator enabled
- PostgreSQL (for schema tools)
```

---

## How to Present in Portfolio / Interviews

**One-liner:** "MCP server viết bằng Java cho phép Claude Code introspect trực tiếp vào Spring Boot applications — beans, endpoints, database schema, logs."

**Demo flow (5 phút):**
1. Show Claude Code với MCP server connected (30 giây)
2. Hỏi một câu về production issue (1 phút setup context)
3. Show Claude calling tools step by step (2 phút)
4. Show final diagnosis và fix suggestion (1 phút)
5. "Đây là cái mà mọi Spring Boot team đều cần" (30 giây)

**Technical depth questions to prepare:**
- Tại sao dùng STDIO transport thay vì HTTP?
- Làm thế nào handle authentication cho actuator endpoints?
- Cách extend thêm tools cho specific frameworks (e.g., JPA queries, Redis)?
- Production considerations: rate limiting, sensitive data masking trong logs?
