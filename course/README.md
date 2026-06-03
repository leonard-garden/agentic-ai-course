# Master Claude Code & Agentic AI — for Java Backend Engineers

> Khóa học thực chiến dành riêng cho Java Backend Engineers muốn làm chủ công nghệ AI Agentic và tích hợp vào hệ thống production.

---

## Giới thiệu khóa học

Bạn đã có 5 năm kinh nghiệm với Java, Spring Boot, microservices. Bạn biết cách xây dựng REST APIs vững chắc, thiết kế database schema, debug production issues. Nhưng thế giới đang thay đổi — AI không còn là "nice to have" mà đang trở thành yêu cầu cốt lõi trong mọi hệ thống hiện đại.

Khóa học này **không dạy bạn dùng ChatGPT** hay viết prompts đơn giản. Khóa học này dạy bạn **tư duy và xây dựng Agentic AI systems** — những hệ thống AI có thể lên kế hoạch, thực thi nhiều bước, sử dụng công cụ, và giải quyết các bài toán phức tạp trong môi trường production.

### Bạn sẽ học được gì?

Sau khóa học này, bạn có thể:

- **Hiểu sâu** Anthropic ecosystem: Claude API, Claude Code, Agent SDK, MCP protocol
- **Thiết kế** multi-agent workflows cho các bài toán thực tế trong Java/Spring Boot
- **Tích hợp** Claude API vào Java projects với production-grade patterns (retry, caching, error handling)
- **Xây dựng** MCP servers bằng Java để cấp công cụ cho AI agents
- **Sử dụng** Claude Code như một senior pair programmer thực sự
- **Deploy** agentic systems với monitoring, observability, và cost control
- **Portfolio** 3 dự án thực chiến có thể show cho nhà tuyển dụng

---

## Đối tượng học viên

Khóa học này **phù hợp** với bạn nếu:

- Java Backend Engineer với 3–7 năm kinh nghiệm
- Đã làm việc với Spring Boot, REST APIs, microservices
- Đã biết PostgreSQL/MySQL và cơ bản về database design
- Muốn chuyển từ "biết dùng AI" sang "biết xây dựng AI systems"
- Muốn tăng productivity cá nhân với Claude Code

Khóa học này **không phù hợp** nếu bạn:

- Mới học lập trình, chưa biết Java
- Tìm kiếm một "AI tool" đơn giản, không muốn hiểu sâu
- Không sẵn sàng đọc tài liệu tiếng Anh (Anthropic docs, source code)

---

## Prerequisites

### Kỹ năng bắt buộc

| Kỹ năng | Mức độ yêu cầu | Tại sao cần |
|---------|----------------|-------------|
| Java (8+) | Thành thạo | Toàn bộ code examples dùng Java |
| Spring Boot | Intermediate | Module 4, 5 xây dựng Spring Boot agents |
| REST APIs | Thành thạo | Tích hợp với Claude API, MCP |
| Maven/Gradle | Cơ bản | Quản lý dependencies |
| PostgreSQL | Cơ bản | Một số exercises dùng database |
| Git | Cơ bản | Version control cho projects |
| Terminal/CLI | Cơ bản | Dùng Claude Code từ terminal |

### Kỹ năng khuyến nghị (không bắt buộc)

| Kỹ năng | Lợi ích |
|---------|---------|
| Docker | Deploy exercises dễ hơn |
| Kotlin | Một số examples có Kotlin variant |
| TypeScript | Đọc hiểu MCP SDK examples |
| Redis | Caching patterns trong Module 4 |

### Chuẩn bị môi trường

```bash
# 1. Java 21 LTS (khuyến nghị) hoặc Java 17+
java --version  # java 21.x.x

# 2. Maven 3.9+ hoặc Gradle 8+
mvn --version

# 3. Claude Code CLI
npm install -g @anthropic-ai/claude-code
claude --version

# 4. Anthropic API key (trả phí hoặc dùng free tier)
export ANTHROPIC_API_KEY="sk-ant-..."

# 5. IDE: IntelliJ IDEA (khuyến nghị) hoặc VS Code với Java Extension Pack
```

---

## Cấu trúc khóa học

| Module | Tên | Thời gian | Key Skills |
|--------|-----|-----------|------------|
| **00** | Foundation & Mental Model | 1 tuần | AI primitives, Anthropic ecosystem, mental model |
| **01** | Claude API & Java SDK | 2 tuần | anthropic-sdk-java, streaming, tool use, caching |
| **02** | Claude Code Mastery | 1 tuần | Workflow, slash commands, multi-file refactoring |
| **03** | MCP Protocol & Java | 2 tuần | Build MCP servers, custom tools, Spring Boot integration |
| **04** | Multi-Agent Systems | 2 tuần | Agent patterns, orchestration, workflows |
| **05** | Portfolio Projects | 2 tuần | 3 real-world projects end-to-end |

**Tổng thời gian**: ~10 tuần (học bán thời gian, ~1–2 giờ/ngày)

---

## Chi tiết từng Module

### Module 00 — Foundation & Mental Model (1 tuần)
> **Mục tiêu**: Hiểu đúng Agentic AI là gì và Anthropic ecosystem hoạt động như thế nào

- Lesson 01: [Mental Model — Agentic AI khác gì Traditional Software](./module-00-foundation/lessons/01-mental-model.md)
- Lesson 02: [Anthropic Ecosystem — Claude API, SDK, MCP, Agent SDK](./module-00-foundation/lessons/02-anthropic-ecosystem.md)
- Lesson 03: Thiết lập môi trường phát triển (Java + Claude Code)

### Module 01 — Claude API & Java SDK (2 tuần)
> **Mục tiêu**: Tích hợp Claude API vào Java/Spring Boot production-grade

- Lesson 01: anthropic-sdk-java — Setup, basic calls, error handling
- Lesson 02: Streaming responses — Server-Sent Events trong Spring Boot
- Lesson 03: Tool Use (Function Calling) — Cấp công cụ cho Claude
- Lesson 04: Prompt Caching — Giảm chi phí 90% cho repeated context
- Lesson 05: Batch API — Xử lý hàng nghìn requests async
- Lab: Build một AI-powered code review service

### Module 02 — Claude Code Mastery (1 tuần)
> **Mục tiêu**: Dùng Claude Code như senior pair programmer

- Lesson 01: Claude Code workflow — Cách "nói chuyện" với code hiệu quả
- Lesson 02: Multi-file refactoring — Refactor Spring Boot monolith
- Lesson 03: Test generation — TDD với Claude Code
- Lesson 04: Custom slash commands và CLAUDE.md
- Lab: Refactor một legacy Spring Boot app với Claude Code

### Module 03 — MCP Protocol & Java (2 tuần)
> **Mục tiêu**: Xây dựng MCP servers để cấp custom tools cho AI

- Lesson 01: MCP Protocol — Architecture và design principles
- Lesson 02: Java MCP SDK — Build tools, resources, prompts
- Lesson 03: Spring Boot MCP Server — Production-ready implementation
- Lesson 04: Database Tools — PostgreSQL access qua MCP
- Lesson 05: Testing và debugging MCP servers
- Lab: Build một MCP server cho internal ticketing system

### Module 04 — Multi-Agent Systems (2 tuần)
> **Mục tiêu**: Thiết kế và implement agentic workflows phức tạp

- Lesson 01: Agent patterns — Khi nào dùng patterns nào
- Lesson 02: Orchestrator-Worker pattern trong Java
- Lesson 03: State management cho long-running agents
- Lesson 04: Error handling và recovery trong agent systems
- Lesson 05: Cost optimization và monitoring
- Lab: Build một autonomous code migration agent

### Module 05 — Portfolio Projects (2 tuần)
> **Mục tiêu**: 3 dự án hoàn chỉnh, có thể deploy và show CV

- Project 1: AI Code Review Bot (Spring Boot + GitHub webhooks)
- Project 2: Intelligent API Documentation Generator
- Project 3: Autonomous Database Migration Assistant

---

## Cách học hiệu quả nhất

### Nguyên tắc 1: Học qua làm (Learning by Doing)

Mỗi lesson có một **Lab** hoặc **Exercise** cụ thể. Đừng đọc xong rồi bỏ qua — hãy thực hành ngay. Nếu bạn chỉ đọc mà không code, bạn sẽ quên 80% sau 1 tuần.

```
Đọc concept (15 phút) → 
Chạy code example (15 phút) → 
Tự làm exercise (30–45 phút) → 
Review và note lại những gì chưa hiểu
```

### Nguyên tắc 2: Connect with what you know

Bạn đã biết Spring Boot, microservices. Mỗi khái niệm AI mới, hãy tự hỏi: *"Cái này giống cái gì tôi đã biết trong Java world?"*

- LLM = external service với latency cao và non-deterministic output
- Agent = long-running background job với decision making
- MCP tool = REST endpoint mà AI có thể gọi
- Context window = request payload với hard size limit

### Nguyên tắc 3: Build something real ASAP

Đừng hoàn thành toàn bộ theory trước khi bắt đầu build. Sau Module 01, hãy tích hợp Claude API vào một project thực của bạn — dù nhỏ. Real context giúp bạn học nhanh gấp 3 lần.

### Nguyên tắc 4: Đọc Anthropic docs gốc

Anthropic có tài liệu cực kỳ chất lượng. Mỗi bài trong khóa học này đều link đến docs gốc. Hãy đọc ít nhất một lần để hiểu đúng nguồn.

### Nguyên tắc 5: Theo dõi chi phí API

Bạn sẽ tốn tiền API khi học. Thiết lập spending limit ngay từ đầu:
- **Học**: $5–10/tháng là đủ nếu dùng Haiku thông minh
- **Projects**: $20–30/tháng
- Dùng prompt caching để giảm chi phí khi lặp lại experiments

---

## Quick Start (30 phút đầu tiên)

### Bước 1: Tạo Anthropic account và lấy API key

1. Truy cập [console.anthropic.com](https://console.anthropic.com)
2. Tạo account, thêm payment method
3. Set spending limit: $10/tháng cho giai đoạn học
4. Vào API Keys → Create Key → copy vào `.env`

### Bước 2: Cài Claude Code

```bash
npm install -g @anthropic-ai/claude-code
claude login  # hoặc set ANTHROPIC_API_KEY env var
claude --version  # kiểm tra cài đặt thành công
```

### Bước 3: Tạo Java project đầu tiên

```bash
# Tạo project với Maven
mvn archetype:generate \
  -DgroupId=com.example \
  -DartifactId=ai-learning \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DarchetypeVersion=1.4 \
  -DinteractiveMode=false

cd ai-learning
```

### Bước 4: Thêm Anthropic Java SDK

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.anthropic</groupId>
    <artifactId>anthropic-java</artifactId>
    <version>0.8.0</version>
</dependency>
```

### Bước 5: Hello World với Claude API

```java
import com.anthropic.client.Anthropic;
import com.anthropic.models.messages.*;

public class HelloClaude {
    public static void main(String[] args) {
        var client = Anthropic.builder()
            .apiKey(System.getenv("ANTHROPIC_API_KEY"))
            .build();

        var message = client.messages().create(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_SONNET_4_5)
                .maxTokens(1024)
                .addUserMessage("Xin chào! Giải thích Agentic AI trong 2 câu.")
                .build()
        );

        System.out.println(message.content().get(0));
    }
}
```

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
mvn compile exec:java -Dexec.mainClass="HelloClaude"
```

Nếu bạn thấy Claude trả lời → bạn đã sẵn sàng bắt đầu!

---

## Tracking Progress

Mỗi module có checklist riêng. Dùng file này để track:

```
[ ] Module 00 — Foundation (tuần 1)
    [ ] Lesson 01: Mental Model
    [ ] Lesson 02: Anthropic Ecosystem
    [ ] Lesson 03: Setup môi trường
    [ ] Quiz Module 00

[ ] Module 01 — Claude API & Java SDK (tuần 2-3)
    ...

[ ] Module 02 — Claude Code (tuần 4)
    ...
```

---

## Resources

### Tài liệu chính thức
- [Anthropic Documentation](https://docs.anthropic.com) — nguồn đáng tin cậy nhất
- [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) — bài đọc bắt buộc
- [Anthropic Academy](https://anthropic.com/academy) — free courses
- [anthropic-sdk-java GitHub](https://github.com/anthropics/anthropic-sdk-java)
- [Model Context Protocol Spec](https://spec.modelcontextprotocol.io)

### Cộng đồng
- [Anthropic Discord](https://discord.gg/anthropic)
- [Claude Code GitHub Discussions](https://github.com/anthropics/claude-code)

### Tools
- [Anthropic Console](https://console.anthropic.com) — API keys, usage, Workbench
- [Claude.ai](https://claude.ai) — Dùng Claude trực tiếp để test prompts
- [MCP Inspector](https://github.com/modelcontextprotocol/inspector) — Debug MCP servers

---

*Khóa học được thiết kế cho Claude Sonnet 4.6, cập nhật tháng 6/2026.*
