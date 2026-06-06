# Lesson 02 — Anthropic Ecosystem: Toàn cảnh và Kiến trúc Quyết định

> **Module**: 00 — Foundation & Mental Model
> **Thời gian đọc**: ~50 phút | **Thời gian thực hành**: ~40 phút
> **Difficulty**: Beginner–Intermediate

---

## Mục tiêu bài học

Sau bài này bạn có thể:
- Mô tả toàn bộ Anthropic ecosystem và vai trò của từng thành phần
- Chọn đúng model (Opus/Sonnet/Haiku) cho từng use case
- Thiết kế model routing strategy cho multi-agent systems
- Ước tính chi phí token cho một project
- Setup anthropic-sdk-java trong Java project
- Quyết định khi nào dùng Messages API, Managed Agents, hay Claude Code

---

## 1. Sơ đồ Anthropic Ecosystem

Trước khi đi vào chi tiết, hãy nhìn bức tranh tổng thể:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ANTHROPIC ECOSYSTEM                           │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    FOUNDATION MODELS                         │    │
│  │   Claude Opus 4    │   Claude Sonnet 4.6   │  Claude Haiku 4.5│   │
│  │   (Deep reasoning) │   (Best coding)       │  (Fast & cheap)  │   │
│  └──────────────────────────────┬──────────────────────────────┘    │
│                                  │ Model API                          │
│  ┌───────────────────────────────▼──────────────────────────────┐   │
│  │                     CLAUDE API (Core)                         │   │
│  │   Messages API  │  Files API  │  Batch API  │  Admin API      │   │
│  └───────────────┬───────────────────────────────────────────────┘  │
│                  │                                                    │
│     ┌────────────┼────────────────────────────────┐                 │
│     │            │                                 │                 │
│     ▼            ▼                                 ▼                 │
│  ┌──────┐  ┌─────────────┐              ┌──────────────────┐        │
│  │ MCP  │  │  Agent SDK  │              │   Claude Code    │        │
│  │      │  │  (Managed   │              │   (CLI Tool)     │        │
│  │Proto │  │   Agents)   │              │                  │        │
│  │-col  │  │             │              │  - REPL          │        │
│  │      │  │  - Threads  │              │  - File editing  │        │
│  │Tools │  │  - Runs     │              │  - Bash exec     │        │
│  │Resrc │  │  - Tools    │              │  - Git ops       │        │
│  │Prmpt │  │             │              │  - MCP client    │        │
│  └──┬───┘  └─────────────┘              └──────────────────┘        │
│     │                                                                 │
│  ┌──▼──────────────────────────────┐                                │
│  │      MCP SERVERS (Ecosystem)    │                                 │
│  │  Your Java Server │ GitHub MCP  │                                 │
│  │  Database MCP     │ Jira MCP    │                                 │
│  │  Slack MCP        │ Custom MCPs │                                 │
│  └─────────────────────────────────┘                                │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    LANGUAGE SDKs                             │    │
│  │  Python SDK  │  TypeScript SDK  │  Java SDK  │  Go SDK       │   │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

### Thành phần chính

**Foundation Models**: Claude Opus 4, Sonnet 4.6, Haiku 4.5 — các model AI với capabilities khác nhau (chi tiết ở Section 2)

**Claude API (Core)**: HTTP API để gọi Claude. Gồm:
- **Messages API**: Primary API, send messages → get responses. Hỗ trợ tool use, vision, streaming
- **Files API**: Upload files (PDF, images, documents) để Claude process
- **Batch API**: Submit hàng nghìn requests async, xử lý offline với giá 50% rẻ hơn
- **Admin API**: Manage API keys, usage, billing programmatically

**MCP (Model Context Protocol)**: Open standard protocol để cấp tools, resources, và prompts cho AI. Bạn build MCP servers bằng Java/Python/TS, Claude connect đến và sử dụng.

**Agent SDK (Managed Agents)**: Higher-level framework, Anthropic quản lý conversation threading, tool call loops, run lifecycle. Python-first hiện tại.

**Claude Code**: CLI tool cho developers. Claude chạy trong terminal, có thể đọc/viết files, chạy commands, commit code, search codebase.

**Language SDKs**: Official SDKs cho Python, TypeScript, Java, Go — wrap HTTP API với type safety và convenience methods.

---

## 2. Model Family: Opus / Sonnet / Haiku

### Claude Opus 4 — "The Deep Reasoner"

Opus là model mạnh nhất, được thiết kế cho các tasks đòi hỏi reasoning phức tạp nhất.

**Capabilities nổi bật**:
- Extended thinking — có thể "suy nghĩ" trước khi trả lời (visible chain-of-thought)
- Complex multi-step reasoning
- Phân tích sâu, research, strategic planning
- Tốt nhất cho agentic tasks cần judgment cao

**Thông số** (approximate, kiểm tra docs để có số chính xác nhất):
- Context window: 200K tokens
- Tốc độ: Chậm hơn Sonnet (~2–3x)
- Chi phí: Cao nhất (~3–5x Sonnet)

**Khi nào dùng Opus**:
- Orchestrator trong multi-agent system (lên plan, quyết định strategy)
- Phân tích architecture, security review
- Tasks cần reasoning qua nhiều bước logic phức tạp
- Một lần chạy quan trọng hơn cost (e.g., critical decision point)
- Research, synthesis, writing complex technical documents

**Java analogy**: Như senior architect — chậm hơn, tốn kém hơn, nhưng output quality vượt trội cho complex decisions.

---

### Claude Sonnet 4.6 — "The Best Coding Model"

Sonnet là "sweet spot" — balance tốt nhất giữa capability, speed, và cost. Đây là model bạn sẽ dùng nhiều nhất.

**Capabilities nổi bật**:
- Best-in-class coding ability
- Strong reasoning without extended thinking overhead
- Fast enough cho interactive use cases
- Tool use rất tốt

**Thông số**:
- Context window: 200K tokens
- Tốc độ: Nhanh, phù hợp interactive
- Chi phí: Middle tier

**Khi nào dùng Sonnet**:
- Code generation, review, refactoring
- Orchestration logic trong multi-agent systems
- API integration, data processing pipelines
- Interactive applications (response time < 5s quan trọng)
- Mọi coding tasks trong khóa học này

**Java analogy**: Như senior developer — capable, reliable, affordable. Workhorse của team.

---

### Claude Haiku 4.5 — "The Efficient Worker"

Haiku là model nhỏ nhất, nhanh nhất, rẻ nhất. Đừng underestimate — với tasks phù hợp, nó perform 90% của Sonnet.

**Capabilities nổi bật**:
- Fastest response time (sub-second possible)
- Lowest cost per token (~10–20x rẻ hơn Opus)
- Sufficient cho classification, extraction, simple generation
- Ideal cho high-volume, repeated tasks

**Thông số**:
- Context window: 200K tokens  
- Tốc độ: Fastest
- Chi phí: Thấp nhất

**Khi nào dùng Haiku**:
- Classification tasks (route tickets, categorize input)
- Data extraction từ structured documents
- Worker agents trong large-scale parallel processing
- Filtering/triage trước khi pass to more capable model
- High-volume batch processing (thousands of requests)
- Drafting initial output mà Sonnet/Opus sẽ refine

**Java analogy**: Như junior developer cho well-defined tasks — fast, cheap, reliable nếu task rõ ràng.

---

### Model Comparison Table

| Dimension | Opus 4 | Sonnet 4.6 | Haiku 4.5 |
|-----------|--------|-----------|-----------|
| **Reasoning depth** | ★★★★★ | ★★★★☆ | ★★★☆☆ |
| **Coding ability** | ★★★★★ | ★★★★★ | ★★★☆☆ |
| **Speed** | ★★☆☆☆ | ★★★★☆ | ★★★★★ |
| **Cost** | $$$$$ | $$$ | $ |
| **Best role** | Planner/Architect | Developer/Orchestrator | Worker/Filter |
| **Context window** | 200K | 200K | 200K |
| **Extended thinking** | Yes | Limited | No |

---

## 3. Model Routing Strategy cho Multi-Agent Systems

Trong multi-agent systems, một trong những quyết định quan trọng nhất là: **model nào cho agent nào?** Wrong routing = poor performance hoặc unnecessary cost.

### Chiến lược: Tiered Model Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    ORCHESTRATION TIER                        │
│                                                              │
│  Opus: Strategic planning, architecture decisions,           │
│         complex reasoning, final synthesis                   │
└────────────────────────────┬────────────────────────────────┘
                             │ delegates to
┌────────────────────────────▼────────────────────────────────┐
│                    EXECUTION TIER                            │
│                                                              │
│  Sonnet: Code generation, API integration, data processing, │
│          complex tool use, interactive features              │
└────────────────────────────┬────────────────────────────────┘
                             │ delegates to
┌────────────────────────────▼────────────────────────────────┐
│                    WORKER TIER                               │
│                                                              │
│  Haiku: Classification, extraction, filtering,              │
│         high-volume parallel tasks, triage                  │
└─────────────────────────────────────────────────────────────┘
```

### Ví dụ: Code Review Pipeline

```java
public class CodeReviewPipeline {

    // Tier 1: Haiku filters out non-issues quickly (cheap, fast)
    private static final String TRIAGE_MODEL = "claude-haiku-4-5";

    // Tier 2: Sonnet does detailed review (good code understanding)
    private static final String REVIEW_MODEL = "claude-sonnet-4-5";

    // Tier 3: Opus for architectural concerns (deep reasoning)
    private static final String ARCHITECTURE_MODEL = "claude-opus-4-5";

    public ReviewReport review(PullRequest pr) {
        // Step 1: Haiku quickly triages each file
        List<FileFlag> flags = pr.getFiles().parallelStream()
            .map(file -> triageWithHaiku(file))  // cheap, parallel
            .filter(FileFlag::hasIssues)
            .collect(toList());

        // Step 2: Sonnet does detailed review on flagged files
        List<DetailedIssue> issues = flags.stream()
            .map(flag -> reviewWithSonnet(flag))  // thorough code review
            .collect(toList());

        // Step 3: Opus analyzes architectural implications (only if needed)
        ArchitectureAnalysis arch = null;
        if (issues.stream().anyMatch(DetailedIssue::isArchitectural)) {
            arch = analyzeWithOpus(pr, issues);  // expensive, but worth it
        }

        return buildReport(issues, arch);
    }
}
```

### Model Routing Rules of Thumb

| Signal | Model Choice |
|--------|-------------|
| High volume, simple classification | Haiku |
| Code generation, API calls | Sonnet |
| Complex reasoning, planning | Sonnet or Opus |
| One-shot architectural decision | Opus |
| Interactive user-facing | Sonnet (latency matters) |
| Batch offline processing | Haiku (cost matters) |
| Security-critical analysis | Opus |
| Data extraction from documents | Haiku |

---

## 4. Token Economics: Hiểu và Kiểm soát Chi phí

### Tokens là gì?

Tokens là đơn vị xử lý của LLM. Roughly:
- 1 token ≈ 0.75 words tiếng Anh
- 1 token ≈ 0.5 words tiếng Việt (Unicode characters tốn nhiều token hơn)
- `"Hello, world!"` ≈ 4 tokens
- 1000 words ≈ 1333 tokens

### Pricing Structure

Bạn trả cho cả **input tokens** (những gì bạn gửi) và **output tokens** (những gì Claude trả về).

Output tokens thường đắt hơn input tokens ~3–5x.

```
Tổng chi phí = (input_tokens × input_price) + (output_tokens × output_price)
```

**Ví dụ ước tính** (giá tham khảo, check [anthropic.com/pricing](https://anthropic.com/pricing) để có giá chính xác nhất):

```
Sonnet (Approximate):
  Input: ~$3 / million tokens
  Output: ~$15 / million tokens

Một request:
  - System prompt: 500 tokens
  - User message: 200 tokens
  - Response: 500 tokens

Chi phí = (700 × $3/1M) + (500 × $15/1M)
        = $0.0021 + $0.0075
        = ~$0.01 per request

1000 requests/ngày = ~$10/ngày = ~$300/tháng
```

### Context Window và Memory

Claude không có "memory" giữa các conversations. Mỗi API call là independent.

Để maintain context, bạn phải gửi conversation history trong mỗi request:

```java
// Mỗi call phải include full history
List<MessageParam> history = conversationStore.getHistory(sessionId);
history.add(MessageParam.user("Câu hỏi mới của user"));

var response = client.messages().create(
    MessageCreateParams.builder()
        .model(Model.CLAUDE_SONNET_4_5)
        .maxTokens(1024L)
        .messages(history)  // ← toàn bộ conversation history
        .build()
);
```

**Implication**: Conversation càng dài, tokens input càng nhiều → chi phí tăng tuyến tính.

**Context Window Limits**:
- Tất cả Claude models hiện tại: 200K tokens input
- 200K tokens ≈ ~150,000 words ≈ ~500 trang sách
- Đủ cho hầu hết use cases, nhưng vẫn cần quản lý cho long-running conversations

### Prompt Caching — Giảm Chi phí 80–90%

Đây là tính năng quan trọng nhất để tiết kiệm chi phí trong production.

**Vấn đề**: Nếu bạn có một system prompt dài 5000 tokens và gửi 1000 requests, bạn trả cho 5 million tokens system prompt — lặp đi lặp lại.

**Giải pháp**: Prompt Caching. Anthropic cache phần static của prompt lên server. Cached tokens giảm giá:
- Cache write: ~25% rẻ hơn normal input
- Cache read: ~90% rẻ hơn normal input

```java
// Prompt caching với Java SDK
var response = client.messages().create(
    MessageCreateParams.builder()
        .model(Model.CLAUDE_SONNET_4_5)
        .maxTokens(1024L)
        // System prompt lớn — mark để cache
        .system(List.of(
            TextBlockParam.builder()
                .text(largeSystemPrompt)           // 5000 tokens
                .cacheControl(CacheControlEphemeral.builder().build()) // ← cache this!
                .build()
        ))
        .addUserMessage(userMessage)
        .build()
);
```

**Khi nào prompt caching có ích nhất**:
- System prompts dài (>1024 tokens) được dùng nhiều lần
- Codebase context (gửi nhiều files vào prompt)
- Document Q&A (cùng document, nhiều câu hỏi khác nhau)
- Few-shot examples dài

**Savings ví dụ**:
```
Without caching:
  1000 requests × 5000 token system prompt × $3/1M = $15

With caching:
  1 cache write: 5000 tokens × $3.75/1M = $0.019
  999 cache reads: 5000 tokens × $0.30/1M = $1.50
  Total: ~$1.52  (90% savings!)
```

---

## 5. Java SDK: anthropic-sdk-java

### Overview

`anthropic-sdk-java` là official Java SDK do Anthropic maintain. Hỗ trợ Java 8+ và Kotlin.

**Repository**: [github.com/anthropics/anthropic-sdk-java](https://github.com/anthropics/anthropic-sdk-java)

**Features**:
- Synchronous và async clients
- Streaming support (Server-Sent Events)
- Type-safe request/response models
- Automatic retry với exponential backoff
- Java 8+ compatible (không cần Java 21)
- Kotlin-friendly

### Maven Dependency

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>com.anthropic</groupId>
        <artifactId>anthropic-java</artifactId>
        <version>0.8.0</version>  <!-- Kiểm tra phiên bản mới nhất trên Maven Central -->
    </dependency>
</dependencies>
```

### Gradle Dependency

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.anthropic:anthropic-java:0.8.0")
}
```

### Basic Setup và First Call

```java
import com.anthropic.client.AnthropicClient;
import com.anthropic.client.okhttp.AnthropicOkHttpClient;
import com.anthropic.models.messages.*;

public class AnthropicSetup {

    // 1. Tạo client — đọc API key từ environment variable
    private static final AnthropicClient client = new AnthropicOkHttpClient.Builder()
        .apiKey(System.getenv("ANTHROPIC_API_KEY")) // KHÔNG hardcode key!
        .build();

    public static void main(String[] args) {
        // 2. Basic message creation
        Message response = client.messages().create(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_SONNET_4_5)    // model enum — type safe
                .maxTokens(1024L)
                .addUserMessage("Giải thích Spring Boot trong 3 bullet points")
                .build()
        );

        // 3. Extract text từ response
        String text = response.content().stream()
            .filter(ContentBlock::isText)
            .map(block -> block.asText().text())
            .findFirst()
            .orElse("");

        System.out.println(text);

        // 4. Usage information (tokens dùng)
        Usage usage = response.usage();
        System.out.printf("Tokens: %d input, %d output%n",
            usage.inputTokens(), usage.outputTokens());
    }
}
```

### Async Client (Non-blocking)

```java
import com.anthropic.client.AnthropicClientAsync;
import com.anthropic.client.okhttp.AnthropicOkHttpClientAsync;

// Async client cho non-blocking operations
AnthropicClientAsync asyncClient = new AnthropicOkHttpClientAsync.Builder()
    .apiKey(System.getenv("ANTHROPIC_API_KEY"))
    .build();

// Returns CompletableFuture
CompletableFuture<Message> future = asyncClient.messages().create(
    MessageCreateParams.builder()
        .model(Model.CLAUDE_SONNET_4_5)
        .maxTokens(1024L)
        .addUserMessage("Async message")
        .build()
);

future.thenAccept(response -> {
    System.out.println("Response received: " + response.content());
}).exceptionally(ex -> {
    System.err.println("Error: " + ex.getMessage());
    return null;
});
```

### Streaming Response

```java
// Streaming — nhận response từng chunk
try (var stream = client.messages().createStreaming(
    MessageCreateParams.builder()
        .model(Model.CLAUDE_SONNET_4_5)
        .maxTokens(1024L)
        .addUserMessage("Viết một đoạn code Java")
        .build()
)) {
    stream.stream()
        .filter(event -> event instanceof RawContentBlockDeltaEvent)
        .map(event -> (RawContentBlockDeltaEvent) event)
        .filter(event -> event.delta() instanceof ContentBlockDelta.TextDelta)
        .forEach(event -> {
            String chunk = ((ContentBlockDelta.TextDelta) event.delta()).text();
            System.out.print(chunk); // Print từng chunk khi arrive
            System.out.flush();
        });
}
```

### Spring Boot Integration

```java
// Configuration bean
@Configuration
public class AnthropicConfig {

    @Value("${anthropic.api-key}")
    private String apiKey;

    @Bean
    public AnthropicClient anthropicClient() {
        return new AnthropicOkHttpClient.Builder()
            .apiKey(apiKey)
            .build();
    }
}

// Service sử dụng
@Service
public class AiService {

    private final AnthropicClient anthropic;

    public AiService(AnthropicClient anthropic) {
        this.anthropic = anthropic;
    }

    public String generateResponse(String userMessage) {
        Message response = anthropic.messages().create(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_SONNET_4_5)
                .maxTokens(2048L)
                .system("Bạn là một Java expert assistant.")
                .addUserMessage(userMessage)
                .build()
        );

        return extractText(response);
    }

    private String extractText(Message message) {
        return message.content().stream()
            .filter(ContentBlock::isText)
            .map(b -> b.asText().text())
            .collect(Collectors.joining("\n"));
    }
}
```

### application.yml

```yaml
# application.yml
anthropic:
  api-key: ${ANTHROPIC_API_KEY}  # từ environment variable

# Không bao giờ hardcode API key!
# Không commit API key lên git!
```

---

## 6. Kiến trúc Quyết định: Khi nào dùng gì?

Đây là framework giúp bạn quyết định nhanh khi bắt đầu một AI project:

### Decision Tree

```
Bài toán của tôi là gì?
│
├─ Tôi muốn VIẾT CODE, làm việc trong terminal với codebase?
│   └─ → Claude Code (CLI tool)
│       Dùng khi: pair programming, refactoring, debugging,
│                 code review, generating tests
│
├─ Tôi đang BUILD một application/service dùng Java?
│   └─ → Messages API (anthropic-sdk-java)
│       Sub-questions:
│       ├─ Single call, simple response? → Basic Messages API
│       ├─ Need tools/function calling? → Messages API + Tool Use
│       ├─ Long conversation với history? → Messages API + conversation management
│       ├─ High volume, cost-sensitive? → Messages API + Prompt Caching + Batch API
│       └─ Stream response to user? → Messages API + Streaming
│
├─ Tôi muốn EXPOSE TOOLS cho AI (file system, DB, APIs)?
│   └─ → Build MCP Server (Java MCP SDK)
│       Dùng khi: Claude Code cần access vào internal systems,
│                 custom tools cho agents, standardized tool interface
│
└─ Tôi muốn RAPID PROTOTYPE một agent với minimal code?
    └─ → Managed Agents (Agent SDK)
        Note: Python-first, Java support limited
        Dùng khi: POC, exploring capabilities, không cần fine-grained control
```

### Khi nào dùng Claude Code?

Claude Code là công cụ **cho developers** — nó chạy trong terminal và assist bạn trong quá trình coding.

**Dùng Claude Code khi:**
- Bạn muốn AI giúp viết/refactor/debug code trong project thực
- Pair programming với AI
- Generate tests cho existing code
- Explain codebase cho new team member
- Automated code review trong CI/CD

**KHÔNG dùng Claude Code để:**
- Build applications cho end users
- Create API endpoints (đây là Messages API territory)
- Xây dựng automated pipelines chạy không có human

```bash
# Claude Code trong thực tế
cd /path/to/your/spring-boot-project
claude

# Trong Claude Code session:
> "Tìm tất cả các chỗ chưa handle null pointer exceptions trong service layer"
> "Viết unit tests cho UserService với JUnit 5 và Mockito"
> "Refactor OrderController để tách validation logic ra riêng"
```

### Khi nào dùng Messages API?

Messages API là cho **applications bạn build cho người khác** dùng.

**Dùng khi:**
- Build AI features trong Spring Boot services
- Create chatbots, assistants, AI pipelines
- Process documents, generate reports
- Any production application với AI

```java
// Messages API trong Spring Boot service
@RestController
public class AIController {

    @PostMapping("/api/v1/analyze")
    public AnalysisResult analyze(@RequestBody CodeAnalysisRequest request) {
        // Đây là Messages API territory
        return aiService.analyzeCode(request.getCode());
    }
}
```

### Khi nào dùng MCP?

MCP (Model Context Protocol) là khi bạn muốn **cấp tools** cho AI agents.

**Dùng khi:**
- Muốn Claude Code access vào internal database, APIs, file systems
- Build custom tools cho AI assistants trong organization
- Standardize tool interface giữa nhiều AI agents
- Muốn share tools giữa Claude Code và other AI tools

```java
// Java MCP Server — expose database access như một tool
@McpTool(name = "query_orders", description = "Query order data from database")
public QueryResult queryOrders(
    @McpParam("customer_id") String customerId,
    @McpParam("date_range") DateRange dateRange
) {
    return orderRepository.findByCustomerAndDate(customerId, dateRange);
}
```

---

## 7. Reading List — Tài liệu Bắt buộc

### Tier 1: Bắt buộc đọc (làm bài tập này week này)

**"Building Effective Agents" — Anthropic Blog**
- URL: [anthropic.com/research/building-effective-agents](https://www.anthropic.com/research/building-effective-agents)
- Thời gian đọc: ~30 phút
- Tại sao đọc: Bài viết gốc định nghĩa các patterns bạn vừa học. Đọc source để hiểu đúng intent
- Key takeaways: "Use simplest solution that works", workflow patterns taxonomy, when NOT to use agents

**Anthropic Documentation — Messages API**
- URL: [docs.anthropic.com/en/api/messages](https://docs.anthropic.com/en/api/messages)
- Thời gian đọc: ~20 phút (skim structure, đọc kỹ tool use section)
- Tại sao đọc: Bạn sẽ dùng Messages API hàng ngày — biết structure chính xác là must

**anthropic-sdk-java README**
- URL: [github.com/anthropics/anthropic-sdk-java](https://github.com/anthropics/anthropic-sdk-java)
- Thời gian đọc: ~15 phút
- Tại sao đọc: Java SDK documentation, examples, known limitations

### Tier 2: Khuyến nghị đọc (trong Module 01)

**Prompt Caching Guide**
- URL: [docs.anthropic.com/en/docs/build-with-claude/prompt-caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- Khi nào đọc: Trước Lesson 04 (Module 01)
- Key: Cache breakpoints, cache TTL, patterns for maximum savings

**Tool Use (Function Calling) Guide**
- URL: [docs.anthropic.com/en/docs/build-with-claude/tool-use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)
- Khi nào đọc: Trước Lesson 03 (Module 01)
- Key: Tool definition schema, handling tool calls in response, multi-turn tool use

**Model Context Protocol Specification**
- URL: [spec.modelcontextprotocol.io](https://spec.modelcontextprotocol.io)
- Khi nào đọc: Trước Module 03
- Key: Protocol primitives (tools, resources, prompts), transport layer

### Tier 3: Mở rộng (tự chọn)

**Anthropic Academy** — [anthropic.com/academy](https://anthropic.com/academy)
Các free courses ngắn: prompt engineering, tool use, agents. Bổ sung cho khóa học này.

**Anthropic Cookbook** — [github.com/anthropics/anthropic-cookbook](https://github.com/anthropics/anthropic-cookbook)
Python examples cho nhiều patterns. Useful để hiểu concepts dù bạn dùng Java.

### Summary của "Building Effective Agents"

Đây là tóm tắt những điểm quan trọng nhất (đọc bài gốc để có full context):

**Core thesis**: LLM-powered agents có thể thực hiện complex tasks, nhưng success không đến từ maximum sophistication — mà đến từ right tool for the job.

**Key principles**:

1. **Simplicity wins**: "We recommend that you try to use the simplest solution that meets your requirements." Prompt chaining thường tốt hơn complex agent.

2. **Workflows vs Agents**: Workflows có predictable paths, phù hợp khi requirements rõ ràng. Agents phù hợp khi tasks cần dynamic decision-making không thể predefine.

3. **Trust and verification**: Agents cần access vào tools, nhưng mỗi tool là một risk. Implement least-privilege — chỉ cấp tools cần thiết.

4. **Human in the loop**: Biết khi nào cần pause và request human approval. Autonomous không có nghĩa là unmonitored.

5. **Evaluation-driven development**: Build evals sớm. Bạn không thể improve gì bạn không measure.

---

## 8. Exercise: Vẽ Ecosystem Diagram cho Java Project thực của bạn

### Mục tiêu
Không có cách học tốt hơn là apply vào context thực của bạn.

### Bước 1: Chọn một Java project thực
Chọn một project bạn đang làm (work project hoặc side project). Nếu không có, tạo một hypothetical scenario: "E-commerce platform với Spring Boot, PostgreSQL, REST APIs."

### Bước 2: Vẽ sơ đồ hiện tại
Vẽ architecture diagram của project: services, databases, APIs, external integrations.

### Bước 3: Overlay Anthropic ecosystem
Cho mỗi thành phần trong Anthropic ecosystem, trả lời:
- **Claude API + Sonnet**: Tôi có thể add AI features gì vào application? (e.g., intelligent search, document processing, automated summaries)
- **Claude Code**: Tôi có thể dùng Claude Code ở đâu để tăng productivity? (e.g., generate tests, review PRs, refactor legacy code)
- **MCP Server**: Tôi có thể expose tools nào qua MCP? (e.g., database queries, internal APIs, file systems)
- **Prompt Caching**: Có data nào được dùng repeatedly trong prompts không? (e.g., company docs, product catalog)
- **Batch API**: Có tasks nào phù hợp với async batch processing? (e.g., nightly report generation, bulk data analysis)

### Bước 4: Tạo Decision Record

Viết một markdown file với format:

```markdown
# AI Integration Decision Record — [Project Name]

## Project Context
[Mô tả ngắn về project, stack, scale]

## AI Components

### Component 1: [Tên feature]
- **What**: [Mô tả feature]
- **Anthropic primitive**: Messages API / Claude Code / MCP
- **Model**: Haiku / Sonnet / Opus — Tại sao?
- **Estimated cost**: [Rough estimate]
- **Priority**: High / Medium / Low

### Component 2: ...

## What I WON'T use AI for
[Danh sách các thứ KHÔNG nên dùng AI — tại sao?]

## Open Questions
[Những điều chưa rõ, cần research thêm]
```

### Bước 5: Chia sẻ và review
Nếu học theo nhóm, share diagram với người khác và discuss decisions.

---

## Tổng kết bài học

| Concept | Key Takeaway |
|---------|-------------|
| Anthropic Ecosystem | 5 layers: Models → API → MCP/Agents/Claude Code → SDKs → Your app |
| Opus | Deep reasoning, planning, architecture — expensive, slow |
| Sonnet | Best coding, orchestration, interactive — sweet spot |
| Haiku | Classification, extraction, workers — fast, cheap |
| Model Routing | Haiku=workers, Sonnet=execution, Opus=planning |
| Token Economics | Input + Output tokens, output 3–5x pricier |
| Prompt Caching | 80–90% savings on repeated context |
| Messages API | Build applications, full control, Java-native |
| Managed Agents | Rapid prototype, Python-first |
| Claude Code | Developer productivity tool, pair programming |
| MCP | Standard protocol để expose tools cho AI |

---

## Quiz tự kiểm tra

**Q1**: Bạn có 10,000 support tickets cần classify vào 5 categories. Budget limited. Model nào?

<details>
<summary>Đáp án</summary>
**Haiku** — Classification task đơn giản, high volume, cost-sensitive. Haiku perform ~90% của Sonnet cho classification tasks ở một phần nhỏ chi phí.
</details>

**Q2**: Bạn build một AI code review tool integrate vào GitHub Actions. Model nào cho reviewer agent?

<details>
<summary>Đáp án</summary>
**Sonnet** — Code review cần good reasoning và code understanding. Sonnet là best coding model. Opus thường overkill trừ khi review architecture-level concerns. Có thể dùng Haiku để triage/filter trước khi Sonnet review deep.
</details>

**Q3**: System prompt của bạn dài 8000 tokens (company context, guidelines, examples). Bạn có 500 requests/ngày. Cost sẽ giảm bao nhiêu nếu dùng prompt caching?

<details>
<summary>Đáp án</summary>
Without caching: 500 × 8000 × $3/1M = $12/ngày
With caching (1 write + 499 reads):
  Write: 8000 × $3.75/1M = $0.03
  Reads: 499 × 8000 × $0.30/1M = $1.20
  Total: ~$1.23/ngày
→ Tiết kiệm ~90%, từ $360/tháng xuống ~$37/tháng.
</details>

**Q4**: Team của bạn muốn Claude Code access vào internal Jira để read tickets và create subtasks. Bạn cần build gì?

<details>
<summary>Đáp án</summary>
**MCP Server** — Build một Java MCP server expose Jira API như tools (read_ticket, create_subtask, etc.). Claude Code sẽ connect đến MCP server và có thể gọi các tools này trong conversation.
</details>

---

*Tiếp theo: [Lesson 03 — Thiết lập môi trường phát triển](./03-setup.md)*

*Quay lại: [Lesson 01 — Mental Model](./01-mental-model.md)*
