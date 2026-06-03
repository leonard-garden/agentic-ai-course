# Module 00 — Foundation & Mental Model

> **Thời gian**: 1 tuần (4–6 giờ học)
> **Difficulty**: Beginner
> **Prerequisites**: Không cần kinh nghiệm AI trước đó, chỉ cần nền tảng Java

---

## Tại sao Module này quan trọng?

Hầu hết developers mắc lỗi khi học AI: họ nhảy thẳng vào code mà không xây dựng mental model đúng. Kết quả là họ dùng AI như một "smart autocomplete" thay vì như một **reasoning engine** có thể tham gia vào các workflows phức tạp.

Module này xây dựng nền tảng tư duy đúng. Sau module này, bạn sẽ thấy sự khác biệt rõ ràng giữa:
- Gọi Claude API một lần → nhận text → xong
- Xây dựng một **agent** có thể lên kế hoạch, dùng tools, tự sửa lỗi, và hoàn thành multi-step tasks

---

## Learning Objectives

Sau khi hoàn thành Module 00, bạn có thể:

- [ ] Giải thích Agentic AI khác Traditional Software ở điểm nào (mà không cần nhìn notes)
- [ ] Phân biệt được khi nào dùng **Messages API** vs **Managed Agents**
- [ ] Nhận ra 5 workflow patterns từ Anthropic và map chúng sang Java microservices concepts
- [ ] Vẽ sơ đồ toàn bộ Anthropic ecosystem từ trí nhớ
- [ ] Chọn đúng Claude model (Opus/Sonnet/Haiku) cho từng use case
- [ ] Ước tính chi phí token cho một use case cụ thể
- [ ] Thiết lập Java project với anthropic-sdk-java và chạy được Hello World

---

## Lessons

### Lesson 01 — Mental Model: Agentic AI là gì?
**Thời gian**: 1.5 giờ | **File**: [lessons/01-mental-model.md](./lessons/01-mental-model.md)

Xây dựng mental model đúng về Agentic AI. Hiểu LLM as a reasoning engine. Học 5 workflow patterns từ Anthropic và map sang Java/microservices.

**Key concepts**: Agentic AI, LLM primitives, Workflows vs Agents, Prompt chaining, Routing, Parallelization, Orchestrator-Workers, Evaluator-Optimizer

---

### Lesson 02 — Anthropic Ecosystem
**Thời gian**: 1.5 giờ | **File**: [lessons/02-anthropic-ecosystem.md](./lessons/02-anthropic-ecosystem.md)

Toàn cảnh Anthropic ecosystem. Model family, token economics, Java SDK overview, kiến trúc quyết định khi nào dùng gì.

**Key concepts**: Claude API, Claude Code, MCP, Managed Agents, Opus/Sonnet/Haiku, Token economics, Prompt caching, Java SDK

---

### Lesson 03 — Thiết lập môi trường phát triển
**Thời gian**: 1 giờ | **File**: [lessons/03-setup.md](./lessons/03-setup.md)

Cài đặt và cấu hình toàn bộ stack: Java 21, Maven, anthropic-sdk-java, Claude Code CLI, IntelliJ IDEA với AI extensions.

**Key concepts**: Environment setup, API key management, `.env` files, First API call, Claude Code login

---

## What You'll Build

Cuối Module 00, bạn sẽ có:

1. **Mental Model Document** — Một sơ đồ tự vẽ (whiteboard hoặc draw.io) về Anthropic ecosystem, annotated với các quyết định kiến trúc cho một Java project cụ thể

2. **Working Java Project** — Project Maven/Gradle với `anthropic-sdk-java` đã setup, có thể gọi Claude API thành công

3. **Decision Cheatsheet** — File markdown cá nhân tóm tắt: khi nào dùng Haiku/Sonnet/Opus, khi nào dùng Messages API vs Managed Agents

---

## Module Quiz

Sau khi học xong cả 3 lessons, tự ôn lại bằng cách vẽ sơ đồ Anthropic ecosystem và map vào project Java của bạn (exercise cuối Lesson 02).

---

## Chuyển sang Module tiếp theo

Khi bạn có thể:
- Trả lời 3 câu hỏi trong mục Learning Objectives mà không nhìn notes
- Chạy Hello World Java với Claude API thành công
- Vẽ lại sơ đồ ecosystem từ trí nhớ trong 5 phút

→ Bạn sẵn sàng cho [Module 01 — Claude API & Java SDK](../module-01-claude-api-java/README.md)
