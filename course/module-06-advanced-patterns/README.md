# Module 06: Advanced Agentic Patterns

> **Level**: Advanced | **Duration**: 4 tuần (~ 40 giờ) | **Language**: Vietnamese + English technical terms

---

## Tại sao Module này quan trọng?

Sau 5 module đầu, bạn đã biết:
- Gọi Claude API từ Java (Module 01)
- Dùng Claude Code để automate workflows (Module 02)
- Xây MCP server để extend Claude với custom tools (Module 03)
- Orchestrate multi-agent pipelines (Module 04)
- Ship production-ready AI features (Module 05)

Nhưng có một **khoảng cách lớn** giữa _"nó chạy được trên máy tôi"_ và _"nó đang chạy production xử lý 10,000 requests/ngày."_

Khoảng cách đó chính xác là những gì Module 06 giải quyết.

---

### The Gap: "It Works" vs "Production Grade"

```
"It works"                          "Production Grade"
─────────────────────────────────────────────────────────────
Agent chạy happy path            → Xử lý được failure ở mọi step
Restart từ đầu khi fail          → Checkpoint và resume từ step N
Output không nhất quán          → Structured output + validation
Agent "tự nghĩ" mà không plan   → Explicit planning trước khi act
Một agent làm tất cả            → Specialized agents phối hợp
Không biết agent đang làm gì    → Full observability + tracing
Token cost không kiểm soát      → Context management + caching
Không test được                  → Eval harness + regression tests
```

Mỗi dòng trên tương ứng với một hoặc nhiều bài học trong module này.

---

## Danh sách bài học

| # | Tên bài | Thời gian | Pattern |
|---|---------|-----------|---------|
| 01 | [ReAct Pattern — Reasoning + Acting](lessons/01-react-pattern.md) | 4 giờ | ReAct Loop |
| 02 | [Planning Patterns — Plan Before You Act](lessons/02-planning-patterns.md) | 4 giờ | CoT, Plan-Execute, ToT |
| 03 | [State Management — Checkpointing & Resumability](lessons/03-state-management.md) | 4 giờ | Stateful Agents |
| 04 | Structured Output & Output Validation | 3 giờ | JSON Schema, Guards |
| 05 | Context Window Management | 3 giờ | Caching, Summarization |
| 06 | Error Handling & Retry Strategies | 4 giờ | Circuit Breaker, Fallback |
| 07 | Agent Observability & Tracing | 4 giờ | OpenTelemetry, Logging |
| 08 | Evaluating Agentic Systems | 5 giờ | Eval Harness, Metrics |
| 09 | Capstone: Production-Grade Agent Pipeline | 9 giờ | All patterns combined |

---

## Prerequisites

Trước khi bắt đầu module này, bạn cần hoàn thành:

- [x] **Module 00** — Foundation: AI/LLM concepts, prompt engineering cơ bản
- [x] **Module 01** — Claude API với Java: `AnthropicClient`, `MessageCreateParams`, tool use
- [x] **Module 02** — Claude Code: agentic coding workflows, slash commands
- [x] **Module 03** — MCP: xây MCP server, expose tools cho Claude
- [x] **Module 04** — Multi-Agent: orchestrator/worker patterns, agent pipelines
- [x] **Module 05** — Portfolio: ship production features, auth, billing integration

Nếu bạn chưa hoàn thành các module trên, hãy quay lại — module này giả định bạn đã viết Java code gọi Claude API thành thạo và hiểu cơ bản về tool use.

---

## Capstone Project: Production-Grade Code Review Pipeline

Xuyên suốt module này, bạn sẽ xây dựng từng bước một hệ thống **Code Review Pipeline** hoàn chỉnh cho Java codebase — một hệ thống thực sự production-ready.

### Mô tả

Khi developer mở Pull Request, hệ thống sẽ:

1. **Phân tích PR** — đọc diff, hiểu ngữ cảnh, lập kế hoạch review
2. **Chạy ReAct loop** — reasoning + calling tools (checkstyle, PMD, custom linters)
3. **Checkpoint progress** — nếu timeout hoặc fail, resume từ bước dừng
4. **Generate structured output** — comments theo JSON schema, không hallucinate
5. **Manage context** — large PRs không vượt context window
6. **Handle errors gracefully** — retry transient failures, fallback khi cần
7. **Trace everything** — mỗi agent decision đều có trace để debug
8. **Evaluate quality** — automated eval để đảm bảo review quality không giảm

### Tech Stack

```
Java 21 + Spring Boot 3.x
Anthropic Java SDK
PostgreSQL (checkpoints, eval results)
Redis (fast state cache)
OpenTelemetry (tracing)
GitHub API (PR integration)
```

### Kiến trúc cuối cùng

```
┌─────────────────────────────────────────────────────────────────┐
│                    Code Review Pipeline                          │
│                                                                  │
│  GitHub PR ──► Orchestrator                                      │
│                    │                                             │
│                    ├─► PlannerAgent (Lesson 02)                  │
│                    │       └─► Review Plan (5-10 steps)          │
│                    │                                             │
│                    ├─► ReActAgent (Lesson 01)                    │
│                    │       ├─► Thought: analyze diff             │
│                    │       ├─► Action: run_checkstyle(file)      │
│                    │       ├─► Observation: 3 violations found   │
│                    │       └─► ... (up to 15 steps)              │
│                    │                                             │
│                    ├─► CheckpointStore (Lesson 03)               │
│                    │       └─► PostgreSQL / Redis                │
│                    │                                             │
│                    ├─► StructuredOutputValidator (Lesson 04)     │
│                    │       └─► JSON Schema validation            │
│                    │                                             │
│                    ├─► ContextManager (Lesson 05)                │
│                    │       └─► Summarize old turns               │
│                    │                                             │
│                    ├─► ErrorHandler (Lesson 06)                  │
│                    │       └─► Retry + Circuit Breaker           │
│                    │                                             │
│                    ├─► TracingLayer (Lesson 07)                  │
│                    │       └─► OpenTelemetry spans               │
│                    │                                             │
│                    └─► EvalHarness (Lesson 08)                   │
│                            └─► Regression tests                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Cách học hiệu quả nhất

### Approach được khuyến nghị

1. **Đọc lý thuyết trước** — Hiểu _tại sao_ trước khi code
2. **Chạy code examples** — Copy code vào project, chạy thử, quan sát
3. **Làm exercise** — Mỗi bài có 1 exercise thực hành, đừng bỏ qua
4. **Build capstone incrementally** — Sau mỗi bài, thêm pattern đó vào capstone

### Môi trường setup

```bash
# Clone starter project (nếu chưa có)
git clone https://github.com/your-org/module-06-starter

# Cấu trúc project
module-06-starter/
├── src/main/java/
│   └── com/course/module06/
│       ├── lesson01/          # ReAct
│       ├── lesson02/          # Planning
│       ├── lesson03/          # State Management
│       └── capstone/          # Code Review Pipeline
├── src/test/java/
└── docker-compose.yml         # PostgreSQL + Redis

# Start infrastructure
docker-compose up -d

# Required env vars
export ANTHROPIC_API_KEY=sk-ant-...
export DATABASE_URL=jdbc:postgresql://localhost:5432/module06
export REDIS_URL=redis://localhost:6379
```

---

## Learning Outcomes

Sau khi hoàn thành Module 06, bạn sẽ có khả năng:

- **Implement ReAct loops** trong Java với proper stopping conditions
- **Choose đúng planning pattern** cho từng loại task (CoT, Plan-Execute, ToT)
- **Add checkpointing** vào bất kỳ agent nào để support resumability
- **Validate structured output** từ Claude với JSON Schema
- **Manage context window** cho long-running agents
- **Handle errors** ở mọi layer của agent pipeline
- **Instrument agents** với OpenTelemetry cho production observability
- **Write evals** để test agentic behavior một cách systematic
- **Ship** một production-grade agent pipeline end-to-end

---

## Tài liệu tham khảo

- [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629) — Yao et al., 2022
- [Tree of Thoughts](https://arxiv.org/abs/2305.10601) — Yao et al., 2023
- [Anthropic Agentic AI Overview](https://docs.anthropic.com/en/docs/build-with-claude/agents-overview)
- [Anthropic Java SDK](https://github.com/anthropics/anthropic-sdk-java)
- [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) — Anthropic, 2024

---

> **Bắt đầu**: [Lesson 01 — ReAct Pattern](lessons/01-react-pattern.md)
