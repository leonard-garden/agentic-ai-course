# Lesson 03: Tool Use (Function Calling)

> **Thời lượng**: 3 buổi (~7.5 giờ)  
> **Mục tiêu**: Hiểu sâu tool use flow, xây dựng Java AI service với multiple tools, handle edge cases trong production  
> **Tại sao quan trọng nhất**: Tool use là cơ chế biến Claude từ "chatbot" thành "AI agent" có khả năng thực thi actions trong hệ thống của bạn

---

## 1. Tool Use là gì — Mindset Shift

### 1.1 Vấn đề với pure LLM

Claude biết **rất nhiều** nhưng có giới hạn quan trọng:

```
❌ Claude không biết dữ liệu real-time của bạn:
   "Inventory của product ID 12345 là bao nhiêu?" → Claude không thể trả lời chính xác

❌ Claude không thể thực thi actions:
   "Tạo order cho customer này" → Claude chỉ nói "tôi sẽ tạo order" nhưng không làm được

❌ Claude có knowledge cutoff:
   "Giá cổ phiếu AAPL hôm nay?" → Claude không biết (training data cũ)
```

### 1.2 Tool Use giải quyết vấn đề này

Tool use cho phép Claude **gọi functions** trong code của bạn:

```
User: "Inventory của product 12345 còn bao nhiêu?"
         ↓
Claude nhận ra cần dữ liệu real-time
         ↓
Claude ra lệnh: "Gọi tool get_product_inventory(productId=12345)"
         ↓
Java code thực thi: query database → 42 units
         ↓
Claude nhận kết quả → trả lời: "Product 12345 còn 42 units trong kho"
```

### 1.3 Một điểm quan trọng về control flow

```
⚠️  Claude KHÔNG thực sự "gọi" functions.
    Claude CHỈ "nói" muốn gọi function nào với arguments gì.
    YOUR Java code phải thực thi function đó và trả kết quả lại.
```

Đây là distinction quan trọng — **bạn luôn kiểm soát** những gì được thực thi. Claude chỉ ra quyết định "orchestration", code của bạn thực hiện "execution".

---

## 2. Tool Use Flow Chi tiết

### 2.1 Complete flow diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    TOOL USE FLOW                             │
│                                                             │
│  1. User message ──────────────────────────────────────┐   │
│                                                         ↓   │
│  2. [POST /messages với tools definition]               │   │
│                        ↓                               │   │
│  3. Claude response: stop_reason = "tool_use"          │   │
│     content: [{type:"tool_use", name:"...", input:{}}] │   │
│                        ↓                               │   │
│  4. YOUR CODE: parse tool_use block                    │   │
│                        ↓                               │   │
│  5. YOUR CODE: execute actual Java function            │   │
│                        ↓                               │   │
│  6. [POST /messages với tool_result]                   │   │
│     messages: [...history, assistant_msg, tool_result] │   │
│                        ↓                               │   │
│  7. Claude processes tool result                       │   │
│                        ↓                               │   │
│  8. Claude final response: stop_reason = "end_turn"    │   │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Messages trong tool use — structure chi tiết

```json
// Step 1: Initial request
{
  "role": "user",
  "content": "What's the current price of product SKU-001?"
}

// Step 3: Claude response (tool_use)
{
  "role": "assistant",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_01A09q90qw90lq917835lq9",
      "name": "get_product_price",
      "input": {
        "sku": "SKU-001"
      }
    }
  ],
  "stop_reason": "tool_use"
}

// Step 6: Your tool result (sent as user message)
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01A09q90qw90lq917835lq9",
      "content": "{\"sku\": \"SKU-001\", \"price\": 29.99, \"currency\": \"USD\"}"
    }
  ]
}

// Step 8: Claude final response
{
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "The current price of product SKU-001 is $29.99 USD."
    }
  ],
  "stop_reason": "end_turn"
}
```

---

## 3. Defining Tools trong Java

### 3.1 Tool Structure

Mỗi tool cần:
- **`name`**: Tên function (snake_case, unique)
- **`description`**: Giải thích RÕRÀNG tool làm gì — Claude dùng cái này để quyết định có gọi không
- **`inputSchema`**: JSON Schema của parameters

```java
package com.example.lesson03;

import com.anthropic.sdk.models.*;
import com.anthropic.sdk.core.JsonValue;
import java.util.Map;

public class ToolDefinitions {

    /**
     * Tool 1: Search products in database
     */
    public static Tool searchProductsTool() {
        return Tool.builder()
            .name("search_products")
            .description("""
                Search for products in the e-commerce database by keyword.
                Returns a list of matching products with their details including
                id, name, price, stock quantity, and category.
                Use this when the user asks about products, inventory, or pricing.
                """)
            .inputSchema(Tool.InputSchema.builder()
                .type("object")
                .putProperty("query", JsonValue.from(Map.of(
                    "type", "string",
                    "description", "Search keyword to find products"
                )))
                .putProperty("category", JsonValue.from(Map.of(
                    "type", "string",
                    "description", "Optional: filter by category (electronics, clothing, etc.)",
                    "enum", List.of("electronics", "clothing", "food", "books", "other")
                )))
                .putProperty("max_results", JsonValue.from(Map.of(
                    "type", "integer",
                    "description", "Maximum number of results to return (default: 10, max: 50)",
                    "default", 10
                )))
                .addRequired("query") // 'query' là required, 'category' và 'max_results' optional
                .build())
            .build();
    }

    /**
     * Tool 2: Get order details
     */
    public static Tool getOrderDetailsTool() {
        return Tool.builder()
            .name("get_order_details")
            .description("""
                Retrieve detailed information about a specific order by order ID.
                Returns order status, items, customer info, shipping details, and payment status.
                Use this when user asks about a specific order status or details.
                """)
            .inputSchema(Tool.InputSchema.builder()
                .type("object")
                .putProperty("order_id", JsonValue.from(Map.of(
                    "type", "string",
                    "description", "The unique order identifier (format: ORD-XXXXX)"
                )))
                .addRequired("order_id")
                .build())
            .build();
    }

    /**
     * Tool 3: Get current weather (external API example)
     */
    public static Tool getWeatherTool() {
        return Tool.builder()
            .name("get_current_weather")
            .description("""
                Get current weather information for a specific city.
                Returns temperature, humidity, wind speed, and weather conditions.
                """)
            .inputSchema(Tool.InputSchema.builder()
                .type("object")
                .putProperty("city", JsonValue.from(Map.of(
                    "type", "string",
                    "description", "City name (e.g., 'Ho Chi Minh City', 'Hanoi')"
                )))
                .putProperty("unit", JsonValue.from(Map.of(
                    "type", "string",
                    "description", "Temperature unit",
                    "enum", List.of("celsius", "fahrenheit"),
                    "default", "celsius"
                )))
                .addRequired("city")
                .build())
            .build();
    }
}
```

### 3.2 Tool Description — Nghệ thuật quan trọng

Description tốt = Claude gọi đúng tool đúng lúc. Description tệ = Claude confused, gọi nhầm tool hoặc không gọi khi cần.

```java
// ❌ BAD description — quá ngắn, không rõ khi nào dùng
Tool badTool = Tool.builder()
    .name("search")
    .description("Search for things")
    .build();

// ✅ GOOD description — rõ mục đích, input, output, và khi nào dùng
Tool goodTool = Tool.builder()
    .name("search_products")
    .description("""
        Search for products in the product catalog database.
        
        Returns: list of products with fields:
        - id (string): unique product identifier
        - name (string): product display name
        - price (number): current price in USD
        - stock (integer): available inventory count
        - category (string): product category
        
        Use this tool when:
        - User asks about product availability
        - User wants to find products by name or keyword
        - User asks about pricing
        
        Do NOT use this for:
        - Order status queries (use get_order_details instead)
        - Customer information (use get_customer_profile instead)
        """)
    .build();
```

---

## 4. Implementing the Tool Use Loop

### 4.1 Core Tool Use Engine

```java
package com.example.lesson03;

import com.anthropic.sdk.AnthropicClient;
import com.anthropic.sdk.models.*;
import com.anthropic.sdk.core.JsonValue;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.util.*;

public class ToolUseEngine {

    private static final Logger log = LoggerFactory.getLogger(ToolUseEngine.class);
    private static final ObjectMapper objectMapper = new ObjectMapper();

    private final AnthropicClient client;
    private final Map<String, ToolHandler> toolHandlers;
    private final List<Tool> tools;

    /**
     * ToolHandler interface — implement này cho mỗi tool
     */
    @FunctionalInterface
    public interface ToolHandler {
        String execute(JsonNode input) throws Exception;
    }

    public ToolUseEngine(AnthropicClient client) {
        this.client = client;
        this.toolHandlers = new HashMap<>();
        this.tools = new ArrayList<>();
    }

    /**
     * Đăng ký một tool với handler
     */
    public void registerTool(Tool tool, ToolHandler handler) {
        tools.add(tool);
        toolHandlers.put(tool.name(), handler);
        log.info("Registered tool: {}", tool.name());
    }

    /**
     * Main agentic loop — chạy cho đến khi Claude không còn gọi tool nào
     */
    public String run(String userMessage) {
        List<MessageParam> messages = new ArrayList<>();
        messages.add(MessageParam.builder()
            .role(MessageParam.Role.USER)
            .content(userMessage)
            .build());

        int maxIterations = 10; // Giới hạn để tránh infinite loop
        int iteration = 0;

        while (iteration < maxIterations) {
            iteration++;
            log.debug("Tool use iteration {}/{}", iteration, maxIterations);

            // Gọi Claude với tools
            Message response = client.messages().create(
                MessageCreateParams.builder()
                    .model(Model.CLAUDE_SONNET_4_6)
                    .maxTokens(4096)
                    .tools(tools)
                    .messages(messages)
                    .build()
            );

            // Thêm Claude response vào history
            messages.add(MessageParam.builder()
                .role(MessageParam.Role.ASSISTANT)
                .content(response.content())
                .build());

            // Kiểm tra stop reason
            String stopReason = response.stopReason().toString();
            log.debug("Stop reason: {}", stopReason);

            if ("end_turn".equals(stopReason)) {
                // Claude đã hoàn thành — extract final text response
                return extractText(response);
            }

            if ("tool_use".equals(stopReason)) {
                // Claude muốn gọi tools — process tất cả tool calls
                List<ContentBlockParam> toolResults = processToolCalls(response);

                // Thêm tool results vào history như một user message
                messages.add(MessageParam.builder()
                    .role(MessageParam.Role.USER)
                    .content(toolResults)
                    .build());
                // Loop lại — Claude sẽ xử lý kết quả và có thể gọi thêm tools
            } else {
                log.warn("Unexpected stop reason: {}", stopReason);
                break;
            }
        }

        log.warn("Reached max iterations ({}). Returning partial response.", maxIterations);
        return extractText(/* last message */ null);
    }

    /**
     * Process tất cả tool_use blocks trong response
     */
    private List<ContentBlockParam> processToolCalls(Message response) {
        List<ContentBlockParam> toolResults = new ArrayList<>();

        for (ContentBlock block : response.content()) {
            if (!(block instanceof ContentBlock.ToolUseBlock toolUseBlock)) continue;

            String toolName = toolUseBlock.name();
            String toolUseId = toolUseBlock.id();
            JsonValue inputJson = toolUseBlock.input();

            log.info("Executing tool: {} with input: {}", toolName, inputJson);

            String resultContent;
            boolean isError = false;

            try {
                ToolHandler handler = toolHandlers.get(toolName);
                if (handler == null) {
                    throw new IllegalArgumentException("Unknown tool: " + toolName);
                }

                // Parse input JSON
                JsonNode inputNode = objectMapper.readTree(inputJson.toString());

                // Execute tool
                resultContent = handler.execute(inputNode);
                log.info("Tool {} completed successfully", toolName);

            } catch (Exception e) {
                log.error("Tool {} failed: {}", toolName, e.getMessage(), e);
                resultContent = "Error executing " + toolName + ": " + e.getMessage();
                isError = true;
            }

            // Build tool_result block
            ToolResultBlockParam.Builder resultBuilder = ToolResultBlockParam.builder()
                .toolUseId(toolUseId)
                .content(resultContent);

            if (isError) {
                resultBuilder.isError(true);
            }

            toolResults.add(ContentBlockParam.ofToolResult(resultBuilder.build()));
        }

        return toolResults;
    }

    private String extractText(Message message) {
        if (message == null) return "Processing incomplete.";
        return message.content().stream()
            .filter(b -> b instanceof ContentBlock.TextBlock)
            .map(b -> ((ContentBlock.TextBlock) b).text())
            .findFirst()
            .orElse("No text response.");
    }
}
```

### 4.2 Implementing Tool Handlers

```java
package com.example.lesson03;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.*;

public class ToolHandlers {

    private static final ObjectMapper mapper = new ObjectMapper();

    // Simulated database — trong thực tế dùng JPA/JDBC
    private static final List<Map<String, Object>> PRODUCTS = List.of(
        Map.of("id", "P001", "name", "MacBook Pro 14\"", "price", 1999.99,
               "stock", 15, "category", "electronics"),
        Map.of("id", "P002", "name", "iPhone 15 Pro", "price", 999.99,
               "stock", 42, "category", "electronics"),
        Map.of("id", "P003", "name", "Java Programming Book", "price", 45.99,
               "stock", 100, "category", "books"),
        Map.of("id", "P004", "name", "Spring Boot T-Shirt", "price", 29.99,
               "stock", 0, "category", "clothing")
    );

    /**
     * Handler cho search_products tool
     */
    public static String searchProducts(JsonNode input) throws Exception {
        String query = input.get("query").asText().toLowerCase();
        String category = input.has("category") ? input.get("category").asText() : null;
        int maxResults = input.has("max_results") ? input.get("max_results").asInt() : 10;

        List<Map<String, Object>> results = PRODUCTS.stream()
            .filter(p -> ((String) p.get("name")).toLowerCase().contains(query)
                || ((String) p.get("category")).toLowerCase().contains(query))
            .filter(p -> category == null || category.equals(p.get("category")))
            .limit(maxResults)
            .toList();

        Map<String, Object> response = new LinkedHashMap<>();
        response.put("query", query);
        response.put("count", results.size());
        response.put("products", results);

        return mapper.writeValueAsString(response);
    }

    /**
     * Handler cho get_order_details tool
     */
    public static String getOrderDetails(JsonNode input) throws Exception {
        String orderId = input.get("order_id").asText();

        // Validate format
        if (!orderId.matches("ORD-\\d{5}")) {
            throw new IllegalArgumentException(
                "Invalid order ID format. Expected: ORD-XXXXX, got: " + orderId);
        }

        // Simulated order lookup
        if ("ORD-12345".equals(orderId)) {
            Map<String, Object> order = new LinkedHashMap<>();
            order.put("order_id", orderId);
            order.put("status", "SHIPPED");
            order.put("customer", Map.of("name", "Nguyen Van A", "email", "nguyenvana@example.com"));
            order.put("items", List.of(
                Map.of("product_id", "P001", "name", "MacBook Pro 14\"",
                       "quantity", 1, "price", 1999.99)
            ));
            order.put("total", 1999.99);
            order.put("shipping_address", "123 Le Loi, Q1, HCMC");
            order.put("tracking_number", "VN123456789");
            order.put("estimated_delivery", "2024-01-15");
            return mapper.writeValueAsString(order);
        }

        throw new IllegalArgumentException("Order not found: " + orderId);
    }

    /**
     * Handler cho get_current_weather tool — gọi external API
     */
    public static String getCurrentWeather(JsonNode input) throws Exception {
        String city = input.get("city").asText();
        String unit = input.has("unit") ? input.get("unit").asText() : "celsius";

        // Trong thực tế: gọi OpenWeather API hoặc tương tự
        // Ở đây simulate response
        Map<String, Object> weather = new LinkedHashMap<>();
        weather.put("city", city);
        weather.put("temperature", unit.equals("celsius") ? 28 : 82);
        weather.put("unit", unit);
        weather.put("humidity", "75%");
        weather.put("wind_speed", "15 km/h");
        weather.put("conditions", "Partly cloudy");
        weather.put("timestamp", System.currentTimeMillis());

        return mapper.writeValueAsString(weather);
    }
}
```

### 4.3 Wiring Everything Together

```java
package com.example.lesson03;

import com.anthropic.sdk.AnthropicClient;

public class EcommerceAssistant {

    public static void main(String[] args) {
        AnthropicClient client = AnthropicClient.builder()
            .apiKey(System.getenv("ANTHROPIC_API_KEY"))
            .build();

        // Tạo engine và register tools
        ToolUseEngine engine = new ToolUseEngine(client);

        engine.registerTool(
            ToolDefinitions.searchProductsTool(),
            ToolHandlers::searchProducts
        );

        engine.registerTool(
            ToolDefinitions.getOrderDetailsTool(),
            ToolHandlers::getOrderDetails
        );

        engine.registerTool(
            ToolDefinitions.getWeatherTool(),
            ToolHandlers::getCurrentWeather
        );

        // Test queries
        String[] queries = {
            "Tôi muốn tìm laptop trong cửa hàng",
            "Order ORD-12345 của tôi đang ở đâu rồi?",
            "Thời tiết Hà Nội hôm nay thế nào?",
            "Còn sách nào về Java không? Và thời tiết HCMC hôm nay?"  // Multi-tool
        };

        for (String query : queries) {
            System.out.println("\n" + "=".repeat(60));
            System.out.println("User: " + query);
            System.out.println("-".repeat(60));

            String response = engine.run(query);
            System.out.println("Assistant: " + response);
        }
    }
}
```

**Expected output:**
```
============================================================
User: Tôi muốn tìm laptop trong cửa hàng
------------------------------------------------------------
[INFO] Executing tool: search_products with input: {"query":"laptop"}
[INFO] Tool search_products completed successfully
Assistant: Tôi tìm thấy 1 sản phẩm laptop trong cửa hàng:
- **MacBook Pro 14"** (ID: P001) - Giá: $1,999.99 - Còn hàng: 15 chiếc

============================================================
User: Còn sách nào về Java không? Và thời tiết HCMC hôm nay?
------------------------------------------------------------
[INFO] Executing tool: search_products with input: {"query":"java","category":"books"}
[INFO] Executing tool: get_current_weather with input: {"city":"HCMC"}
Assistant: Về sách Java: Có 1 cuốn "Java Programming Book" ($45.99, còn 100 cuốn).
Thời tiết HCMC hôm nay: 28°C, ẩm độ 75%, Partly cloudy.
```

---

## 5. Parallel Tool Calls

Claude có thể request nhiều tools **cùng lúc** trong một response. SDK của bạn phải handle điều này:

```java
/**
 * Process tool calls với parallel execution
 * Hiệu quả hơn khi có nhiều tool calls độc lập
 */
private List<ContentBlockParam> processToolCallsParallel(Message response) {
    // Tìm tất cả tool_use blocks
    List<ContentBlock.ToolUseBlock> toolCalls = response.content().stream()
        .filter(b -> b instanceof ContentBlock.ToolUseBlock)
        .map(b -> (ContentBlock.ToolUseBlock) b)
        .toList();

    if (toolCalls.isEmpty()) return List.of();

    // Execute tất cả tools song song
    List<CompletableFuture<ContentBlockParam>> futures = toolCalls.stream()
        .map(toolCall -> CompletableFuture.supplyAsync(() -> {
            String toolName = toolCall.name();
            String toolUseId = toolCall.id();
            JsonValue inputJson = toolCall.input();

            log.info("Executing tool (parallel): {}", toolName);

            try {
                ToolHandler handler = toolHandlers.get(toolName);
                if (handler == null) {
                    throw new IllegalArgumentException("Unknown tool: " + toolName);
                }
                JsonNode inputNode = objectMapper.readTree(inputJson.toString());
                String result = handler.execute(inputNode);

                return ContentBlockParam.ofToolResult(
                    ToolResultBlockParam.builder()
                        .toolUseId(toolUseId)
                        .content(result)
                        .build()
                );
            } catch (Exception e) {
                log.error("Tool {} failed: {}", toolName, e.getMessage());
                return ContentBlockParam.ofToolResult(
                    ToolResultBlockParam.builder()
                        .toolUseId(toolUseId)
                        .content("Error: " + e.getMessage())
                        .isError(true)
                        .build()
                );
            }
        }, toolExecutorService))
        .toList();

    // Chờ tất cả xong
    return futures.stream()
        .map(CompletableFuture::join)
        .toList();
}
```

---

## 6. Tool Choice Control

Đôi khi bạn muốn **force** Claude gọi một tool cụ thể, hoặc không gọi tool nào:

```java
// Mặc định: Claude tự quyết định có gọi tool không
MessageCreateParams autoParams = MessageCreateParams.builder()
    .tools(tools)
    .toolChoice(ToolChoiceAuto.builder().build()) // Default
    .build();

// Force gọi ít nhất 1 tool (bất kỳ tool nào)
MessageCreateParams anyToolParams = MessageCreateParams.builder()
    .tools(tools)
    .toolChoice(ToolChoiceAny.builder().build())
    .build();

// Force gọi một tool cụ thể
MessageCreateParams specificToolParams = MessageCreateParams.builder()
    .tools(tools)
    .toolChoice(ToolChoiceTool.builder()
        .name("search_products") // Phải gọi tool này
        .build())
    .build();

// Không cho phép gọi tools (chỉ text response)
MessageCreateParams noToolParams = MessageCreateParams.builder()
    .tools(tools)
    .toolChoice(ToolChoiceNone.builder().build())
    .build();
```

**Khi nào dùng force tool:**
- Structured data extraction: force gọi `extract_data` tool để Claude trả về JSON
- Validation: muốn Claude luôn "submit" form thay vì chỉ nói
- Testing: đảm bảo tool được gọi trong integration tests

---

## 7. Error Handling khi Tool Fails

### 7.1 Graceful error response

```java
// Khi tool fail, gửi error về Claude thay vì crash
try {
    String result = handler.execute(inputNode);
    return ContentBlockParam.ofToolResult(
        ToolResultBlockParam.builder()
            .toolUseId(toolUseId)
            .content(result)
            .build()
    );
} catch (DatabaseException e) {
    // Database error — Claude sẽ nhận được và có thể suggest user retry
    return ContentBlockParam.ofToolResult(
        ToolResultBlockParam.builder()
            .toolUseId(toolUseId)
            .content("""
                {
                  "error": "database_unavailable",
                  "message": "Unable to query database at this time",
                  "suggestion": "Please try again in a moment"
                }
                """)
            .isError(true)
            .build()
    );
} catch (ValidationException e) {
    // Validation error — Claude có thể sửa input và retry
    return ContentBlockParam.ofToolResult(
        ToolResultBlockParam.builder()
            .toolUseId(toolUseId)
            .content(String.format("""
                {
                  "error": "validation_error",
                  "message": "%s",
                  "valid_formats": ["ORD-12345", "ORD-99999"]
                }
                """, e.getMessage()))
            .isError(true)
            .build()
    );
}
```

### 7.2 Claude behavior khi nhận error

Khi Claude nhận `isError=true`, nó sẽ:
- Explain lỗi cho user một cách friendly
- Suggest cách fix (nếu có thể)
- Trong một số cases, có thể retry với corrected parameters

---

## 8. Full Example: Java AI Database Assistant

### 8.1 Architecture

```
User Input (natural language)
       ↓
  Spring Boot Controller
       ↓
  ToolUseEngine (Claude orchestrator)
       ↓
  [Tool: execute_sql_query] → JdbcTemplate → PostgreSQL
  [Tool: get_table_schema] → DatabaseMetaData
  [Tool: explain_query_plan] → EXPLAIN ANALYZE
       ↓
  Claude synthesizes results
       ↓
  Natural language response
```

### 8.2 Production-ready implementation

```java
package com.example.lesson03.assistant;

import com.anthropic.sdk.AnthropicClient;
import com.anthropic.sdk.models.*;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;
import java.util.*;

@Service
public class DatabaseAssistantService {

    private static final Logger log = LoggerFactory.getLogger(DatabaseAssistantService.class);

    private final AnthropicClient anthropicClient;
    private final JdbcTemplate jdbcTemplate;
    private final ToolUseEngine engine;
    private static final ObjectMapper mapper = new ObjectMapper();

    // QUAN TRỌNG: Chỉ cho phép SELECT, bảo vệ khỏi SQL injection intent
    private static final List<String> ALLOWED_SQL_PREFIXES =
        List.of("SELECT", "EXPLAIN", "WITH");

    public DatabaseAssistantService(AnthropicClient anthropicClient,
                                    JdbcTemplate jdbcTemplate) {
        this.anthropicClient = anthropicClient;
        this.jdbcTemplate = jdbcTemplate;
        this.engine = new ToolUseEngine(anthropicClient);
        registerTools();
    }

    private void registerTools() {
        // Tool 1: Execute SQL query
        engine.registerTool(
            buildExecuteSqlTool(),
            this::executeSqlQuery
        );

        // Tool 2: Get table schema info
        engine.registerTool(
            buildGetSchemaTool(),
            this::getTableSchema
        );

        // Tool 3: List available tables
        engine.registerTool(
            buildListTablesTool(),
            this::listTables
        );
    }

    public String ask(String naturalLanguageQuestion) {
        return engine.run(naturalLanguageQuestion);
    }

    // ── Tool definitions ──────────────────────────────────────

    private Tool buildExecuteSqlTool() {
        return Tool.builder()
            .name("execute_sql_query")
            .description("""
                Execute a read-only SQL SELECT query against the database.
                Only SELECT and EXPLAIN statements are allowed.
                Returns the query results as JSON array.
                Use this to answer data questions that require actual database queries.
                Maximum 100 rows returned.
                """)
            .inputSchema(Tool.InputSchema.builder()
                .type("object")
                .putProperty("sql", JsonValue.from(Map.of(
                    "type", "string",
                    "description", "The SQL SELECT query to execute"
                )))
                .putProperty("description", JsonValue.from(Map.of(
                    "type", "string",
                    "description", "Human-readable description of what this query does"
                )))
                .addRequired("sql")
                .addRequired("description")
                .build())
            .build();
    }

    private Tool buildGetSchemaTool() {
        return Tool.builder()
            .name("get_table_schema")
            .description("""
                Get the column definitions and constraints for a specific database table.
                Use this before writing queries to understand the table structure.
                """)
            .inputSchema(Tool.InputSchema.builder()
                .type("object")
                .putProperty("table_name", JsonValue.from(Map.of(
                    "type", "string",
                    "description", "Name of the table to inspect"
                )))
                .addRequired("table_name")
                .build())
            .build();
    }

    private Tool buildListTablesTool() {
        return Tool.builder()
            .name("list_tables")
            .description("""
                List all available tables in the database with their row counts.
                Use this first to understand what data is available before writing queries.
                """)
            .inputSchema(Tool.InputSchema.builder()
                .type("object")
                .build()) // No required params
            .build();
    }

    // ── Tool handlers ──────────────────────────────────────────

    private String executeSqlQuery(JsonNode input) throws Exception {
        String sql = input.get("sql").asText().trim().toUpperCase();
        String description = input.get("description").asText();

        // Security: chỉ cho phép SELECT/EXPLAIN
        boolean isAllowed = ALLOWED_SQL_PREFIXES.stream()
            .anyMatch(sql::startsWith);
        if (!isAllowed) {
            throw new SecurityException(
                "Only SELECT queries are allowed. Got: " + sql.substring(0, 10));
        }

        log.info("Executing query: {} ({})", description, sql);

        // Execute query với limit
        String limitedSql = input.get("sql").asText();
        if (!limitedSql.toUpperCase().contains("LIMIT")) {
            limitedSql += " LIMIT 100";
        }

        List<Map<String, Object>> rows = jdbcTemplate.queryForList(limitedSql);

        Map<String, Object> result = new LinkedHashMap<>();
        result.put("query", input.get("sql").asText());
        result.put("description", description);
        result.put("row_count", rows.size());
        result.put("rows", rows);

        return mapper.writeValueAsString(result);
    }

    private String getTableSchema(JsonNode input) throws Exception {
        String tableName = input.get("table_name").asText();

        List<Map<String, Object>> columns = jdbcTemplate.queryForList("""
            SELECT column_name, data_type, is_nullable, column_default,
                   character_maximum_length
            FROM information_schema.columns
            WHERE table_name = ? AND table_schema = 'public'
            ORDER BY ordinal_position
            """, tableName);

        if (columns.isEmpty()) {
            throw new IllegalArgumentException("Table not found: " + tableName);
        }

        Map<String, Object> schema = new LinkedHashMap<>();
        schema.put("table_name", tableName);
        schema.put("columns", columns);

        return mapper.writeValueAsString(schema);
    }

    private String listTables(JsonNode input) throws Exception {
        List<Map<String, Object>> tables = jdbcTemplate.queryForList("""
            SELECT t.table_name,
                   (SELECT COUNT(*) FROM information_schema.columns c
                    WHERE c.table_name = t.table_name) as column_count
            FROM information_schema.tables t
            WHERE t.table_schema = 'public'
            AND t.table_type = 'BASE TABLE'
            ORDER BY t.table_name
            """);

        return mapper.writeValueAsString(Map.of("tables", tables));
    }
}
```

### 8.3 Spring Boot Controller

```java
@RestController
@RequestMapping("/api/database-assistant")
public class DatabaseAssistantController {

    private final DatabaseAssistantService assistantService;

    public DatabaseAssistantController(DatabaseAssistantService assistantService) {
        this.assistantService = assistantService;
    }

    @PostMapping("/ask")
    public ResponseEntity<AssistantResponse> ask(@RequestBody AssistantRequest request) {
        if (request.question() == null || request.question().isBlank()) {
            return ResponseEntity.badRequest()
                .body(new AssistantResponse(null, "Question cannot be empty"));
        }

        String answer = assistantService.ask(request.question());
        return ResponseEntity.ok(new AssistantResponse(answer, null));
    }

    record AssistantRequest(String question) {}
    record AssistantResponse(String answer, String error) {}
}
```

**Test:**
```bash
curl -X POST http://localhost:8080/api/database-assistant/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Show me the top 5 customers by total order value this month"}'

# Response:
{
  "answer": "Đây là top 5 khách hàng theo giá trị đơn hàng tháng này:\n
  1. Nguyen Van A - $5,234.99 (8 orders)\n
  2. Tran Thi B - $3,891.50 (12 orders)\n
  ..."
}
```

---

## 9. Practical Project — Mini AI Database Assistant

### Exercise: Hoàn thiện DatabaseAssistantService

**Yêu cầu:**
1. Implement đầy đủ 3 tool handlers (một phần đã có trong ví dụ trên)
2. Thêm tool thứ 4: `get_query_plan` — chạy `EXPLAIN ANALYZE` và parse kết quả
3. Thêm system prompt hướng dẫn Claude:
   - Luôn gọi `list_tables` trước khi viết query lần đầu
   - Luôn verify bảng tồn tại trước khi query
   - Format kết quả dạng readable table (không phải raw JSON)
4. Test với 5 queries tự nhiên:
   - "Có bao nhiêu users đăng ký trong tuần này?"
   - "Top 3 products bán chạy nhất?"
   - "Order nào chưa được ship?"
   - "Doanh thu trung bình mỗi ngày trong tháng trước?"
   - "Khách hàng nào có nhiều orders nhất?"

---

## Tóm tắt Lesson 03

| Khái niệm | Key Points |
|-----------|-----------|
| Tool use flow | User → Claude (tool_use) → Java executes → tool_result → Claude final |
| Tool definition | name + description (quan trọng) + JSON Schema input |
| Tool handler | Java function nhận JsonNode, trả String (JSON) |
| Agentic loop | Lặp cho đến khi stop_reason = "end_turn" |
| Parallel tools | Claude có thể request nhiều tools cùng lúc — handle concurrently |
| Tool choice | auto / any / specific / none — control khi nào tools được dùng |
| Error handling | isError=true — Claude nhận lỗi và explain/retry gracefully |
| Security | Validate inputs, chỉ cho phép safe operations (SELECT, no DELETE) |

**Tiếp theo**: [Lesson 04 — Prompt Engineering](./04-prompt-engineering.md) — Làm thế nào để viết prompts hiệu quả, systematic testing, và structured output trong Java.
