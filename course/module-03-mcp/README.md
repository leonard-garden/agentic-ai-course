# Module 03: Model Context Protocol (MCP)

> **Thời lượng**: 3 tuần | **Level**: Intermediate → Advanced  
> **Prerequisite**: Module 01 (Claude API Java), Module 02 (Claude Code)

---

## Tổng quan

Module này đi sâu vào **Model Context Protocol (MCP)** — open standard protocol cho phép AI models kết nối với external tools, databases, và data sources theo cách chuẩn hóa, có thể tái sử dụng.

Nếu bạn đã làm việc với JDBC, JPA, hay bất kỳ database driver nào trong Java career — bạn đã hiểu 60% ý tưởng của MCP. Phần còn lại là apply nó cho AI.

---

## Tại sao Module này quan trọng với Java Backend Engineer?

Trước MCP, mỗi AI application phải tự integrate với từng tool riêng lẻ:
- Muốn Claude đọc database → viết custom code
- Muốn Claude browse GitHub → viết custom code
- Muốn Claude phân tích logs → viết custom code

Kết quả: **N AI apps × M tools = N×M integrations**. Không scale được.

MCP giải quyết bằng cách chuẩn hóa giao thức:
- Mỗi tool chỉ cần implement MCP server **một lần**
- Mọi MCP-compatible AI client đều dùng được ngay

Với Java backend engineer, MCP mở ra khả năng:
- Claude trực tiếp query production database để debug
- Claude đọc Spring Actuator endpoints để analyze application health
- Claude browse Flyway migrations để hiểu schema evolution
- Claude search codebase và GitHub issues cùng lúc

---

## Learning Objectives

Sau khi hoàn thành module này, bạn có thể:

1. **Giải thích** MCP architecture và tại sao nó tồn tại
2. **Configure** và sử dụng existing MCP servers (postgres, github, filesystem)
3. **Build** MCP server bằng Spring Boot expose Spring application internals
4. **Deploy** MCP server cho team sử dụng
5. **Design** MCP server architecture cho enterprise Java systems

---

## Cấu trúc Module

```
module-03-mcp/
├── README.md                          ← File này
├── lessons/
│   ├── 01-mcp-concepts.md            ← Tuần 1: Hiểu MCP từ góc độ Java
│   ├── 02-consuming-mcp-servers.md   ← Tuần 2: Dùng MCP servers có sẵn
│   └── 03-building-mcp-server-java.md ← Tuần 3: Build MCP server bằng Spring Boot
├── exercises/
│   └── project-enterprise-mcp-suite.md ← Capstone project
└── code/
    ├── actuator-mcp-server/           ← Spring Boot MCP server starter
    └── mcp-client-examples/           ← Java client code examples
```

---

## Lịch học theo tuần

### Tuần 1: MCP Concepts — "JDBC cho AI"
**Bài học**: `lessons/01-mcp-concepts.md`

- MCP là gì và tại sao nó ra đời
- JDBC analogy: hiểu MCP qua lens của Java developer
- Architecture deep-dive: Host, Server, Client
- 3 primitives: Tools, Resources, Prompts
- Transport: stdio vs SSE
- MCP ecosystem hiện tại (2026)

**Mục tiêu tuần 1**: Có mental model vững về MCP, giải thích được cho đồng nghiệp không cần slide deck.

---

### Tuần 2: Consuming MCP Servers
**Bài học**: `lessons/02-consuming-mcp-servers.md`

- Config MCP servers trong Claude Code
- Hands-on với 3 servers: filesystem, postgres, github
- Debug MCP connection issues
- Best practices cho team environment

**Mục tiêu tuần 2**: Setup và sử dụng thành thạo ít nhất 3 MCP servers trong Spring Boot project context.

---

### Tuần 3: Building MCP Server với Java
**Bài học**: `lessons/03-building-mcp-server-java.md`

- Spring AI MCP integration
- Build Spring Boot Actuator MCP Server
- Expose Resources (DB schema, config)
- Testing với MCP Inspector
- Security và deployment

**Mục tiêu tuần 3**: Ship một working MCP server mà team có thể dùng ngay.

---

### Capstone Project: JavaOps MCP Suite
**Project spec**: `exercises/project-enterprise-mcp-suite.md`

Build một production-ready MCP server suite với 7 tools, full test coverage, và deployment instructions.

---

## Tech Stack của Module này

| Component | Technology |
|-----------|------------|
| MCP Server Framework | Spring Boot 3.x + Spring AI MCP |
| Build Tool | Maven |
| Testing | JUnit 5 + MCP Inspector |
| Database | PostgreSQL (Flyway migrations) |
| CI/CD | GitHub Actions |
| AI Client | Claude Code / Claude Desktop |

---

## Prerequisites Check

Trước khi bắt đầu module này, verify:

```bash
# Java 21+
java --version

# Maven 3.9+
mvn --version

# Node.js 18+ (cần cho một số MCP servers dùng npx)
node --version

# Claude Code installed
claude --version

# PostgreSQL running (cho exercises)
psql --version
```

---

## Key Resources

- [MCP Official Docs](https://modelcontextprotocol.io)
- [Spring AI MCP Integration](https://docs.spring.io/spring-ai/reference/api/mcp/)
- [MCP Server Registry](https://github.com/modelcontextprotocol/servers)
- [MCP Inspector Tool](https://github.com/modelcontextprotocol/inspector)

---

## Mindset cho Module này

MCP không phải là "thêm một library vào project". Nó là một **shift trong cách bạn thiết kế tooling**.

Thay vì nghĩ: *"Tôi cần viết script để analyze database"*

Hãy nghĩ: *"Tôi cần expose database access như một MCP tool, để Claude — và mọi AI agent trong tương lai — có thể dùng được"*

Đây là **infrastructure thinking**: build once, reuse many times. Rất quen thuộc với Java engineer đã từng build shared libraries hay internal frameworks.

---

*Module 03 of 5 — AI Engineering for Java Backend Engineers*
