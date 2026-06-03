# Bài 03: Building MCP Server với Java

> **Module**: 03 — Model Context Protocol  
> **Thời lượng**: ~5 giờ đọc + thực hành  
> **Level**: Intermediate → Advanced

---

## Mục tiêu bài học

- Setup Spring Boot project với Spring AI MCP Server
- Implement Tools, Resources, và Prompts bằng annotations
- Build Spring Boot Actuator MCP Server thực tế
- Test server với MCP Inspector
- Package và deploy cho team sử dụng
- Áp dụng security best practices

---

## 1. Tại sao Build Custom MCP Server?

Existing MCP servers (filesystem, postgres, github) xử lý generic use cases. Nhưng trong enterprise Java context, bạn cần **domain-specific tools** mà chỉ bạn mới biết cách build:

- **Spring Actuator integration**: Claude cần biết application health, bean graph, endpoint list — không có server nào làm được ngoài bạn
- **Flyway migration context**: Claude cần đọc migration history theo cách Spring-aware
- **Domain-specific operations**: Run specific health checks, validate business rules, trigger specific workflows
- **Internal APIs**: Connect với internal systems không có public MCP server

Custom MCP server = **expose application internals một cách an toàn và có kiểm soát**.

---

## 2. Spring AI MCP — Architecture Overview

Spring AI implements MCP protocol với Spring Boot auto-configuration. Architecture:

```
Claude Code (MCP Host)
         │
         │ JSON-RPC over stdio
         ▼
Spring Boot MCP Server (your app)
         │
         ├── @Tool methods     → exposed as MCP Tools
         ├── @McpResource     → exposed as MCP Resources  
         └── @McpPrompt       → exposed as MCP Prompts
         │
         ├── Spring Actuator  → /actuator/beans, /health, /mappings
         ├── JdbcTemplate     → Database queries
         └── ApplicationContext → Spring internals
```

Spring AI MCP Server tự động:
- Handle MCP handshake và capability negotiation
- Route JSON-RPC calls tới `@Tool` methods
- Serialize/deserialize tool parameters
- Format responses theo MCP spec

---

## 3. Project Setup

### 3.1 Tạo Spring Boot Project

```bash
# Dùng Spring Initializr
curl https://start.spring.io/starter.zip \
  -d type=maven-project \
  -d language=java \
  -d bootVersion=3.3.0 \
  -d baseDir=actuator-mcp-server \
  -d groupId=com.example \
  -d artifactId=actuator-mcp-server \
  -d name=ActuatorMcpServer \
  -d dependencies=web,actuator \
  -o actuator-mcp-server.zip

unzip actuator-mcp-server.zip
cd actuator-mcp-server
```

### 3.2 Maven Dependencies

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.0</version>
    </parent>
    
    <groupId>com.example</groupId>
    <artifactId>actuator-mcp-server</artifactId>
    <version>1.0.0</version>
    <name>Spring Boot Actuator MCP Server</name>
    
    <properties>
        <java.version>21</java.version>
        <spring-ai.version>1.0.0</spring-ai.version>
    </properties>
    
    <dependencies>
        <!-- Spring Boot Web (cần cho Actuator endpoints) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        
        <!-- Spring Boot Actuator -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        
        <!-- Spring AI MCP Server — đây là dependency chính -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-mcp-server-spring-boot-starter</artifactId>
            <version>${spring-ai.version}</version>
        </dependency>
        
        <!-- JDBC nếu expose database tools -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>
        
        <!-- PostgreSQL driver -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        
        <!-- Lombok cho cleaner code -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        
        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.ai</groupId>
                <artifactId>spring-ai-bom</artifactId>
                <version>${spring-ai.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### 3.3 Application Properties

```yaml
# src/main/resources/application.yml

spring:
  application:
    name: actuator-mcp-server
  
  # MCP Server configuration
  ai:
    mcp:
      server:
        name: "Spring Boot Actuator MCP Server"
        version: "1.0.0"
        # stdio transport: server nhận commands từ stdin, trả kết quả qua stdout
        transport: stdio
  
  # Database (nếu expose DB tools)
  datasource:
    url: ${DATABASE_URL:jdbc:postgresql://localhost:5432/myapp_dev}
    username: ${DB_USERNAME:mcp_readonly}
    password: ${DB_PASSWORD:}

# Expose tất cả Actuator endpoints (cho MCP server đọc)
management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    health:
      show-details: always
    loggers:
      enabled: true

# Target application Actuator URL
# MCP server sẽ call Actuator endpoints của application này
app:
  actuator:
    base-url: ${ACTUATOR_BASE_URL:http://localhost:8080/actuator}

# Logging: MCP server log ra stderr, không ra stdout (stdout dành cho JSON-RPC)
logging:
  file:
    name: logs/mcp-server.log
  pattern:
    console: ""  # Tắt console logging (vì stdout dùng cho MCP protocol)
```

**Quan trọng**: Khi dùng stdio transport, stdout của process **phải** là JSON-RPC messages. Bất kỳ log nào ra stdout sẽ break MCP protocol. Luôn log ra stderr hoặc file.

---

## 4. Main Application Class

```java
package com.example.actuatormcp;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ActuatorMcpServerApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(ActuatorMcpServerApplication.class, args);
    }
}
```

---

## 5. Implementing Tools

### 5.1 Actuator Client — Gọi Actuator Endpoints

Trước khi implement tools, cần một service để gọi Actuator endpoints:

```java
package com.example.actuatormcp.client;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;

@Component
@Slf4j
public class ActuatorClient {
    
    private final RestTemplate restTemplate;
    private final ObjectMapper objectMapper;
    private final String baseUrl;
    
    public ActuatorClient(
        RestTemplate restTemplate,
        ObjectMapper objectMapper,
        @Value("${app.actuator.base-url}") String baseUrl
    ) {
        this.restTemplate = restTemplate;
        this.objectMapper = objectMapper;
        this.baseUrl = baseUrl;
    }
    
    public String getBeans() {
        try {
            JsonNode response = restTemplate.getForObject(
                baseUrl + "/beans", JsonNode.class);
            return formatBeansResponse(response);
        } catch (Exception e) {
            log.error("Failed to fetch beans from actuator", e);
            return "Error: Could not connect to application actuator at " + baseUrl 
                   + ". Is the application running? Error: " + e.getMessage();
        }
    }
    
    public String getHealth() {
        try {
            JsonNode response = restTemplate.getForObject(
                baseUrl + "/health", JsonNode.class);
            return objectMapper.writerWithDefaultPrettyPrinter()
                              .writeValueAsString(response);
        } catch (Exception e) {
            log.error("Failed to fetch health from actuator", e);
            return "Error fetching health: " + e.getMessage();
        }
    }
    
    public String getMappings() {
        try {
            JsonNode response = restTemplate.getForObject(
                baseUrl + "/mappings", JsonNode.class);
            return formatMappingsResponse(response);
        } catch (Exception e) {
            log.error("Failed to fetch mappings", e);
            return "Error fetching mappings: " + e.getMessage();
        }
    }
    
    public String getFlywayStatus() {
        try {
            JsonNode response = restTemplate.getForObject(
                baseUrl + "/flyway", JsonNode.class);
            return formatFlywayResponse(response);
        } catch (Exception e) {
            return "Flyway endpoint not available. " +
                   "Ensure flyway is on classpath and /actuator/flyway is exposed. " +
                   "Error: " + e.getMessage();
        }
    }
    
    public String getLogs(String level, int count) {
        try {
            // Logfile endpoint
            String logContent = restTemplate.getForObject(
                baseUrl + "/logfile", String.class);
            
            if (logContent == null) {
                return "Log file not available. Set logging.file.name in application properties.";
            }
            
            // Filter by level and return last N lines
            return logContent.lines()
                .filter(line -> level == null || line.contains(level))
                .reduce((first, second) -> second) // keep going to find last lines
                .map(last -> {
                    String[] allLines = logContent.lines()
                        .filter(line -> level == null || line.contains(level))
                        .toArray(String[]::new);
                    int start = Math.max(0, allLines.length - count);
                    StringBuilder sb = new StringBuilder();
                    for (int i = start; i < allLines.length; i++) {
                        sb.append(allLines[i]).append("\n");
                    }
                    return sb.toString();
                })
                .orElse("No log entries found for level: " + level);
                
        } catch (Exception e) {
            return "Error fetching logs: " + e.getMessage();
        }
    }
    
    private String formatBeansResponse(JsonNode response) {
        StringBuilder sb = new StringBuilder();
        sb.append("Spring Application Beans:\n\n");
        
        JsonNode contexts = response.path("contexts");
        contexts.fields().forEachRemaining(contextEntry -> {
            JsonNode beans = contextEntry.getValue().path("beans");
            sb.append("Context: ").append(contextEntry.getKey()).append("\n");
            sb.append("Total beans: ").append(beans.size()).append("\n\n");
            
            beans.fields().forEachRemaining(beanEntry -> {
                JsonNode bean = beanEntry.getValue();
                sb.append("Bean: ").append(beanEntry.getKey()).append("\n");
                sb.append("  Type: ").append(bean.path("type").asText()).append("\n");
                
                JsonNode dependencies = bean.path("dependencies");
                if (dependencies.size() > 0) {
                    sb.append("  Dependencies: ");
                    dependencies.forEach(dep -> sb.append(dep.asText()).append(", "));
                    sb.append("\n");
                }
                sb.append("\n");
            });
        });
        
        return sb.toString();
    }
    
    private String formatMappingsResponse(JsonNode response) {
        StringBuilder sb = new StringBuilder();
        sb.append("REST API Endpoints:\n\n");
        
        JsonNode contexts = response.path("contexts");
        contexts.fields().forEachRemaining(contextEntry -> {
            JsonNode mappings = contextEntry.getValue()
                                            .path("mappings")
                                            .path("dispatcherServlets")
                                            .path("dispatcherServlet");
            
            if (mappings.isArray()) {
                mappings.forEach(mapping -> {
                    JsonNode details = mapping.path("details");
                    JsonNode requestMappingConditions = details.path("requestMappingConditions");
                    
                    String methods = requestMappingConditions.path("methods").toString();
                    String patterns = requestMappingConditions.path("patterns").toString();
                    String handlerMethod = details.path("handlerMethod").path("descriptor").asText();
                    
                    sb.append(methods).append(" ").append(patterns)
                      .append("\n  Handler: ").append(handlerMethod).append("\n\n");
                });
            }
        });
        
        return sb.toString();
    }
    
    private String formatFlywayResponse(JsonNode response) {
        StringBuilder sb = new StringBuilder();
        sb.append("Flyway Migration Status:\n\n");
        
        JsonNode contexts = response.path("contexts");
        contexts.fields().forEachRemaining(contextEntry -> {
            JsonNode flywayBeans = contextEntry.getValue().path("flywayBeans");
            flywayBeans.fields().forEachRemaining(beanEntry -> {
                JsonNode migrations = beanEntry.getValue().path("migrations");
                sb.append("Flyway instance: ").append(beanEntry.getKey()).append("\n");
                
                migrations.forEach(migration -> {
                    sb.append(String.format("  [%s] V%s - %s (%s)\n",
                        migration.path("state").asText(),
                        migration.path("version").asText("?"),
                        migration.path("description").asText(),
                        migration.path("installedOn").asText("pending")
                    ));
                });
                sb.append("\n");
            });
        });
        
        return sb.toString();
    }
}
```

### 5.2 Main Tools Implementation

```java
package com.example.actuatormcp.tools;

import com.example.actuatormcp.client.ActuatorClient;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
@Slf4j
public class ActuatorTools {
    
    private final ActuatorClient actuatorClient;
    
    @Tool(description = """
        Get all Spring beans in the application context.
        Returns bean names, their types, and dependency graph.
        Useful for understanding application structure, finding autowiring issues,
        or checking if a specific component is registered.
        """)
    public String getSpringBeans() {
        log.debug("MCP tool invoked: getSpringBeans");
        return actuatorClient.getBeans();
    }
    
    @Tool(description = """
        Check Flyway database migration status.
        Returns each migration with: version number, description, 
        execution status (SUCCESS/FAILED/PENDING), and installation timestamp.
        Useful for verifying migrations ran correctly after deployment.
        """)
    public String getDatabaseMigrationStatus() {
        log.debug("MCP tool invoked: getDatabaseMigrationStatus");
        return actuatorClient.getFlywayStatus();
    }
    
    @Tool(description = """
        List all REST API endpoints with HTTP method, URL pattern, and handler class/method.
        Returns a structured list of all @RequestMapping, @GetMapping, @PostMapping etc.
        Useful for API documentation, finding endpoints, checking URL patterns.
        """)
    public String getApiEndpoints() {
        log.debug("MCP tool invoked: getApiEndpoints");
        return actuatorClient.getMappings();
    }
    
    @Tool(description = """
        Get application health status from Spring Actuator.
        Includes: overall status, database connectivity, disk space, 
        custom health indicators, and component details.
        Returns HEALTHY/UNHEALTHY with details for each component.
        """)
    public String getHealthStatus() {
        log.debug("MCP tool invoked: getHealthStatus");
        return actuatorClient.getHealth();
    }
    
    @Tool(description = """
        Get recent application log entries filtered by log level.
        Use this to investigate errors, warnings, or trace specific operations.
        Returns the most recent N log lines matching the specified level.
        """)
    public String getRecentLogs(
        @ToolParam(description = "Log level to filter: ERROR, WARN, INFO, DEBUG. " +
                                 "Pass null or empty string for all levels.") 
        String level,
        
        @ToolParam(description = "Number of recent log lines to return. " +
                                 "Recommended: 50-200. Max: 1000.") 
        int count
    ) {
        log.debug("MCP tool invoked: getRecentLogs level={} count={}", level, count);
        
        // Input validation
        if (count < 1 || count > 1000) {
            return "Error: count must be between 1 and 1000. Provided: " + count;
        }
        
        String normalizedLevel = (level == null || level.isBlank()) ? null : level.toUpperCase();
        if (normalizedLevel != null && !normalizedLevel.matches("ERROR|WARN|INFO|DEBUG|TRACE")) {
            return "Error: Invalid log level '" + level + "'. Valid values: ERROR, WARN, INFO, DEBUG, TRACE";
        }
        
        return actuatorClient.getLogs(normalizedLevel, count);
    }
}
```

### 5.3 Database Tools

```java
package com.example.actuatormcp.tools;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.Map;

@Component
@RequiredArgsConstructor
@Slf4j
public class DatabaseTools {
    
    private final JdbcTemplate jdbcTemplate;
    
    @Tool(description = """
        Get the full database schema: all tables with their columns, data types,
        nullable constraints, and primary keys. Also includes foreign key relationships.
        Use this first to understand database structure before writing queries.
        """)
    public String getDatabaseSchema() {
        log.debug("MCP tool invoked: getDatabaseSchema");
        
        try {
            // Get all tables
            List<String> tables = jdbcTemplate.queryForList(
                """
                SELECT table_name 
                FROM information_schema.tables 
                WHERE table_schema = 'public' 
                  AND table_type = 'BASE TABLE'
                ORDER BY table_name
                """,
                String.class
            );
            
            StringBuilder sb = new StringBuilder();
            sb.append("Database Schema\n");
            sb.append("==============\n\n");
            sb.append("Tables: ").append(tables.size()).append("\n\n");
            
            for (String table : tables) {
                sb.append("TABLE: ").append(table).append("\n");
                sb.append("-".repeat(40)).append("\n");
                
                // Get columns
                List<Map<String, Object>> columns = jdbcTemplate.queryForList(
                    """
                    SELECT 
                        c.column_name,
                        c.data_type,
                        c.character_maximum_length,
                        c.is_nullable,
                        c.column_default,
                        CASE WHEN kcu.column_name IS NOT NULL THEN 'PK' ELSE '' END as pk
                    FROM information_schema.columns c
                    LEFT JOIN information_schema.key_column_usage kcu
                        ON c.table_name = kcu.table_name 
                        AND c.column_name = kcu.column_name
                        AND kcu.constraint_name LIKE '%_pkey'
                    WHERE c.table_schema = 'public'
                      AND c.table_name = ?
                    ORDER BY c.ordinal_position
                    """,
                    table
                );
                
                for (Map<String, Object> col : columns) {
                    String type = col.get("data_type").toString();
                    if (col.get("character_maximum_length") != null) {
                        type += "(" + col.get("character_maximum_length") + ")";
                    }
                    String nullable = "YES".equals(col.get("is_nullable")) ? "nullable" : "NOT NULL";
                    String pk = col.get("pk").toString();
                    
                    sb.append(String.format("  %-30s %-25s %-10s %s\n",
                        col.get("column_name"),
                        type,
                        nullable,
                        pk
                    ));
                }
                
                // Get foreign keys
                List<Map<String, Object>> fks = jdbcTemplate.queryForList(
                    """
                    SELECT
                        kcu.column_name,
                        ccu.table_name AS foreign_table,
                        ccu.column_name AS foreign_column
                    FROM information_schema.table_constraints tc
                    JOIN information_schema.key_column_usage kcu
                        ON tc.constraint_name = kcu.constraint_name
                    JOIN information_schema.constraint_column_usage ccu
                        ON ccu.constraint_name = tc.constraint_name
                    WHERE tc.constraint_type = 'FOREIGN KEY'
                      AND tc.table_name = ?
                    """,
                    table
                );
                
                if (!fks.isEmpty()) {
                    sb.append("\n  Foreign Keys:\n");
                    fks.forEach(fk -> sb.append(String.format(
                        "    %s → %s.%s\n",
                        fk.get("column_name"),
                        fk.get("foreign_table"),
                        fk.get("foreign_column")
                    )));
                }
                
                sb.append("\n");
            }
            
            return sb.toString();
            
        } catch (Exception e) {
            log.error("Error fetching database schema", e);
            return "Error fetching schema: " + e.getMessage();
        }
    }
    
    @Tool(description = """
        Execute a read-only SQL SELECT query against the database.
        ONLY SELECT statements are allowed — no INSERT, UPDATE, DELETE, DROP, etc.
        Returns results as formatted text. Limit results with LIMIT clause.
        Maximum 500 rows returned regardless of query.
        """)
    public String executeQuery(
        @ToolParam(description = "SQL SELECT query to execute. Must start with SELECT.")
        String sql
    ) {
        log.info("MCP tool invoked: executeQuery sql={}", sql);
        
        // Security: only allow SELECT
        String trimmed = sql.trim().toUpperCase();
        if (!trimmed.startsWith("SELECT") && !trimmed.startsWith("WITH")) {
            return "Error: Only SELECT (and CTE WITH...SELECT) queries are allowed. " +
                   "Received: " + sql.substring(0, Math.min(50, sql.length()));
        }
        
        // Block dangerous keywords even inside SELECT
        String[] blocked = {"INSERT", "UPDATE", "DELETE", "DROP", "TRUNCATE", 
                           "ALTER", "CREATE", "GRANT", "REVOKE", "EXECUTE", "CALL"};
        for (String keyword : blocked) {
            if (trimmed.contains(keyword + " ") || trimmed.contains(keyword + ";")) {
                return "Error: Query contains blocked keyword: " + keyword;
            }
        }
        
        try {
            // Add LIMIT if not present
            String finalSql = sql;
            if (!trimmed.contains("LIMIT")) {
                finalSql = sql + " LIMIT 500";
            }
            
            List<Map<String, Object>> results = jdbcTemplate.queryForList(finalSql);
            
            if (results.isEmpty()) {
                return "Query returned 0 rows.";
            }
            
            // Format as table
            StringBuilder sb = new StringBuilder();
            sb.append("Results (").append(results.size()).append(" rows):\n\n");
            
            // Headers
            results.get(0).keySet().forEach(col -> 
                sb.append(String.format("%-20s", col)));
            sb.append("\n");
            sb.append("-".repeat(20 * results.get(0).size())).append("\n");
            
            // Rows
            results.forEach(row -> {
                row.values().forEach(val -> 
                    sb.append(String.format("%-20s", val == null ? "NULL" : val.toString())));
                sb.append("\n");
            });
            
            return sb.toString();
            
        } catch (Exception e) {
            log.error("Error executing query: {}", sql, e);
            return "Query error: " + e.getMessage() + 
                   "\nQuery was: " + sql;
        }
    }
}
```

---

## 6. Implementing Resources

Resources expose data AI đọc như context, không cần explicit tool call:

```java
package com.example.actuatormcp.resources;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.ai.mcp.annotation.McpResource;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.env.Environment;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
@Slf4j
public class ApplicationResources {
    
    private final Environment environment;
    
    @Value("${spring.application.name:unknown}")
    private String applicationName;
    
    @McpResource(
        uri = "app://info",
        description = "Application metadata: name, version, active profiles, and key configuration"
    )
    public String getApplicationInfo() {
        return """
            Application: %s
            Active Profiles: %s
            Java Version: %s
            Spring Boot Version: %s
            """.formatted(
            applicationName,
            String.join(", ", environment.getActiveProfiles()),
            System.getProperty("java.version"),
            org.springframework.boot.SpringBootVersion.getVersion()
        );
    }
    
    @McpResource(
        uri = "app://environment",
        description = "Non-sensitive environment configuration keys (no passwords or tokens)"
    )
    public String getSafeEnvironmentConfig() {
        StringBuilder sb = new StringBuilder();
        sb.append("Environment Configuration (safe keys only):\n\n");
        
        // Only expose safe, non-sensitive config
        String[] safeKeys = {
            "spring.application.name",
            "spring.profiles.active", 
            "server.port",
            "spring.datasource.url",   // URL safe, không bao gồm password
            "management.endpoints.web.exposure.include",
            "logging.level.root"
        };
        
        for (String key : safeKeys) {
            String value = environment.getProperty(key);
            if (value != null) {
                // Mask passwords trong URLs
                String safeValue = value.replaceAll(":[^@/]+@", ":***@");
                sb.append(key).append(" = ").append(safeValue).append("\n");
            }
        }
        
        return sb.toString();
    }
}
```

---

## 7. Implementing Prompts

```java
package com.example.actuatormcp.prompts;

import org.springframework.ai.mcp.annotation.McpPrompt;
import org.springframework.ai.mcp.annotation.PromptParam;
import org.springframework.ai.mcp.spec.schema.PromptMessage;
import org.springframework.stereotype.Component;

@Component
public class ApplicationPrompts {
    
    @McpPrompt(
        name = "analyze-spring-issue",
        description = "Template for analyzing Spring Boot application issues. " +
                      "Guides systematic investigation of production problems."
    )
    public PromptMessage analyzeSpringIssue(
        @PromptParam("symptom") String symptom,
        @PromptParam("affected_endpoint") String endpoint
    ) {
        return PromptMessage.user("""
            Analyze this Spring Boot issue:
            
            Symptom: %s
            Affected endpoint: %s
            
            Investigation steps:
            1. Check application health status (use getHealthStatus tool)
            2. Look for ERROR logs around the time of the issue (use getRecentLogs)
            3. Verify the endpoint exists and handler is correct (use getApiEndpoints)
            4. Check database connectivity if endpoint uses data (use getDatabaseSchema)
            5. Look for relevant beans in application context (use getSpringBeans)
            
            Provide: root cause hypothesis, evidence from tools, recommended fix.
            """.formatted(symptom, endpoint));
    }
    
    @McpPrompt(
        name = "review-migration",
        description = "Template for reviewing database migration before applying to production."
    )
    public PromptMessage reviewMigration(
        @PromptParam("migration_sql") String migrationSql,
        @PromptParam("migration_version") String version
    ) {
        return PromptMessage.user("""
            Review this Flyway migration before production deployment:
            
            Version: %s
            SQL:
            ```sql
            %s
            ```
            
            Check for:
            1. Performance impact: table locks, index creation on large tables
            2. Rollback strategy: is this reversible?
            3. Data migration safety: any data loss risk?
            4. Index naming conventions
            5. Foreign key constraints: will they cause issues with existing data?
            
            Cross-reference with current schema (use getDatabaseSchema tool).
            """.formatted(version, migrationSql));
    }
}
```

---

## 8. RestTemplate Bean

```java
package com.example.actuatormcp.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

@Configuration
public class AppConfig {
    
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

---

## 9. Testing với MCP Inspector

MCP Inspector là official testing tool từ Anthropic để test MCP servers interactively.

### Install và Run

```bash
# Install MCP Inspector
npm install -g @modelcontextprotocol/inspector

# Build Spring Boot JAR trước
cd your-mcp-server-project
mvn clean package -DskipTests

# Run Inspector với your server
mcp-inspector java -jar target/actuator-mcp-server-1.0.0.jar
```

Inspector mở browser UI tại `http://localhost:5173` với:
- **Tools tab**: List tất cả tools, test từng tool với sample inputs
- **Resources tab**: Browse và read resources
- **Prompts tab**: Test prompt templates
- **Logs**: Real-time JSON-RPC message log

### Test Cases trong Inspector

```
Test 1: getHealthStatus
  → Expected: JSON với status UP/DOWN cho từng component

Test 2: getApiEndpoints
  → Expected: List tất cả @RequestMapping endpoints

Test 3: getDatabaseMigrationStatus
  → Expected: List migrations với version và status

Test 4: executeQuery với SELECT
  Input: sql = "SELECT COUNT(*) as total FROM users"
  → Expected: Results table với count

Test 5: executeQuery với non-SELECT (security test)
  Input: sql = "DROP TABLE users"
  → Expected: Error message "Only SELECT queries are allowed"

Test 6: getRecentLogs
  Input: level = "ERROR", count = 20
  → Expected: Last 20 ERROR log lines

Test 7: getDatabaseSchema
  → Expected: Full schema với tất cả tables
```

### Unit Testing

```java
package com.example.actuatormcp.tools;

import com.example.actuatormcp.client.ActuatorClient;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class ActuatorToolsTest {
    
    @Mock
    private ActuatorClient actuatorClient;
    
    @InjectMocks
    private ActuatorTools actuatorTools;
    
    @Test
    void getHealthStatus_returnsActuatorResponse() {
        when(actuatorClient.getHealth()).thenReturn("{\"status\": \"UP\"}");
        
        String result = actuatorTools.getHealthStatus();
        
        assertThat(result).contains("UP");
    }
    
    @Test
    void getRecentLogs_validatesCountRange() {
        String result = actuatorTools.getRecentLogs("ERROR", 0);
        assertThat(result).contains("Error: count must be between 1 and 1000");
    }
    
    @Test
    void getRecentLogs_validatesLogLevel() {
        String result = actuatorTools.getRecentLogs("INVALID_LEVEL", 10);
        assertThat(result).contains("Invalid log level");
    }
}
```

```java
package com.example.actuatormcp.tools;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.jdbc.core.JdbcTemplate;

import static org.assertj.core.api.Assertions.assertThat;

@ExtendWith(MockitoExtension.class)
class DatabaseToolsSecurityTest {
    
    @Mock
    private JdbcTemplate jdbcTemplate;
    
    @InjectMocks
    private DatabaseTools databaseTools;
    
    @Test
    void executeQuery_blocksDropStatement() {
        String result = databaseTools.executeQuery("DROP TABLE users");
        assertThat(result).contains("Only SELECT");
    }
    
    @Test
    void executeQuery_blocksDeleteStatement() {
        String result = databaseTools.executeQuery("DELETE FROM users WHERE id = 1");
        assertThat(result).contains("Only SELECT");
    }
    
    @Test
    void executeQuery_blocksInsertStatement() {
        String result = databaseTools.executeQuery("INSERT INTO users VALUES (1, 'test')");
        assertThat(result).contains("Only SELECT");
    }
    
    @Test
    void executeQuery_blocksDropInSubquery() {
        String result = databaseTools.executeQuery(
            "SELECT 1; DROP TABLE users --");
        assertThat(result).contains("blocked keyword: DROP");
    }
}
```

---

## 10. Packaging và Deployment

### 10.1 Build Executable JAR

```bash
mvn clean package

# JAR sẽ ở: target/actuator-mcp-server-1.0.0.jar
# Verify:
java -jar target/actuator-mcp-server-1.0.0.jar --help
```

### 10.2 Config trong Claude Code

Tạo `.claude/settings.json` trong Spring Boot project của bạn:

```json
{
  "mcpServers": {
    "spring-actuator": {
      "command": "java",
      "args": [
        "-jar",
        "/Users/yourname/tools/actuator-mcp-server-1.0.0.jar"
      ],
      "env": {
        "DATABASE_URL": "postgresql://mcp_readonly:pass@localhost:5432/myapp_dev",
        "ACTUATOR_BASE_URL": "http://localhost:8080/actuator",
        "DB_USERNAME": "mcp_readonly",
        "DB_PASSWORD": "your_password"
      }
    }
  }
}
```

### 10.3 Team Deployment Script

```bash
#!/bin/bash
# deploy-mcp-server.sh

MCP_SERVER_VERSION="1.0.0"
MCP_SERVER_JAR="actuator-mcp-server-${MCP_SERVER_VERSION}.jar"
INSTALL_DIR="$HOME/.local/share/mcp-servers"

echo "Deploying Spring Boot Actuator MCP Server v${MCP_SERVER_VERSION}"

# Create install directory
mkdir -p "$INSTALL_DIR"

# Copy JAR
cp "target/${MCP_SERVER_JAR}" "$INSTALL_DIR/"

echo "Installed to: ${INSTALL_DIR}/${MCP_SERVER_JAR}"
echo ""
echo "Add to .claude/settings.json:"
echo '{
  "mcpServers": {
    "spring-actuator": {
      "command": "java",
      "args": ["-jar", "'"${INSTALL_DIR}/${MCP_SERVER_JAR}"'"],
      "env": {
        "DATABASE_URL": "YOUR_DB_URL",
        "ACTUATOR_BASE_URL": "http://localhost:8080/actuator"
      }
    }
  }
}'
```

---

## 11. Security Best Practices

### Database Security

```sql
-- Tạo dedicated read-only user cho MCP server
CREATE USER mcp_server WITH PASSWORD 'use_a_strong_random_password_here';
GRANT CONNECT ON DATABASE myapp_dev TO mcp_server;
GRANT USAGE ON SCHEMA public TO mcp_server;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO mcp_server;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO mcp_server;

-- KHÔNG grant: INSERT, UPDATE, DELETE, DROP, CREATE, TRUNCATE
-- KHÔNG dùng: superuser, app user, admin user
```

### Input Validation Pattern

```java
// Luôn validate tool inputs trước khi xử lý
@Tool(description = "...")
public String someTool(
    @ToolParam(description = "...") String userInput
) {
    // 1. Null/blank check
    if (userInput == null || userInput.isBlank()) {
        return "Error: Input cannot be empty";
    }
    
    // 2. Length limits
    if (userInput.length() > 10_000) {
        return "Error: Input too long (max 10,000 characters)";
    }
    
    // 3. Format validation (nếu applicable)
    if (!userInput.matches("[a-zA-Z0-9_\\-\\.]+")) {
        return "Error: Input contains invalid characters";
    }
    
    // 4. Business logic validation
    // ...
    
    // 5. Audit log
    log.info("Tool called with input: {}", userInput.substring(0, Math.min(100, userInput.length())));
    
    // 6. Execute
    return doActualWork(userInput);
}
```

### Sensitive Data Masking

```java
// Không expose passwords, tokens, keys trong resources hay tool responses
private String maskSensitiveData(String value) {
    // Mask JDBC passwords
    value = value.replaceAll(":[^@/]+@", ":***@");
    // Mask API keys pattern
    value = value.replaceAll("(key|token|secret|password)=[^&\\s]+", "$1=***");
    return value;
}
```

---

## 12. Exercise

### Exercise 3.1 — Implement Missing Tools

Implement 2 tools còn thiếu trong `ActuatorTools`:

**Tool 1: `getJvmMetrics`**
- Description: JVM memory usage (heap/non-heap), thread count, GC stats
- Source: `/actuator/metrics/jvm.memory.used` và related endpoints
- Format: Human-readable với percentages

**Tool 2: `getConnectionPoolStatus`**
- Description: HikariCP connection pool stats (active, idle, waiting, max)
- Source: `/actuator/metrics/hikaricp.connections.*`
- Alert nếu active connections > 80% of max

### Exercise 3.2 — Add Resource

Implement `McpResource` expose danh sách Flyway migrations dưới dạng structured markdown:

```
@McpResource(uri = "db://migrations", ...)
public String getMigrationsAsMarkdown() {
    // Query flyway_schema_history table trực tiếp
    // Format: | Version | Description | Installed On | Status |
}
```

### Exercise 3.3 — Security Audit

Tự review code bằng cách trả lời:

1. Tool nào có thể gây side effects ngoài ý muốn? Fix?
2. Input validation có gaps ở đâu?
3. Thông tin nhạy cảm nào có thể leak qua resources?
4. Nếu có attacker control được MCP client, họ có thể làm gì?

---

## Tóm tắt bài học

| Topic | Key Point |
|-------|-----------|
| Spring AI MCP | `spring-ai-mcp-server-spring-boot-starter` — auto-configure |
| stdio transport | stdout dành cho JSON-RPC, log ra stderr hoặc file |
| @Tool | Annotate methods để expose như MCP tools |
| @McpResource | Expose data như MCP resources theo URI |
| @McpPrompt | Expose instruction templates |
| Input validation | Validate tất cả tool params trước khi execute |
| SQL security | Chỉ cho phép SELECT, block tất cả write operations |
| DB user | Read-only user với minimal permissions |
| Testing | MCP Inspector + JUnit unit tests |
| Deployment | Executable JAR + config trong .claude/settings.json |

---

## Bài tiếp theo

**Capstone Project: JavaOps MCP Suite** — Build production-ready MCP server với 7 tools covering tất cả Spring Boot operational concerns. Đây là dự án bạn có thể đưa thẳng vào team.

---

*Bài 03/03 — Module 03: Model Context Protocol*
