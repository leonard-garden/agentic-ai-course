# Module 05: Portfolio & Career Development

> **Dành cho:** Java Backend Engineer 5+ năm kinh nghiệm đang chuyển sang AI Engineering
> **Thời lượng:** Ongoing — cập nhật liên tục cùng với market
> **Mục tiêu:** Xây dựng portfolio AI Engineering đủ mạnh để landing job hoặc promote trong 6-12 tháng

---

## Tại sao Portfolio quan trọng với AI Engineer?

AI Engineering là ngành mới. Không có tiêu chuẩn tuyển dụng rõ ràng như Software Engineering truyền thống — không có "5 năm Spring Boot" hay "senior Java developer" mà nhà tuyển dụng quen thuộc. Thay vào đó, **portfolio là tiếng nói lớn nhất**.

Khi hiring manager nhìn vào CV của bạn, họ đang hỏi:

- *"Engineer này có thực sự build AI systems không, hay chỉ dùng ChatGPT?"*
- *"Họ hiểu production AI — failure modes, cost, latency — hay chỉ biết happy path?"*
- *"Họ có thể integrate AI vào enterprise stack (Java, Spring, legacy systems) không?"*

Portfolio trả lời tất cả những câu hỏi đó bằng **bằng chứng cụ thể**, không phải bằng lời nói.

### Portfolio vs Resume: Sự khác biệt quyết định

| Resume nói | Portfolio chứng minh |
|-----------|---------------------|
| "Có kinh nghiệm Claude API" | GitHub repo với Claude integration đang chạy |
| "Hiểu multi-agent systems" | JavaGuard chạy trên real PR với comments |
| "Biết MCP protocol" | MCP server published, 50+ GitHub stars |
| "Production-ready AI" | Monitoring dashboard, cost tracking, error handling |

---

## Cấu trúc Module

```
module-05-portfolio/
├── README.md                    ← Bạn đang đọc file này
├── lessons/
│   └── 01-career-path.md        ← Career tracks, market signals, positioning
└── projects/
    ├── 01-enterprise-mcp-suite.md      ⭐ Showcase piece
    ├── 02-ai-code-review-pipeline.md   ⭐ Team-level impact
    ├── 03-intelligent-incident-response.md  ⭐ Production maturity
    └── 04-autonomous-doc-generator.md  ⭐ Advanced agentic patterns
```

---

## 4 Portfolio Projects

Mỗi project được thiết kế để demonstrate một aspect khác nhau của AI engineering:

### Project 1: JavaOps MCP Suite ⭐
**File:** `projects/01-enterprise-mcp-suite.md`

Connect Claude Code trực tiếp vào Spring Boot application internals. Đây là **showcase piece** — Java-native, enterprise use case, hot skill (MCP).

- **Demonstrates:** MCP protocol, Java/Spring integration, tool design
- **Differentiator:** Hầu hết AI engineers là Python — bạn mang Java expertise
- **Time to build:** 2-3 ngày focused work

### Project 2: JavaGuard — AI Code Review Pipeline ⭐
**File:** `projects/02-ai-code-review-pipeline.md`

Automated multi-agent code review tích hợp vào GitHub Actions. Review mọi PR tự động.

- **Demonstrates:** Multi-agent orchestration, GitHub integration, production cost management
- **Differentiator:** End-to-end agentic system với real business value
- **Time to build:** 3-4 ngày

### Project 3: IntelliOps — Incident Response Agent ⭐
**File:** `projects/03-intelligent-incident-response.md`

AI assistant giúp diagnose và resolve production incidents. Human-in-the-loop design.

- **Demonstrates:** Safety-first AI design, human oversight, high-stakes agentic tasks
- **Differentiator:** Shows production maturity — bạn nghĩ về safety, không chỉ features
- **Time to build:** 4-5 ngày

### Project 4: DocBot — Autonomous Documentation Generator ⭐
**File:** `projects/04-autonomous-doc-generator.md`

Self-maintaining documentation system dùng Evaluator-Optimizer pattern.

- **Demonstrates:** Advanced agentic patterns, CI/CD integration, quality loops
- **Differentiator:** Evaluator-optimizer là pattern ít người biết, rất impressive
- **Time to build:** 3-4 ngày

---

## Portfolio Presentation Strategy

### GitHub Profile Setup

Tạo một GitHub profile README (`username/username/README.md`) với:

```markdown
## AI Engineering Projects

| Project | Stack | What it does | Stars |
|---------|-------|-------------|-------|
| JavaOps MCP Suite | Java, Spring Boot, MCP | Connect Claude to Spring Boot internals | ⭐ |
| JavaGuard | Java, Python, GitHub Actions | Multi-agent PR code review | ⭐ |
| IntelliOps | Spring Boot, Claude | AI incident response with human oversight | ⭐ |
| DocBot | Java, Claude, Git | Self-maintaining codebase documentation | ⭐ |
```

### Thứ tự build portfolio

Đừng build tất cả cùng lúc — **build có chiến lược**:

1. **Tháng 1-2:** JavaOps MCP Suite (showcase piece, nhanh nhất)
2. **Tháng 2-3:** JavaGuard (team-level impact, dễ demo)
3. **Tháng 3-4:** IntelliOps (production maturity signal)
4. **Tháng 4-5:** DocBot (advanced patterns)

Sau mỗi project: write a blog post (LinkedIn/Medium/Zenn). Blog posts khuếch đại portfolio lên 3-5x.

---

## Học tiếp theo

Sau module này, bạn nên có:
- [ ] Ít nhất 2 completed portfolio projects trên GitHub
- [ ] LinkedIn profile được optimize cho "Java AI Engineer"
- [ ] 1-2 blog posts về những gì bạn đã build
- [ ] Hiểu rõ career track nào phù hợp với goals của bạn

Xem `lessons/01-career-path.md` để hiểu market landscape và cách position bản thân.
