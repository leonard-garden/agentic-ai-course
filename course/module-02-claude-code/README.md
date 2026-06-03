# Module 02: Claude Code & Agent SDK

> **Thời lượng**: 4 tuần | **Level**: Intermediate → Advanced  
> **Prerequisite**: Module 01 (Claude API & Java Integration)  
> **Target**: Java Backend Engineer 5+ năm kinh nghiệm

---

## Tổng quan Module

Module này đưa bạn từ việc *gọi Claude API thủ công* sang việc **làm chủ Claude Code** — công cụ agentic coding assistant mạnh nhất hiện tại — và **Agent SDK** để lập trình hóa toàn bộ workflow AI của bạn.

Sau module này, bạn sẽ không còn "chat với AI để viết code" nữa. Thay vào đó, bạn sẽ **thiết kế và vận hành một team AI agents** làm việc tự động trên codebase Java/Spring Boot của bạn: review security, viết test, debug, refactor — tất cả theo quy trình bạn định nghĩa.

---

## Tại sao Module này quan trọng với Java Engineer?

Với 5 năm kinh nghiệm, bạn đã biết:
- Code review mất nhiều thời gian, dễ bỏ sót
- Viết test là việc cần thiết nhưng hay bị trì hoãn
- Debug production issues lúc 2am là cực hình
- Onboarding codebase mới tốn hàng tuần

**Claude Code + Agent SDK giải quyết chính xác những pain points này.** Không phải bằng cách thay thế bạn, mà bằng cách cung cấp cho bạn một đội ngũ AI chuyên biệt mà bạn là kỹ sư trưởng điều phối.

---

## Lessons

| # | Lesson | Thời lượng | Trọng tâm |
|---|--------|-----------|-----------|
| 01 | [Claude Code Architecture & Internals](lessons/01-claude-code-architecture.md) | 1 tuần | Agent loop, tools, permissions, CLAUDE.md, hooks |
| 02 | [Custom Subagents — Build Your Agent Team](lessons/02-custom-subagents.md) | 1 tuần | Định nghĩa agents, YAML frontmatter, 5 agents thực tế |
| 03 | [Agent SDK — Programmatic Control](lessons/03-agent-sdk.md) | 1 tuần | Sessions, Python SDK, Java integration patterns |
| 04 | [Claude Code Workflows & Automation](lessons/04-claude-code-workflows.md) | 1 tuần | Hooks, slash commands, CLAUDE.md advanced, worktrees |

---

## Learning Outcomes

Sau khi hoàn thành module, bạn có thể:

### Kỹ năng kỹ thuật
- [ ] Giải thích được agent loop của Claude Code và tại sao nó khác chatbot
- [ ] Viết CLAUDE.md hiệu quả cho bất kỳ Java/Spring Boot project nào
- [ ] Thiết kế và deploy custom subagents với đúng permissions và model
- [ ] Viết hooks tự động hóa: format, test, commit, notify
- [ ] Dùng Agent SDK (Python/TypeScript) để orchestrate multi-agent workflows
- [ ] Tích hợp Agent SDK vào Java service qua subprocess hoặc REST wrapper

### Kỹ năng thiết kế
- [ ] Phân tích một workflow kỹ thuật và chia thành agents hợp lý
- [ ] Chọn đúng model (Haiku/Sonnet/Opus) cho từng loại agent
- [ ] Giới hạn permissions theo nguyên tắc least privilege
- [ ] Đo lường cost và optimize agent pipeline

---

## Tech Stack trong Module

```
Claude Code CLI          — agentic coding assistant
Agent SDK (Python)       — programmatic orchestration  
Java 21 + Spring Boot    — target codebase
Maven                    — build tool trong examples
~/.claude/agents/        — global agent storage
.claude/agents/          — project-level agent storage
~/.claude/settings.json  — global settings & hooks
```

---

## Cấu trúc thư mục

```
module-02-claude-code/
├── README.md                          ← bạn đang đọc
├── lessons/
│   ├── 01-claude-code-architecture.md
│   ├── 02-custom-subagents.md
│   ├── 03-agent-sdk.md
│   └── 04-claude-code-workflows.md
├── code/
│   ├── sample-agents/                 ← 5 subagent definitions
│   ├── agent-sdk-examples/            ← Python SDK scripts
│   └── spring-boot-demo/              ← demo project để test agents
└── exercises/
    ├── ex-01-claude-md.md
    ├── ex-02-custom-agents.md
    ├── ex-03-sdk-workflow.md
    └── ex-04-full-automation.md
```

---

## Cách học hiệu quả

### Nguyên tắc "Eat your own dog food"
Học Claude Code bằng cách **dùng Claude Code để học Claude Code**. Mỗi lesson, hãy mở Claude Code song song và thực hành ngay khi đọc.

### Quy trình mỗi lesson
1. **Đọc lý thuyết** (30 phút) — nắm concepts
2. **Chạy examples** (45 phút) — hands-on trên máy bạn
3. **Làm exercise** (60 phút) — áp dụng vào project thực của bạn
4. **Review lại** (15 phút) — ghi chú những gì surprising

### Đừng bỏ qua exercises
Exercises trong module này được thiết kế để áp dụng vào **project thực của bạn**, không phải project demo. Điều này có nghĩa:
- Bạn sẽ thấy giá trị ngay lập tức
- Bạn sẽ gặp edge cases thực tế
- Sau module, bạn có một agent team hoạt động trên codebase của mình

---

## Prerequisites Check

Trước khi bắt đầu, verify:

```bash
# Claude Code installed
claude --version
# Expected: claude x.x.x

# Node.js (required by Claude Code)
node --version
# Expected: v18+

# Python (for Agent SDK lessons)
python3 --version
# Expected: 3.10+

# Java project để test (nên có sẵn)
ls your-spring-boot-project/
```

Nếu chưa install Claude Code:
```bash
npm install -g @anthropic-ai/claude-code
claude login
```

---

## Mindset Shift

Bắt đầu module này với mindset:

> "Tôi không chỉ đang học một tool. Tôi đang học cách **design AI systems** — xác định responsibilities, phân chia permissions, orchestrate workflows — với Claude Code là runtime environment."

Đây là kỹ năng của **AI Engineer**, không phải **AI user**. Sự khác biệt này sẽ quyết định career trajectory của bạn trong 5 năm tới.

---

*Bắt đầu với [Lesson 01: Claude Code Architecture & Internals](lessons/01-claude-code-architecture.md)*
