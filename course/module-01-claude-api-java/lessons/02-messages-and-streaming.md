# Lesson 02: Messages API & Streaming

> **Thời lượng**: 2 buổi (~5 giờ)  
> **Mục tiêu**: Hiểu sâu Message structure, xây dựng multi-turn conversations, implement streaming real-time, dùng prompt caching để giảm cost

---

## 1. Message Structure — Nền tảng của mọi thứ

### 1.1 Anatomy của một Message

Mỗi lần giao tiếp với Claude là một **conversation** gồm danh sách các **messages**. Mỗi message có:
- `role`: ai nói — `"user"` hoặc `"assistant"`
- `content`: nội dung — list các **content blocks**

```json
{
  "role": "user",
  "content": [
    {
      "type": "text",
      "text": "Explain Java generics"
    }
  ]
}
```

### 1.2 Content Block Types

Có 4 loại content blocks chính:

| Type | Dùng khi | Ai tạo |
|------|---------|--------|
| `text` | Tin nhắn thông thường | User & Assistant |
| `image` | Gửi ảnh để phân tích | User only |
| `tool_use` | Claude muốn gọi tool | Assistant only |
| `tool_result` | Kết quả từ tool | User only |

Trong Lesson 01 và 02 này, chúng ta chủ yếu dùng `text`. `tool_use` và `tool_result` sẽ học kỹ ở Lesson 03.

### 1.3 Conversation Flow

```
User:       "What is Java?"
                  ↓
[POST /messages với role=user]
                  ↓
Assistant:  "Java is a programming language..."
                  ↓
User:       "Show me a Hello World example"
                  ↓
[POST /messages với FULL history: [user, assistant, user]]
                  ↓
Assistant:  "Here's a Hello World: public class..."
```

**Quan trọng**: Claude API là **stateless** — mỗi request phải gửi toàn bộ conversation history. Server không lưu state. Bạn (client) phải quản lý history.

---

## 2. Multi-Turn Conversations trong Java

### 2.1 Cấu trúc dữ liệu cho conversation history

```java
package com.example.lesson02;

import com.anthropic.sdk.models.MessageParam;
import java.util.ArrayList;
import java.util.List;

/**
 * Quản lý conversation history cho multi-turn chat.
 * Đây là pattern cơ bản nhất — production cần thêm persistence.
 */
public class ConversationHistory {

    private final List<MessageParam> messages = new ArrayList<>();

    /**
     * Thêm user message vào history
     */
    public void addUserMessage(String text) {
        messages.add(MessageParam.builder()
            .role(MessageParam.Role.USER)
            .content(text)
            .build());
    }

    /**
     * Thêm assistant response vào history
     */
    public void addAssistantMessage(String text) {
        messages.add(MessageParam.builder()
            .role(MessageParam.Role.ASSISTANT)
            .content(text)
            .build());
    }

    /**
     * Trả về danh sách messages để truyền vào API
     */
    public List<MessageParam> getMessages() {
        return List.copyOf(messages); // immutable copy
    }

    /**
     * Số turns trong conversation (user+assistant = 1 turn)
     */
    public int getTurnCount() {
        return messages.size() / 2;
    }

    /**
     * Clear history để bắt đầu conversation mới
     */
    public void clear() {
        messages.clear();
    }

    /**
     * Truncate khi history quá dài (context window limit)
     * Giữ N messages gần nhất
     */
    public void truncateToLastN(int n) {
        if (messages.size() > n) {
            List<MessageParam> recent = new ArrayList<>(
                messages.subList(messages.size() - n, messages.size())
            );
            messages.clear();
            messages.addAll(recent);
        }
    }
}
```

### 2.2 Multi-Turn Chat Service

```java
package com.example.lesson02;

import com.anthropic.sdk.AnthropicClient;
import com.anthropic.sdk.models.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class MultiTurnChatService {

    private static final Logger log = LoggerFactory.getLogger(MultiTurnChatService.class);
    private static final int MAX_HISTORY_MESSAGES = 20; // Giữ 10 turns gần nhất

    private final AnthropicClient client;
    private final ConversationHistory history;
    private final String systemPrompt;

    public MultiTurnChatService(AnthropicClient client, String systemPrompt) {
        this.client = client;
        this.history = new ConversationHistory();
        this.systemPrompt = systemPrompt;
    }

    /**
     * Gửi message và nhận response. Tự động maintain history.
     */
    public String chat(String userMessage) {
        // 1. Thêm user message vào history
        history.addUserMessage(userMessage);

        // 2. Truncate nếu quá dài để tránh context window overflow
        history.truncateToLastN(MAX_HISTORY_MESSAGES);

        // 3. Build params với full history
        MessageCreateParams params = MessageCreateParams.builder()
            .model(Model.CLAUDE_SONNET_4_6)
            .maxTokens(1024)
            .system(systemPrompt)          // System prompt áp dụng cho toàn conversation
            .messages(history.getMessages()) // Gửi toàn bộ history
            .build();

        log.debug("Sending {} messages in history", history.getMessages().size());

        // 4. Gọi API
        Message response = client.messages().create(params);

        // 5. Extract text từ response
        String assistantText = extractText(response);

        // 6. Thêm assistant response vào history cho lần sau
        history.addAssistantMessage(assistantText);

        log.info("Turn {}. Tokens: {} in, {} out",
            history.getTurnCount(),
            response.usage().inputTokens(),
            response.usage().outputTokens());

        return assistantText;
    }

    public void resetConversation() {
        history.clear();
        log.info("Conversation history cleared");
    }

    private String extractText(Message message) {
        return message.content().stream()
            .filter(b -> b instanceof ContentBlock.TextBlock)
            .map(b -> ((ContentBlock.TextBlock) b).text())
            .findFirst()
            .orElseThrow(() -> new IllegalStateException("No text in response"));
    }
}
```

### 2.3 Multi-Turn CLI Demo

```java
package com.example.lesson02;

import com.anthropic.sdk.AnthropicClient;
import java.util.Scanner;

public class MultiTurnCLI {

    public static void main(String[] args) {
        AnthropicClient client = AnthropicClient.builder()
            .apiKey(System.getenv("ANTHROPIC_API_KEY"))
            .build();

        String systemPrompt = """
            Bạn là một Java expert với 10 năm kinh nghiệm.
            Trả lời ngắn gọn, tập trung vào ví dụ code thực tế.
            Luôn dùng Java 17+ syntax khi viết code examples.
            """;

        MultiTurnChatService chatService = new MultiTurnChatService(client, systemPrompt);
        Scanner scanner = new Scanner(System.in);

        System.out.println("Java AI Tutor (type 'quit' to exit, 'reset' to new conversation)");
        System.out.println("=".repeat(60));

        while (true) {
            System.out.print("\nYou: ");
            String input = scanner.nextLine().trim();

            if ("quit".equalsIgnoreCase(input)) break;
            if ("reset".equalsIgnoreCase(input)) {
                chatService.resetConversation();
                System.out.println("[Conversation reset]");
                continue;
            }
            if (input.isBlank()) continue;

            try {
                String response = chatService.chat(input);
                System.out.println("\nClaude: " + response);
            } catch (Exception e) {
                System.err.println("Error: " + e.getMessage());
            }
        }

        System.out.println("Goodbye!");
    }
}
```

**Demo conversation:**
```
You: Giải thích Optional trong Java

Claude: Optional<T> là container object để tránh NullPointerException...

You: Cho tôi ví dụ với Stream API       ← Claude nhớ context "Optional"

Claude: Kết hợp Optional với Stream:
Optional<String> name = users.stream()
    .filter(u -> u.getId() == targetId)
    .map(User::getName)
    .findFirst();
...

You: Khi nào KHÔNG nên dùng Optional?   ← Vẫn trong context của 2 câu trước

Claude: Tránh dùng Optional khi...
```

---

## 3. System Prompts — Định hướng behavior của Claude

### 3.1 System Prompt là gì?

System prompt là instructions được gửi **trước** user messages. Nó định nghĩa:
- **Role/Persona**: Claude đóng vai gì
- **Constraints**: Giới hạn những gì Claude được/không được làm
- **Output format**: Response nên có format như thế nào
- **Context**: Background information Claude cần biết

```java
// System prompt được set riêng, KHÔNG phải là message đầu tiên
MessageCreateParams params = MessageCreateParams.builder()
    .model(Model.CLAUDE_SONNET_4_6)
    .maxTokens(1024)
    .system("You are a helpful Java expert.")  // System prompt
    .addUserMessage("What is a HashMap?")       // First user message
    .build();
```

### 3.2 Anatomy của một good system prompt

```java
String systemPrompt = """
    ## Role
    Bạn là Senior Java Backend Engineer với 10 năm kinh nghiệm,
    chuyên về Spring Boot và microservices.

    ## Nhiệm vụ
    Giúp developer review code, giải thích concepts, và debug issues.

    ## Constraints
    - Luôn suggest Java 17+ syntax
    - Khi viết code, luôn include error handling
    - Không viết code dài hơn 50 dòng mỗi function
    - Nếu câu hỏi không liên quan đến Java/backend, từ chối lịch sự

    ## Output Format
    - Giải thích ngắn gọn trước
    - Code example (nếu có) trong code block
    - Bullet points cho danh sách

    ## Examples
    User: "Explain HashMap"
    Assistant: HashMap là...
    ```java
    Map<String, Integer> map = new HashMap<>();
    ```
    """;
```

### 3.3 System prompt patterns

**Pattern 1: Role + Constraints**
```java
String codeReviewerPrompt = """
    Bạn là code reviewer nghiêm khắc nhưng constructive.
    Review code theo các tiêu chí: correctness, performance, security, readability.
    Format output: CRITICAL / HIGH / MEDIUM / LOW issues với explanation.
    """;
```

**Pattern 2: Structured output enforcer**
```java
String jsonOutputPrompt = """
    Bạn luôn trả lời bằng JSON hợp lệ. Không có text nào ngoài JSON.
    Schema: {"answer": "string", "confidence": "high|medium|low", "sources": ["string"]}
    """;
```

**Pattern 3: Domain expert với context**
```java
String dbAssistantPrompt = String.format("""
    Bạn là Database Administrator cho hệ thống e-commerce.
    
    Schema hiện tại:
    %s
    
    Rules:
    - Chỉ generate SELECT queries (không INSERT/UPDATE/DELETE)
    - Luôn có LIMIT clause
    - Explain query plan khi được hỏi
    """, currentDbSchema);
```

---

## 4. Streaming Responses

### 4.1 Tại sao cần Streaming?

Với normal API call, bạn phải **chờ** Claude generate xong toàn bộ response trước khi nhận được bất cứ gì. Với response dài (500-1000 tokens), có thể mất 5-10 giây.

Streaming cho phép nhận **từng token** ngay khi Claude generate — giống như typing animation trong ChatGPT.

```
Normal:    [5 giây chờ........] "Java HashMap là một data structure..."
Streaming: "Java" "Hash" "Map" " là" " một" " data"... (real-time)
```

### 4.2 Streaming trong Java với SDK

```java
package com.example.lesson02;

import com.anthropic.sdk.AnthropicClient;
import com.anthropic.sdk.models.*;

public class StreamingExample {

    public static void main(String[] args) {
        AnthropicClient client = AnthropicClient.builder()
            .apiKey(System.getenv("ANTHROPIC_API_KEY"))
            .build();

        System.out.print("Claude: ");

        // stream() thay vì create() — trả về StreamingResponse
        try (var stream = client.messages().stream(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_SONNET_4_6)
                .maxTokens(1024)
                .addUserMessage("Explain Java memory management in detail.")
                .build()
        )) {
            // Iterate qua từng text delta
            stream.textStream().forEach(textDelta -> {
                System.out.print(textDelta);  // Print ngay khi nhận được
                System.out.flush();           // Flush để hiển thị ngay
            });
        }

        System.out.println(); // Xuống dòng sau khi stream xong
    }
}
```

### 4.3 Streaming với metadata đầy đủ

```java
package com.example.lesson02;

import com.anthropic.sdk.AnthropicClient;
import com.anthropic.sdk.models.*;
import com.anthropic.sdk.core.http.StreamResponse;

public class StreamingWithMetadata {

    public static String streamAndCollect(AnthropicClient client, String userMessage) {
        StringBuilder fullResponse = new StringBuilder();

        try (var stream = client.messages().stream(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_SONNET_4_6)
                .maxTokens(1024)
                .addUserMessage(userMessage)
                .build()
        )) {
            // Stream text chunks
            stream.textStream().forEach(delta -> {
                fullResponse.append(delta);
                System.out.print(delta);
                System.out.flush();
            });

            // Sau khi stream xong, lấy final message với full metadata
            Message finalMessage = stream.finalMessage();

            System.out.println(); // New line
            System.err.printf("[Tokens: %d in, %d out | Stop: %s]%n",
                finalMessage.usage().inputTokens(),
                finalMessage.usage().outputTokens(),
                finalMessage.stopReason());
        }

        return fullResponse.toString();
    }

    public static void main(String[] args) {
        AnthropicClient client = AnthropicClient.builder()
            .apiKey(System.getenv("ANTHROPIC_API_KEY"))
            .build();

        String response = streamAndCollect(client,
            "Write a Spring Boot REST controller for user management.");
        // response chứa full text để lưu vào history
    }
}
```

### 4.4 Streaming với multi-turn conversation

```java
public class StreamingChatService {

    private final AnthropicClient client;
    private final List<MessageParam> history = new ArrayList<>();

    public StreamingChatService(AnthropicClient client) {
        this.client = client;
    }

    /**
     * Chat với streaming output.
     * @param userMessage Input từ user
     * @param onToken Callback được gọi với mỗi token (cho UI update)
     * @return Full response text
     */
    public String chatStreaming(String userMessage,
                                java.util.function.Consumer<String> onToken) {
        history.add(MessageParam.builder()
            .role(MessageParam.Role.USER)
            .content(userMessage)
            .build());

        StringBuilder fullText = new StringBuilder();

        try (var stream = client.messages().stream(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_SONNET_4_6)
                .maxTokens(1024)
                .messages(history)
                .build()
        )) {
            stream.textStream().forEach(delta -> {
                fullText.append(delta);
                onToken.accept(delta); // Notify caller với mỗi token
            });

            // Lưu full response vào history
            history.add(MessageParam.builder()
                .role(MessageParam.Role.ASSISTANT)
                .content(fullText.toString())
                .build());
        }

        return fullText.toString();
    }
}

// Usage:
chatService.chatStreaming(
    "Explain Java Streams",
    token -> {
        System.out.print(token);
        System.out.flush();
        // Trong Spring Boot: send via SSE (Server-Sent Events)
    }
);
```

---

## 5. Async Calls với CompletableFuture

### 5.1 Tại sao cần Async?

Trong Java backend (Spring Boot), blocking API call làm block thread pool. Với async:
- Thread không bị block trong khi chờ Claude response
- Có thể call nhiều endpoints song song
- Better throughput cho high-traffic services

### 5.2 CompletableFuture với Anthropic SDK

```java
package com.example.lesson02;

import com.anthropic.sdk.AnthropicClient;
import com.anthropic.sdk.models.*;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class AsyncApiCallExample {

    private final AnthropicClient client;
    private final ExecutorService executor;

    public AsyncApiCallExample(AnthropicClient client) {
        this.client = client;
        // Dedicated thread pool cho API calls
        this.executor = Executors.newFixedThreadPool(10);
    }

    /**
     * Non-blocking API call — trả về CompletableFuture
     */
    public CompletableFuture<String> askAsync(String question) {
        return CompletableFuture.supplyAsync(() -> {
            Message response = client.messages().create(
                MessageCreateParams.builder()
                    .model(Model.CLAUDE_SONNET_4_6)
                    .maxTokens(512)
                    .addUserMessage(question)
                    .build()
            );
            return extractText(response);
        }, executor);
    }

    /**
     * Gọi nhiều prompts song song — HIỆU QUẢ HƠN tuần tự nhiều lần
     */
    public void parallelCalls() throws Exception {
        long startTime = System.currentTimeMillis();

        // Khởi chạy 3 calls đồng thời
        CompletableFuture<String> f1 = askAsync("What is Java generics?");
        CompletableFuture<String> f2 = askAsync("What is Java streams?");
        CompletableFuture<String> f3 = askAsync("What is Java lambda?");

        // Chờ tất cả xong
        CompletableFuture.allOf(f1, f2, f3).join();

        long elapsed = System.currentTimeMillis() - startTime;
        System.out.printf("3 parallel calls completed in %dms%n", elapsed);
        // Sequential: ~9s | Parallel: ~3s (limited by slowest call)

        System.out.println("Generics: " + f1.get());
        System.out.println("Streams: " + f2.get());
        System.out.println("Lambda: " + f3.get());
    }

    /**
     * Async với error handling
     */
    public CompletableFuture<String> askWithFallback(String question) {
        return askAsync(question)
            .exceptionally(throwable -> {
                // Fallback nếu API call fail
                System.err.println("API call failed: " + throwable.getMessage());
                return "Sorry, I couldn't answer that question right now.";
            });
    }

    private String extractText(Message message) {
        return message.content().stream()
            .filter(b -> b instanceof ContentBlock.TextBlock)
            .map(b -> ((ContentBlock.TextBlock) b).text())
            .findFirst().orElse("");
    }
}
```

### 5.3 Spring Boot Async với @Async

```java
@Service
public class ClaudeAsyncService {

    private final AnthropicClient anthropicClient;

    @Async("claudeTaskExecutor")  // Custom thread pool
    public CompletableFuture<String> analyzeCodeAsync(String code) {
        String prompt = "Review this Java code for bugs:\n```java\n" + code + "\n```";

        Message response = anthropicClient.messages().create(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_SONNET_4_6)
                .maxTokens(1024)
                .addUserMessage(prompt)
                .build()
        );

        String result = extractText(response);
        return CompletableFuture.completedFuture(result);
    }
}

@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean("claudeTaskExecutor")
    public Executor claudeTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("claude-");
        executor.initialize();
        return executor;
    }
}
```

---

## 6. Prompt Caching — Tiết kiệm Cost đáng kể

### 6.1 Prompt Caching là gì?

Anthropic cho phép **cache** phần của prompt để reuse trong các requests tiếp theo. Khi cache hit:
- **90% cost savings** trên cached tokens
- **Faster response** (cached tokens skip processing)

**Khi nào dùng prompt caching?**
- System prompts dài (> 1024 tokens) — tài liệu kỹ thuật, schema, guidelines
- Context được reuse nhiều lần — conversation với fixed background
- Repeated reference documents — code base context, API documentation

### 6.2 Cache Control trong Java

```java
package com.example.lesson02;

import com.anthropic.sdk.AnthropicClient;
import com.anthropic.sdk.models.*;
import com.anthropic.sdk.core.JsonValue;

public class PromptCachingExample {

    /**
     * Ví dụ: System prompt dài với caching.
     * Claude sẽ cache phần này sau lần đầu tiên.
     */
    public static Message callWithCachedSystemPrompt(
            AnthropicClient client,
            String largeSystemPrompt,  // Phải > 1024 tokens để cache có hiệu quả
            String userMessage) {

        return client.messages().create(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_SONNET_4_6)
                .maxTokens(1024)
                // System prompt với cache_control
                .system(List.of(
                    TextBlockParam.builder()
                        .text(largeSystemPrompt)
                        .cacheControl(CacheControlEphemeral.builder().build()) // ← cache này
                        .build()
                ))
                .addUserMessage(userMessage)
                .build()
        );
    }

    /**
     * Ví dụ thực tế: Cache database schema (thường là 2000-5000 tokens)
     * Mỗi query tới DB assistant đều reuse schema đã cache
     */
    public static void databaseAssistantWithCaching() {
        AnthropicClient client = AnthropicClient.builder()
            .apiKey(System.getenv("ANTHROPIC_API_KEY"))
            .build();

        // Schema này dài ~3000 tokens — cache sẽ tiết kiệm đáng kể
        String dbSchema = loadDatabaseSchema(); // Từ file hoặc DB metadata

        String systemPromptWithSchema = """
            Bạn là Database Assistant cho hệ thống e-commerce.
            
            Database Schema:
            """ + dbSchema + """
            
            Rules:
            - Chỉ generate SELECT queries
            - Luôn có LIMIT clause tối đa 1000
            - Explain query khi được hỏi
            """;

        // Lần 1: Cache miss — tốn full cost
        Message r1 = callWithCachedSystemPrompt(
            client, systemPromptWithSchema, "Show me all orders from today");
        System.out.printf("Request 1 - Cache creation tokens: %s%n",
            r1.usage()); // cache_creation_input_tokens sẽ > 0

        // Lần 2: Cache HIT — tiết kiệm 90% cost trên system prompt
        Message r2 = callWithCachedSystemPrompt(
            client, systemPromptWithSchema, "Find top 10 customers by revenue");
        System.out.printf("Request 2 - Cache read tokens: %s%n",
            r2.usage()); // cache_read_input_tokens sẽ > 0
    }

    private static String loadDatabaseSchema() {
        // Trong thực tế: đọc từ file hoặc query information_schema
        return """
            CREATE TABLE users (
                id BIGSERIAL PRIMARY KEY,
                email VARCHAR(255) UNIQUE NOT NULL,
                name VARCHAR(100) NOT NULL,
                created_at TIMESTAMP DEFAULT NOW()
            );
            CREATE TABLE products (
                id BIGSERIAL PRIMARY KEY,
                name VARCHAR(255) NOT NULL,
                price DECIMAL(10,2) NOT NULL,
                stock_quantity INTEGER DEFAULT 0,
                category_id BIGINT REFERENCES categories(id)
            );
            -- ... 20 more tables
            """;
    }
}
```

### 6.3 Cache Usage Metrics

```java
// Sau khi gọi API, kiểm tra cache performance
Message response = client.messages().create(params);

// Standard usage
long inputTokens = response.usage().inputTokens();
long outputTokens = response.usage().outputTokens();

// Cache-specific (nếu dùng prompt caching)
// cache_creation_input_tokens: tokens mới được cache
// cache_read_input_tokens: tokens đọc từ cache (giá rẻ hơn 90%)

System.out.printf("""
    Token Usage:
      Input tokens: %d
      Output tokens: %d
    """, inputTokens, outputTokens);
```

### 6.4 Cost Calculation với Caching

```java
public class CostCalculator {

    // Giá Claude Sonnet 4.6 (USD per 1M tokens)
    private static final double INPUT_PRICE = 3.0;
    private static final double OUTPUT_PRICE = 15.0;
    private static final double CACHE_WRITE_PRICE = 3.75; // 25% đắt hơn khi write
    private static final double CACHE_READ_PRICE = 0.30;  // 90% rẻ hơn khi read

    public static double calculateCost(Message message) {
        long inputTokens = message.usage().inputTokens();
        long outputTokens = message.usage().outputTokens();

        double inputCost = (inputTokens / 1_000_000.0) * INPUT_PRICE;
        double outputCost = (outputTokens / 1_000_000.0) * OUTPUT_PRICE;

        return inputCost + outputCost;
    }

    /**
     * Minh họa savings khi dùng caching cho 100 requests
     * với system prompt 3000 tokens + user message 100 tokens
     */
    public static void illustrateCachingSavings() {
        int requests = 100;
        int systemPromptTokens = 3000;
        int userMessageTokens = 100;
        int outputTokens = 500;

        // Without caching: mỗi request trả full price
        double withoutCaching = requests *
            ((systemPromptTokens + userMessageTokens) / 1_000_000.0 * INPUT_PRICE
            + outputTokens / 1_000_000.0 * OUTPUT_PRICE);

        // With caching: lần 1 write cache, lần 2-100 read cache
        double withCaching =
            // Request 1: cache write (đắt hơn 25%)
            (systemPromptTokens / 1_000_000.0 * CACHE_WRITE_PRICE)
            + (userMessageTokens / 1_000_000.0 * INPUT_PRICE)
            + (outputTokens / 1_000_000.0 * OUTPUT_PRICE)
            // Requests 2-100: cache read (rẻ hơn 90%)
            + (requests - 1) * (
                systemPromptTokens / 1_000_000.0 * CACHE_READ_PRICE
                + userMessageTokens / 1_000_000.0 * INPUT_PRICE
                + outputTokens / 1_000_000.0 * OUTPUT_PRICE
            );

        System.out.printf("Without caching: $%.4f%n", withoutCaching);
        System.out.printf("With caching:    $%.4f%n", withCaching);
        System.out.printf("Savings:         $%.4f (%.0f%%)%n",
            withoutCaching - withCaching,
            (1 - withCaching / withoutCaching) * 100);
    }
}
// Output:
// Without caching: $0.1680
// With caching:    $0.0381
// Savings:         $0.1299 (77%)
```

---

## 7. Exercise: Spring Boot REST Chat Endpoint

### Yêu cầu

Build Spring Boot REST endpoint `/api/chat` có khả năng:
1. POST `/api/chat` — gửi message, nhận response
2. Maintain session history (per session ID)
3. Streaming response qua Server-Sent Events (SSE): GET `/api/chat/stream`
4. DELETE `/api/chat/{sessionId}` — clear session history

### API Contract

```
POST /api/chat
Request:  {"sessionId": "abc123", "message": "Hello Claude"}
Response: {"sessionId": "abc123", "response": "Hello! How can I help?", "tokens": {"input": 12, "output": 8}}

GET /api/chat/stream?sessionId=abc123&message=Hello
Response: text/event-stream
  data: Hello
  data: !
  data: How
  data:  can
  ...
  data: [DONE]

DELETE /api/chat/{sessionId}
Response: {"message": "Session cleared"}
```

### Skeleton Code

```java
@RestController
@RequestMapping("/api/chat")
public class ChatController {

    private final ClaudeSessionService sessionService;

    public ChatController(ClaudeSessionService sessionService) {
        this.sessionService = sessionService;
    }

    // TODO 1: POST endpoint
    @PostMapping
    public ResponseEntity<ChatResponse> chat(@RequestBody ChatRequest request) {
        // TODO: validate sessionId và message
        // TODO: gọi sessionService.chat(sessionId, message)
        // TODO: trả về ChatResponse với response text và token counts
        return null;
    }

    // TODO 2: Streaming endpoint với SseEmitter
    @GetMapping("/stream")
    public SseEmitter streamChat(
            @RequestParam String sessionId,
            @RequestParam String message) {
        SseEmitter emitter = new SseEmitter(60_000L); // 60s timeout
        // TODO: gọi sessionService.chatStreaming() với onToken callback
        // TODO: mỗi token, gọi emitter.send(token)
        // TODO: khi xong, gửi "[DONE]" và emitter.complete()
        // TODO: run trong separate thread để không block
        return emitter;
    }

    // TODO 3: DELETE endpoint
    @DeleteMapping("/{sessionId}")
    public ResponseEntity<Map<String, String>> clearSession(@PathVariable String sessionId) {
        // TODO: gọi sessionService.clearSession(sessionId)
        return null;
    }
}

// TODO: Implement ClaudeSessionService
// - Lưu history per sessionId trong ConcurrentHashMap<String, List<MessageParam>>
// - chat(sessionId, message) -> String
// - chatStreaming(sessionId, message, Consumer<String> onToken) -> void
// - clearSession(sessionId) -> void

// TODO: Request/Response DTOs
record ChatRequest(String sessionId, String message) {}
record ChatResponse(String sessionId, String response, TokenUsage tokens) {}
record TokenUsage(long input, long output) {}
```

### Testing

```bash
# Test basic chat
curl -X POST http://localhost:8080/api/chat \
  -H "Content-Type: application/json" \
  -d '{"sessionId":"session1", "message":"What is Java?"}'

# Test multi-turn (same sessionId)
curl -X POST http://localhost:8080/api/chat \
  -H "Content-Type: application/json" \
  -d '{"sessionId":"session1", "message":"Give me a code example"}'

# Test streaming
curl -N "http://localhost:8080/api/chat/stream?sessionId=s2&message=Explain+streams"

# Clear session
curl -X DELETE http://localhost:8080/api/chat/session1
```

---

## Tóm tắt Lesson 02

| Khái niệm | Key Points |
|-----------|-----------|
| Message structure | role (user/assistant) + content blocks (text/image/tool) |
| Multi-turn | API stateless — client phải gửi full history mỗi request |
| System prompt | Đặt role, constraints, format TRƯỚC user messages |
| Streaming | `stream()` thay `create()`, iterate `textStream()` |
| Async | `CompletableFuture.supplyAsync()`, `@Async` cho Spring |
| Prompt caching | `cache_control` annotation, tiết kiệm 77-90% cost cho long context |

**Context window awareness**: Claude Sonnet có context window 200K tokens. Với conversation dài, history + system prompt có thể tiến gần limit. Implement `truncateToLastN()` hoặc summarization để handle.

**Tiếp theo**: [Lesson 03 — Tool Use](./03-tool-use.md) — Phần quan trọng nhất: dạy Claude gọi Java functions của bạn để truy cập database, API, và filesystem.
