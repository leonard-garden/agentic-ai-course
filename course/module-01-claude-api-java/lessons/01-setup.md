# Lesson 01: Setup & First API Call

> **Thời lượng**: 2 buổi (~4 giờ)  
> **Mục tiêu**: Setup môi trường hoàn chỉnh, gọi Claude API lần đầu từ Java, hiểu error handling cơ bản

---

## 1. Tạo Anthropic Account & Lấy API Key

### Bước 1: Đăng ký account

Truy cập https://console.anthropic.com và đăng ký. Anthropic cung cấp **free tier** với $5 credit để bắt đầu — đủ để học module này mà không cần bỏ tiền.

### Bước 2: Lấy API key

1. Đăng nhập vào Console
2. Vào **API Keys** (menu trái)
3. Click **Create Key**
4. Đặt tên gợi nhớ: `java-course-dev`
5. **Copy key ngay** — bạn chỉ thấy 1 lần

API key có dạng: `sk-ant-api03-...` (dài ~100 ký tự)

### Bước 3: Bảo vệ API key — QUAN TRỌNG

```
⚠️  API key = password. Ai có key này có thể dùng account của bạn.
    - KHÔNG commit vào Git
    - KHÔNG paste vào chat/Slack
    - KHÔNG hardcode trong source code
```

Ngay bây giờ, add vào `.gitignore` nếu chưa có:
```
# .gitignore
.env
*.env
application-local.properties
```

---

## 2. Environment Setup

### Option A: Environment Variable (khuyến nghị cho development)

**macOS/Linux:**
```bash
# Thêm vào ~/.zshrc hoặc ~/.bashrc
export ANTHROPIC_API_KEY="sk-ant-api03-your-key-here"

# Reload shell
source ~/.zshrc
```

**Windows (PowerShell):**
```powershell
# Permanent (user-level)
[Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", "sk-ant-...", "User")
```

**Verify:**
```bash
echo $ANTHROPIC_API_KEY
# Output: sk-ant-api03-...
```

### Option B: `.env` file với dotenv-java (cho project-level config)

Nếu project có nhiều env vars, dùng `.env` file:

```bash
# .env (KHÔNG commit file này)
ANTHROPIC_API_KEY=sk-ant-api03-...
DATABASE_URL=jdbc:postgresql://localhost:5432/mydb
```

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.github.cdimascio</groupId>
    <artifactId>dotenv-java</artifactId>
    <version>3.0.0</version>
</dependency>
```

```java
import io.github.cdimascio.dotenv.Dotenv;

Dotenv dotenv = Dotenv.load();
String apiKey = dotenv.get("ANTHROPIC_API_KEY");
```

### Option C: Spring Boot `application.properties`

```properties
# application.properties (commit cái này)
anthropic.api-key=${ANTHROPIC_API_KEY}
anthropic.model=claude-sonnet-4-6
anthropic.max-tokens=1024
```

```java
@Value("${anthropic.api-key}")
private String apiKey;
```

---

## 3. Maven Dependency Setup

### Thêm dependency vào `pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>claude-java-course</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <!-- Anthropic SDK for Java -->
        <dependency>
            <groupId>com.anthropic</groupId>
            <artifactId>anthropic-sdk-java</artifactId>
            <version>0.8.0-alpha.1</version>
        </dependency>

        <!-- Logging -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-simple</artifactId>
            <version>2.0.9</version>
        </dependency>

        <!-- Optional: dotenv for .env file support -->
        <dependency>
            <groupId>io.github.cdimascio</groupId>
            <artifactId>dotenv-java</artifactId>
            <version>3.0.0</version>
        </dependency>
    </dependencies>
</project>
```

**Chạy để download dependencies:**
```bash
mvn dependency:resolve
# Hoặc
mvn compile
```

### Gradle alternative

```groovy
// build.gradle
dependencies {
    implementation 'com.anthropic:anthropic-sdk-java:0.8.0-alpha.1'
    implementation 'org.slf4j:slf4j-simple:2.0.9'
}
```

### Verify SDK đã download

```bash
mvn dependency:tree | grep anthropic
# Output: [INFO] \- com.anthropic:anthropic-sdk-java:jar:0.8.0-alpha.1:compile
```

---

## 4. First API Call — Phân tích từng dòng

Đây là code đầy đủ để gọi Claude API lần đầu:

```java
package com.example.lesson01;

import com.anthropic.sdk.AnthropicClient;
import com.anthropic.sdk.models.ContentBlock;
import com.anthropic.sdk.models.Message;
import com.anthropic.sdk.models.MessageCreateParams;
import com.anthropic.sdk.models.Model;

public class FirstApiCall {

    public static void main(String[] args) {
        // ① Khởi tạo client
        AnthropicClient client = AnthropicClient.builder()
            .apiKey(System.getenv("ANTHROPIC_API_KEY"))  // ② Đọc key từ env
            .build();

        // ③ Tạo request và gửi
        Message message = client.messages().create(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_SONNET_4_6)   // ④ Chọn model
                .maxTokens(1024)                   // ⑤ Giới hạn output
                .addUserMessage("Hello Claude! Tôi là Java engineer. " +
                    "Giải thích ngắn gọn: LLM là gì?")  // ⑥ User message
                .build()
        );

        // ⑦ Lấy text từ response
        ContentBlock firstBlock = message.content().get(0);
        System.out.println("Response: " + firstBlock);

        // ⑧ In thêm metadata
        System.out.println("---");
        System.out.println("Model: " + message.model());
        System.out.println("Stop reason: " + message.stopReason());
        System.out.println("Input tokens: " + message.usage().inputTokens());
        System.out.println("Output tokens: " + message.usage().outputTokens());
    }
}
```

### Giải thích từng phần

**① AnthropicClient — HTTP client wrapper**

```java
AnthropicClient client = AnthropicClient.builder()
    .apiKey(System.getenv("ANTHROPIC_API_KEY"))
    .build();
```

`AnthropicClient` là entry point của SDK. Nó wrap tất cả HTTP calls, authentication, retry logic. Builder pattern quen thuộc trong Java — bạn có thể configure thêm:

```java
AnthropicClient client = AnthropicClient.builder()
    .apiKey(System.getenv("ANTHROPIC_API_KEY"))
    .timeout(Duration.ofSeconds(30))   // Request timeout
    .maxRetries(3)                      // Auto retry on 5xx errors
    .build();
```

**② `System.getenv()` — Đọc environment variable**

```java
.apiKey(System.getenv("ANTHROPIC_API_KEY"))
```

`System.getenv("ANTHROPIC_API_KEY")` trả về `String` hoặc `null` nếu không tìm thấy. Đây là standard Java way để đọc env vars. KHÔNG bao giờ viết:

```java
// ❌ WRONG — hardcode key
.apiKey("sk-ant-api03-abc123...")

// ✅ CORRECT — từ environment
.apiKey(System.getenv("ANTHROPIC_API_KEY"))
```

**③④⑤⑥ MessageCreateParams — Request builder**

```java
MessageCreateParams.builder()
    .model(Model.CLAUDE_SONNET_4_6)  // ④
    .maxTokens(1024)                  // ⑤
    .addUserMessage("...")            // ⑥
    .build()
```

- **`model`**: Enum `Model` chứa tất cả available models. Dùng constant thay vì string để tránh typo
- **`maxTokens`**: BẮTBUỘC phải set. Giới hạn số tokens trong response. 1 token ≈ 0.75 từ tiếng Anh. Set quá nhỏ → response bị cắt. Set quá lớn → tốn tiền không cần thiết
- **`addUserMessage()`**: Thêm user message vào conversation. SDK tự wrap thành `{"role": "user", "content": "..."}`

**⑦ Lấy response content**

```java
ContentBlock firstBlock = message.content().get(0);
```

`message.content()` trả về `List<ContentBlock>`. Trong response thông thường, có 1 block kiểu `text`. Trong tool use (lesson 03), có thể có nhiều blocks.

**⑧ Usage metadata**

```java
message.usage().inputTokens()   // Tokens trong request (prompt)
message.usage().outputTokens()  // Tokens trong response
```

Quan trọng để track cost. `inputTokens * price_in + outputTokens * price_out = total_cost`.

---

## 5. Hiểu Response Object

Hãy explore `Message` object đầy đủ hơn:

```java
public class ExploreResponse {

    public static void main(String[] args) {
        AnthropicClient client = AnthropicClient.builder()
            .apiKey(System.getenv("ANTHROPIC_API_KEY"))
            .build();

        Message message = client.messages().create(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_SONNET_4_6)
                .maxTokens(256)
                .addUserMessage("What is 2 + 2?")
                .build()
        );

        // ID của message — dùng cho logging/debugging
        System.out.println("Message ID: " + message.id());

        // Type luôn là "message"
        System.out.println("Type: " + message.type());

        // Role của response luôn là "assistant"
        System.out.println("Role: " + message.role());

        // Content blocks
        System.out.println("Content blocks count: " + message.content().size());
        for (ContentBlock block : message.content()) {
            System.out.println("  Block type: " + block.type());
            // Cast về TextBlock để lấy text
            if (block instanceof ContentBlock.TextBlock textBlock) {
                System.out.println("  Text: " + textBlock.text());
            }
        }

        // Stop reason: "end_turn", "max_tokens", "stop_sequence", "tool_use"
        System.out.println("Stop reason: " + message.stopReason());

        // Model đã dùng (có thể khác model request nếu có routing)
        System.out.println("Model used: " + message.model());

        // Token usage
        System.out.println("Input tokens: " + message.usage().inputTokens());
        System.out.println("Output tokens: " + message.usage().outputTokens());
    }
}
```

**Output mẫu:**
```
Message ID: msg_01XFDUDYJgAACzvnptvVoYEL
Type: message
Role: assistant
Content blocks count: 1
  Block type: text
  Text: 2 + 2 equals 4.
Stop reason: end_turn
Model used: claude-sonnet-4-6-20250219
Input tokens: 13
Output tokens: 10
```

### Stop Reason — quan trọng để handle đúng

| Stop Reason | Ý nghĩa | Action |
|------------|---------|--------|
| `end_turn` | Claude hoàn thành tự nhiên | Normal — response đầy đủ |
| `max_tokens` | Bị cắt vì đạt maxTokens | Tăng maxTokens hoặc truncate message |
| `stop_sequence` | Gặp stop sequence đã config | Expected nếu bạn set stop sequences |
| `tool_use` | Claude muốn gọi tool | Phải xử lý tool call (Lesson 03) |

```java
// ✅ Nên kiểm tra stop reason
if ("max_tokens".equals(message.stopReason().toString())) {
    logger.warn("Response was truncated! Consider increasing maxTokens.");
}
```

---

## 6. Error Handling Cơ bản

Anthropic SDK throw các exceptions có hierarchy rõ ràng:

```
AnthropicException (base)
├── AnthropicServiceException (HTTP 4xx/5xx từ API)
│   ├── AuthenticationException (401) — API key sai/hết hạn
│   ├── PermissionDeniedException (403) — Không có quyền
│   ├── NotFoundException (404) — Resource không tồn tại
│   ├── UnprocessableEntityException (422) — Request invalid
│   ├── RateLimitException (429) — Quá nhiều requests
│   ├── InternalServerException (500) — Lỗi phía Anthropic
│   └── OverloadedException (529) — Anthropic servers quá tải
└── AnthropicIoException — Network errors, timeout
```

### Basic error handling

```java
package com.example.lesson01;

import com.anthropic.sdk.AnthropicClient;
import com.anthropic.sdk.errors.*;
import com.anthropic.sdk.models.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class ErrorHandlingExample {

    private static final Logger log = LoggerFactory.getLogger(ErrorHandlingExample.class);

    public static void main(String[] args) {
        // Validate API key trước khi tạo client
        String apiKey = System.getenv("ANTHROPIC_API_KEY");
        if (apiKey == null || apiKey.isBlank()) {
            System.err.println("ERROR: ANTHROPIC_API_KEY environment variable is not set!");
            System.err.println("Run: export ANTHROPIC_API_KEY='sk-ant-...'");
            System.exit(1);
        }

        AnthropicClient client = AnthropicClient.builder()
            .apiKey(apiKey)
            .build();

        try {
            Message message = client.messages().create(
                MessageCreateParams.builder()
                    .model(Model.CLAUDE_SONNET_4_6)
                    .maxTokens(256)
                    .addUserMessage("Hello!")
                    .build()
            );

            System.out.println(extractText(message));

        } catch (AuthenticationException e) {
            // 401 — API key sai
            log.error("Authentication failed. Check your ANTHROPIC_API_KEY. " +
                "Status: {}", e.statusCode());
            // Không retry — key sai thì retry cũng vô ích

        } catch (RateLimitException e) {
            // 429 — Quá nhiều requests
            log.warn("Rate limit hit. Status: {}. Retry after a moment.", e.statusCode());
            // Trong production: implement exponential backoff

        } catch (OverloadedException e) {
            // 529 — Anthropic servers đang quá tải
            log.warn("Anthropic servers overloaded. Retry later. Status: {}", e.statusCode());

        } catch (InternalServerException e) {
            // 500 — Lỗi phía Anthropic
            log.error("Anthropic internal server error. Status: {}", e.statusCode());
            // Retry có thể giúp

        } catch (AnthropicServiceException e) {
            // Catch-all cho các HTTP errors khác
            log.error("API error. Status: {}, Message: {}", e.statusCode(), e.getMessage());

        } catch (AnthropicIoException e) {
            // Network errors — timeout, DNS fail, etc.
            log.error("Network error connecting to Anthropic API: {}", e.getMessage());
        }
    }

    // Helper: extract text từ first content block
    private static String extractText(Message message) {
        return message.content().stream()
            .filter(block -> block instanceof ContentBlock.TextBlock)
            .map(block -> ((ContentBlock.TextBlock) block).text())
            .findFirst()
            .orElse("");
    }
}
```

### Production error handling với retry

```java
import java.time.Duration;
import java.util.function.Supplier;

public class ResilientApiClient {

    private final AnthropicClient client;
    private static final Logger log = LoggerFactory.getLogger(ResilientApiClient.class);

    public ResilientApiClient() {
        // SDK có built-in retry cho 5xx errors
        this.client = AnthropicClient.builder()
            .apiKey(System.getenv("ANTHROPIC_API_KEY"))
            .maxRetries(3)         // Retry tối đa 3 lần cho 5xx
            .timeout(Duration.ofSeconds(60))  // Timeout mỗi request
            .build();
    }

    /**
     * Gọi API với retry thủ công cho RateLimitException.
     * SDK không tự retry 429 vì cần respect Retry-After header.
     */
    public Message callWithRetry(MessageCreateParams params) {
        int maxAttempts = 3;
        long delayMs = 1000; // 1 second initial delay

        for (int attempt = 1; attempt <= maxAttempts; attempt++) {
            try {
                return client.messages().create(params);

            } catch (RateLimitException e) {
                if (attempt == maxAttempts) {
                    throw e; // Hết retries, rethrow
                }
                log.warn("Rate limit hit. Attempt {}/{}. Waiting {}ms...",
                    attempt, maxAttempts, delayMs);
                sleep(delayMs);
                delayMs *= 2; // Exponential backoff: 1s → 2s → 4s

            } catch (AuthenticationException | PermissionDeniedException e) {
                // Không retry auth errors
                throw e;
            }
        }
        throw new IllegalStateException("Should not reach here");
    }

    private void sleep(long ms) {
        try {
            Thread.sleep(ms);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

---

## 7. Best Practices — Từ Day 1

### 7.1 API Key Management

```java
// ❌ NEVER DO THIS
AnthropicClient client = AnthropicClient.builder()
    .apiKey("sk-ant-api03-real-key-here")
    .build();

// ✅ Always use environment variables
AnthropicClient client = AnthropicClient.builder()
    .apiKey(System.getenv("ANTHROPIC_API_KEY"))
    .build();

// ✅ Fail fast nếu key không có
public static AnthropicClient createClient() {
    String key = Objects.requireNonNull(
        System.getenv("ANTHROPIC_API_KEY"),
        "ANTHROPIC_API_KEY environment variable must be set"
    );
    return AnthropicClient.builder().apiKey(key).build();
}
```

### 7.2 Logging — Log đủ nhưng không log secrets

```java
// ✅ Log useful info for debugging
log.info("Calling Claude API. Model: {}, MaxTokens: {}", model, maxTokens);
log.debug("Prompt: {}", prompt); // DEBUG level — không log trong production
log.info("Response received. InputTokens: {}, OutputTokens: {}, StopReason: {}",
    message.usage().inputTokens(),
    message.usage().outputTokens(),
    message.stopReason());

// ❌ NEVER log API key
log.info("Using API key: {}", apiKey); // WRONG!
```

### 7.3 Singleton Client

```java
// ✅ Tạo client 1 lần, reuse nhiều lần (như DataSource/HttpClient)
// AnthropicClient là thread-safe, manage connection pooling internally

@Configuration
public class AnthropicConfig {

    @Bean
    @Singleton
    public AnthropicClient anthropicClient(
            @Value("${anthropic.api-key}") String apiKey) {
        return AnthropicClient.builder()
            .apiKey(apiKey)
            .maxRetries(3)
            .timeout(Duration.ofSeconds(60))
            .build();
    }
}
```

### 7.4 MaxTokens Sizing

```java
// Rule of thumb:
// - Short answers (yes/no, classification): 50-100 tokens
// - Paragraph response: 200-500 tokens
// - Detailed explanation: 500-1000 tokens
// - Long-form content: 2000-4000 tokens
// - Max (Claude Sonnet): 8096 tokens

// ✅ Size appropriately, không phải luôn dùng max
MessageCreateParams.builder()
    .maxTokens(estimateRequiredTokens(request))
    .build()

private int estimateRequiredTokens(String requestType) {
    return switch (requestType) {
        case "classification" -> 50;
        case "summary" -> 300;
        case "explanation" -> 800;
        case "code_generation" -> 2000;
        default -> 1024;
    };
}
```

---

## 8. Exercise: Java CLI Tool

### Yêu cầu

Viết Java CLI tool tên `ClaudeChat` với các tính năng:
1. Nhận prompt từ command-line arguments: `java ClaudeChat "Your question here"`
2. Gọi Claude API
3. In response ra stdout
4. Handle errors với user-friendly messages
5. In usage stats (input/output tokens) ra stderr

### Expected Output

```bash
$ java ClaudeChat "What is Java Stream API?"
Java Stream API is a functional-style approach to processing collections...
[stderr] Tokens used: 18 input, 156 output
```

### Skeleton Code

```java
package com.example.exercise01;

import com.anthropic.sdk.AnthropicClient;
import com.anthropic.sdk.errors.*;
import com.anthropic.sdk.models.*;

public class ClaudeChat {

    public static void main(String[] args) {
        // TODO 1: Validate args — phải có đúng 1 argument
        if (args.length != 1) {
            System.err.println("Usage: java ClaudeChat \"Your question here\"");
            System.exit(1);
        }

        String userPrompt = args[0];

        // TODO 2: Validate API key từ environment
        // Gợi ý: dùng Objects.requireNonNull() hoặc kiểm tra null

        // TODO 3: Tạo AnthropicClient
        // Gợi ý: AnthropicClient.builder()...

        // TODO 4: Tạo MessageCreateParams
        // - Model: CLAUDE_SONNET_4_6
        // - MaxTokens: 1024
        // - User message: userPrompt

        // TODO 5: Gọi API với error handling
        // - AuthenticationException: "Invalid API key. Check ANTHROPIC_API_KEY."
        // - RateLimitException: "Rate limit exceeded. Please wait and retry."
        // - AnthropicIoException: "Network error: " + e.getMessage()
        // - Tất cả exceptions khác: "Unexpected error: " + e.getMessage()

        // TODO 6: Print response text ra stdout

        // TODO 7: Print usage stats ra stderr
        // Format: "[stderr] Tokens used: X input, Y output"
    }

    // TODO: Helper method extractText(Message message) -> String
}
```

### Hints

```java
// Cách extract text từ content blocks:
String text = message.content().stream()
    .filter(b -> b instanceof ContentBlock.TextBlock)
    .map(b -> ((ContentBlock.TextBlock) b).text())
    .collect(Collectors.joining("\n"));

// Print to stderr:
System.err.printf("Tokens used: %d input, %d output%n",
    message.usage().inputTokens(),
    message.usage().outputTokens());
```

### Bonus Challenges

1. **Multiple questions**: Nhận nhiều arguments, gửi từng cái một
2. **System prompt**: Thêm `--system "You are a Java expert"` flag
3. **Model selection**: Thêm `--model haiku` flag để chọn model rẻ hơn
4. **JSON output**: Thêm `--json` flag để output dạng JSON với text + tokens

---

## 9. Kiểm tra kiến thức

Trả lời nhanh (không cần code):

1. Tại sao KHÔNG được hardcode API key trong source code?
2. `message.stopReason()` trả về `"max_tokens"` — điều này có nghĩa là gì?
3. Khi nào dùng `maxTokens=50` vs `maxTokens=2000`?
4. Nếu `ANTHROPIC_API_KEY` không được set, `System.getenv()` trả về gì?
5. `AuthenticationException` khác `RateLimitException` ở điểm nào quan trọng nhất?

---

## Tóm tắt Lesson 01

| Khái niệm | Tóm tắt |
|-----------|---------|
| `AnthropicClient` | Singleton, thread-safe, manage HTTP connections |
| `MessageCreateParams` | Builder pattern cho request. Phải có: model, maxTokens, message |
| `Message` | Response object: content, stopReason, usage stats |
| `ContentBlock` | Unit của response — có thể là text, tool_use, image |
| Error handling | Bắt cụ thể: Auth (no retry), RateLimit (retry+backoff), IO (retry) |
| Best practices | Env var cho key, singleton client, log tokens không log key |

**Tiếp theo**: [Lesson 02 — Messages API & Streaming](./02-messages-and-streaming.md) — Xây dựng multi-turn conversations và real-time streaming output.
