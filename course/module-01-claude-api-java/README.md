# Module 01: Claude API với Java

> **Target**: Java backend engineer 5+ năm kinh nghiệm  
> **Thời lượng**: 3 tuần (12 buổi, ~2-3 giờ/buổi)  
> **Ngôn ngữ**: Vietnamese + English technical terms  
> **Level**: Beginner → Intermediate (AI Engineering)

---

## Tổng quan module

Module này là điểm khởi đầu cho hành trình AI Engineering với Java. Sau 3 tuần, bạn sẽ có khả năng tích hợp Claude API vào bất kỳ Java/Spring Boot application nào, xây dựng các AI-powered features thực tế, và hiểu rõ cách LLMs hoạt động từ góc nhìn của một backend engineer.

**Tại sao module này quan trọng?**

Với 5 năm kinh nghiệm Java backend, bạn đã quen với các khái niệm như REST API, database, service layer. AI Engineering không yêu cầu bạn học lại từ đầu — nó thêm một "layer" mới vào architecture quen thuộc. Claude API về bản chất là một HTTP API với JSON payload, không khác gì các third-party APIs bạn đã dùng hàng ngày.

Điểm khác biệt: với AI, **prompt là code**. Cách bạn viết instructions cho model ảnh hưởng trực tiếp đến chất lượng output, giống như cách bạn viết SQL ảnh hưởng đến performance của query.

---

## Prerequisites

### Java knowledge (bạn đã có)
- Java 8+ (lambdas, Optional, CompletableFuture)
- Maven/Gradle dependency management
- Spring Boot basics (REST controllers, services)
- HTTP client fundamentals
- JSON serialization/deserialization (Jackson)

### Cần chuẩn bị
- Anthropic account (free tier available): https://console.anthropic.com
- Java 11+ (khuyến nghị Java 17 LTS)
- IDE: IntelliJ IDEA hoặc VS Code với Java Extension Pack
- Maven 3.8+ hoặc Gradle 7+
- Git (để clone code examples)

---

## Cấu trúc module

```
module-01-claude-api-java/
├── README.md                    ← File này
├── lessons/
│   ├── 01-setup.md              ← Tuần 1, Buổi 1-2
│   ├── 02-messages-and-streaming.md  ← Tuần 1, Buổi 3-4
│   ├── 03-tool-use.md           ← Tuần 2, Buổi 5-7
│   └── 04-prompt-engineering.md ← Tuần 2-3, Buổi 8-10
├── exercises/
│   └── project-ai-database-assistant.md  ← Tuần 3, Buổi 11-12
└── code/
    └── (code examples cho từng lesson)
```

---

## Lịch học 3 tuần

### Tuần 1: Foundation — Hiểu Claude API

| Buổi | Lesson | Nội dung | Thời gian |
|------|--------|----------|-----------|
| 1 | 01-setup | Setup environment, first API call | 2 giờ |
| 2 | 01-setup | Error handling, best practices, exercise | 2 giờ |
| 3 | 02-messages | Message structure, multi-turn conversations | 2.5 giờ |
| 4 | 02-messages | Streaming, async, prompt caching | 2.5 giờ |

**Tuần 1 deliverable**: CLI chatbot đơn giản có khả năng multi-turn conversation với streaming output.

### Tuần 2: Core Skills — Tool Use & Prompt Engineering

| Buổi | Lesson | Nội dung | Thời gian |
|------|--------|----------|-----------|
| 5 | 03-tool-use | Tool use concept, defining tools | 2.5 giờ |
| 6 | 03-tool-use | Handling tool responses, full flow | 2.5 giờ |
| 7 | 03-tool-use | Multiple tools, parallel calls, error handling | 3 giờ |
| 8 | 04-prompt-engineering | Core techniques, system prompts | 2.5 giờ |
| 9 | 04-prompt-engineering | Few-shot, CoT, structured output | 2.5 giờ |
| 10 | 04-prompt-engineering | Prompt testing, templates | 2 giờ |

**Tuần 2 deliverable**: Java service có 3 tools (DB search, REST call, file read) với intelligent routing.

### Tuần 3: Integration Project

| Buổi | Project | Nội dung | Thời gian |
|------|---------|----------|-----------|
| 11 | AI DB Assistant | Setup, tool implementation, testing | 3 giờ |
| 12 | AI DB Assistant | Spring Boot integration, polish, demo | 3 giờ |

**Tuần 3 deliverable**: Production-ready AI Database Assistant — Spring Boot app cho phép query database bằng natural language.

---

## Learning Objectives

Sau khi hoàn thành module, bạn sẽ:

### Kỹ năng kỹ thuật
1. **Setup & Integration**: Tích hợp `anthropic-sdk-java` vào Maven/Gradle project
2. **Messages API**: Gửi requests, xử lý responses, manage conversation history
3. **Streaming**: Implement real-time streaming responses trong Java
4. **Tool Use**: Xây dựng AI agents có khả năng gọi Java functions/services
5. **Prompt Engineering**: Viết prompts hiệu quả với few-shot examples, CoT, XML structure
6. **Production Patterns**: Error handling, retry logic, rate limiting, cost optimization

### Tư duy AI Engineering
- Hiểu LLM là **probabilistic text generator**, không phải deterministic program
- Biết khi nào nên dùng AI, khi nào nên dùng traditional code
- Debug và test AI features (không có stack trace rõ ràng)
- Cost awareness: estimate và optimize token usage

---

## Công nghệ sử dụng

| Technology | Version | Mục đích |
|-----------|---------|---------|
| `anthropic-sdk-java` | latest | Anthropic API client |
| Java | 17 (LTS) | Runtime |
| Spring Boot | 3.x | Web framework (lessons 2, 4, project) |
| Maven | 3.8+ | Build tool |
| Jackson | 2.15+ | JSON serialization |
| SLF4J/Logback | standard | Logging |

**Maven dependency chính:**
```xml
<dependency>
    <groupId>com.anthropic</groupId>
    <artifactId>anthropic-sdk-java</artifactId>
    <version>0.8.0-alpha.1</version>
</dependency>
```

> **Lưu ý**: `anthropic-sdk-java` là official SDK từ Anthropic, Java 8+ compatible, MIT license.  
> GitHub: https://github.com/anthropics/anthropic-sdk-java

---

## Cost & Token Awareness

Với Anthropic API, bạn trả tiền theo số tokens. Quan trọng từ ngày đầu:

| Model | Input (per 1M tokens) | Output (per 1M tokens) |
|-------|----------------------|------------------------|
| Claude Sonnet 4.6 | $3 | $15 |
| Claude Haiku 3.5 | $0.80 | $4 |
| Claude Opus 4 | $15 | $75 |

**Rule of thumb cho engineers**:
- Development/testing: dùng **Claude Haiku** (rẻ, nhanh)
- Production features: dùng **Claude Sonnet** (balance)
- Complex reasoning: dùng **Claude Opus** (khi cần thiết)

Trong module này, examples dùng `CLAUDE_SONNET_4_6` để demonstrate quality, nhưng bạn có thể swap sang Haiku khi testing.

---

## Cách tiếp cận học tập

### Mindset shift quan trọng

Là Java engineer, bạn quen với **deterministic systems**: cùng input → cùng output. LLMs không như vậy:

```
Input:  "Translate 'Hello' to French"
Run 1:  "Bonjour"
Run 2:  "Bonjour!"  
Run 3:  "Salut"      ← Valid answer, different phrasing
```

Điều này không phải bug — đây là feature. Nhưng nó đòi hỏi cách tiếp cận khác:
- Test với nhiều runs, không chỉ 1 lần
- Dùng temperature=0 cho deterministic tasks (code generation, structured output)
- Design cho "good enough" thay vì "perfect"

### Học từ code, không từ theory

Mỗi lesson có ít nhất 1 working code example. Quy trình:
1. Đọc code example
2. Chạy thử, verify output
3. Thay đổi một tham số, xem ảnh hưởng
4. Làm exercise
5. Build mini project

---

## Quick Start (5 phút)

Muốn xem kết quả ngay trước khi đọc chi tiết? Làm theo 4 bước:

**Bước 1**: Tạo account tại https://console.anthropic.com và lấy API key

**Bước 2**: Export API key
```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

**Bước 3**: Tạo Maven project với dependency:
```xml
<dependency>
    <groupId>com.anthropic</groupId>
    <artifactId>anthropic-sdk-java</artifactId>
    <version>0.8.0-alpha.1</version>
</dependency>
```

**Bước 4**: Chạy code này:
```java
import com.anthropic.sdk.AnthropicClient;
import com.anthropic.sdk.models.*;

public class QuickStart {
    public static void main(String[] args) {
        AnthropicClient client = AnthropicClient.builder()
            .apiKey(System.getenv("ANTHROPIC_API_KEY"))
            .build();

        Message message = client.messages().create(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_SONNET_4_6)
                .maxTokens(256)
                .addUserMessage("Explain Java generics in 2 sentences.")
                .build()
        );

        System.out.println(message.content().get(0));
    }
}
```

Nếu thấy response từ Claude, bạn đã sẵn sàng bắt đầu! Chuyển sang [Lesson 01](./lessons/01-setup.md).

---

## Resources

### Official Documentation
- Anthropic API Docs: https://docs.anthropic.com
- SDK Java GitHub: https://github.com/anthropics/anthropic-sdk-java
- Model overview: https://docs.anthropic.com/en/docs/about-claude/models

### Community
- Anthropic Discord: https://discord.gg/anthropic
- Course GitHub (code examples): xem instructor

### Tools
- Anthropic Console (playground): https://console.anthropic.com
- Token counter: https://anthropic-tokenizer.netlify.app

---

## Evaluation & Grading

| Component | Weight | Description |
|-----------|--------|-------------|
| Exercises (4 bài) | 40% | Một exercise mỗi lesson |
| Mini project cuối tuần 2 | 20% | Tool use service với 3 tools |
| Final project (AI DB Assistant) | 40% | Production-ready Spring Boot app |

**Pass threshold**: 70% tổng điểm, phải complete final project.

---

*Module tiếp theo: **Module 02 — Agentic Systems**: Xây dựng multi-agent workflows, memory management, long-running tasks với Java.*
