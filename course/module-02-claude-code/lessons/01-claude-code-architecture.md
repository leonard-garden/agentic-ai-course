# Lesson 01: Claude Code Architecture & Internals

> **Thời lượng**: ~3 giờ đọc + thực hành  
> **Level**: Intermediate  
> **Mục tiêu**: Hiểu Claude Code từ bên trong — không chỉ biết dùng mà còn biết tại sao nó hoạt động như vậy

---

## 1. Claude Code là gì — và quan trọng hơn, là gì không phải

### Không phải chatbot có thêm tools

Nhiều người tiếp cận Claude Code như một chatbot nâng cao: gõ câu hỏi, nhận câu trả lời có code. Đây là cách hiểu sai dẫn đến việc dùng Claude Code kém hiệu quả.

Claude Code là một **agentic coding assistant**. Sự khác biệt cốt lõi:

| Chatbot | Agentic Assistant |
|---------|-------------------|
| Nhận prompt → trả lời một lần | Nhận goal → lập kế hoạch → thực thi nhiều bước |
| Không có context về filesystem | Đọc/ghi file thực trên máy bạn |
| Bạn phải copy-paste code | Claude tự edit file, chạy lệnh, verify kết quả |
| Mỗi message là independent | Maintains context qua nhiều turns |
| Bạn orchestrate | Claude tự orchestrate |

Khi bạn nói với Claude Code: *"Refactor UserService để dùng repository pattern"*, nó sẽ:
1. Đọc `UserService.java` và tất cả files liên quan
2. Tìm hiểu codebase structure (entities, existing repositories)
3. Lập kế hoạch refactor
4. Tạo `UserRepository.java`
5. Sửa `UserService.java`
6. Cập nhật tests
7. Chạy `mvn test` để verify
8. Báo cáo kết quả

Tất cả tự động, không cần bạn can thiệp từng bước.

### Định nghĩa chính xác

> **Claude Code** là một CLI tool chạy một **agent loop**: nhận high-level goal từ người dùng, tự chia nhỏ thành các actions, thực thi actions dùng built-in tools (đọc/ghi file, chạy bash commands, tìm kiếm), quan sát kết quả, điều chỉnh kế hoạch, và lặp lại cho đến khi goal hoàn thành hoặc cần human input.

---

## 2. Agent Loop — Trái tim của Claude Code

### The Loop

```
┌─────────────────────────────────────────────────────┐
│                   AGENT LOOP                        │
│                                                     │
│  User Goal                                          │
│      │                                              │
│      ▼                                              │
│  ┌─────────┐    Think    ┌──────────┐               │
│  │  Claude  │ ──────────► │  Plan    │               │
│  │  (LLM)  │             │ Actions  │               │
│  └────┬────┘             └────┬─────┘               │
│       │                       │                     │
│       │ Observe               │ Execute             │
│       │                       ▼                     │
│  ┌────┴────┐             ┌──────────┐               │
│  │ Results │ ◄────────── │  Tools   │               │
│  │ Context │             │ (Read,   │               │
│  └─────────┘             │  Write,  │               │
│       │                  │  Bash..) │               │
│       │                  └──────────┘               │
│       │                                             │
│       ▼                                             │
│   Complete? ──Yes──► Return to User                 │
│       │                                             │
│       No                                            │
│       │                                             │
│       └──────────────────────────────────┐          │
│                                          │          │
│                             (next iteration)        │
└─────────────────────────────────────────────────────┘
```

### Mỗi iteration của loop bao gồm:

**1. Think (Reasoning)**
Claude phân tích context hiện tại: goal ban đầu, những gì đã làm, kết quả vừa nhận, bước tiếp theo hợp lý nhất là gì.

**2. Act (Tool Call)**
Claude chọn một tool và gọi nó. Ví dụ: `Read("src/main/java/UserService.java")`.

**3. Observe (Tool Result)**
Claude nhận kết quả từ tool — nội dung file, output của bash command, search results, v.v.

**4. Update Context**
Kết quả được thêm vào context window. Claude "nhớ" những gì vừa xảy ra.

**5. Check Completion**
Claude tự quyết định: goal đã đạt chưa? Cần thêm steps nào không? Có cần human input không?

### Tại sao loop này quan trọng với bạn?

Hiểu loop giúp bạn:
- **Viết prompts tốt hơn**: "Fix the bug" kém hơn "Find the NullPointerException in UserService.processOrder(), understand why it happens, fix it, and add a test to prevent regression"
- **Debug khi Claude bị stuck**: Nếu loop không terminate, thường là do goal không rõ ràng hoặc tools không đủ quyền
- **Ước tính cost**: Mỗi iteration tốn tokens. Task phức tạp = nhiều iterations = nhiều tiền

---

## 3. Built-in Tools

Claude Code đi kèm bộ tools built-in. Là Java engineer, bạn cần hiểu chính xác từng tool làm gì để viết prompts và configure permissions đúng.

### File Operations

**`Read`** — Đọc nội dung file
```
Input:  file_path (string)
Output: file content as string
Notes:  Supports all text formats. Với binary files, trả về error.
        Không giới hạn file size nhưng large files tốn nhiều tokens.
```

**`Write`** — Ghi toàn bộ nội dung file (overwrite)
```
Input:  file_path, content
Output: success/error
Notes:  Tạo file nếu chưa tồn tại. Tạo parent directories nếu cần.
        NGUY HIỂM: overwrite toàn bộ file — cần permission confirm.
```

**`Edit`** — Thay thế một đoạn text cụ thể trong file
```
Input:  file_path, old_string, new_string
Output: success/error  
Notes:  Safer hơn Write vì chỉ thay đổi phần cụ thể.
        Fail nếu old_string không tìm thấy hoặc tìm thấy nhiều hơn 1 lần.
        Đây là tool Claude Code dùng nhiều nhất khi sửa code.
```

**`MultiEdit`** — Nhiều edits trong một file
```
Input:  file_path, array of {old_string, new_string}
Output: success/error
Notes:  Atomic — nếu một edit fail, tất cả fail. 
        Dùng khi cần sửa nhiều chỗ trong cùng một file.
```

### Search & Navigation

**`Bash`** — Chạy shell command
```
Input:  command (string), timeout (optional)
Output: stdout + stderr
Notes:  POWERFUL nhất — có thể làm mọi thứ.
        Dùng cho: mvn, gradle, git, grep, find, curl, v.v.
        Permission level: cần explicit approval trừ khi trong allowedTools.
```

**`Glob`** — Tìm files theo pattern
```
Input:  pattern (e.g., "**/*.java", "src/**/*Service.java")
Output: list of matching file paths
Notes:  Nhanh, không tốn nhiều tokens.
        Thường là bước đầu tiên trước khi Read files.
```

**`Grep`** — Tìm kiếm text trong files
```
Input:  pattern (regex), path (optional), include (file pattern)
Output: matching lines với file path và line number
Notes:  Dùng ripgrep nên rất nhanh.
        Ví dụ: tìm tất cả @Transactional trong project.
```

**`LS`** — List directory contents
```
Input:  path
Output: file/directory listing
Notes:  Đơn giản nhưng hữu ích để orient trong codebase mới.
```

### Web & External

**`WebSearch`** — Tìm kiếm web
```
Input:  query
Output: search results với snippets
Notes:  Dùng khi cần tìm docs, stackoverflow answers, CVE info, v.v.
        Tốn thời gian hơn local tools.
```

**`WebFetch`** — Fetch nội dung một URL
```
Input:  url
Output: page content (HTML stripped to text)
Notes:  Dùng để đọc official docs, API specs, GitHub issues.
```

### Ví dụ thực tế: Claude đọc một Spring Boot project

Khi bạn yêu cầu "Review authentication implementation", Claude thường sẽ:

```
1. Glob("**/*.java")                    → list tất cả Java files
2. Grep("@RestController", "src/")      → tìm controllers
3. Read("src/.../AuthController.java")  → đọc auth controller
4. Grep("@Service", "src/")             → tìm services
5. Read("src/.../AuthService.java")     → đọc auth service
6. Read("src/.../SecurityConfig.java")  → đọc security config
7. Grep("password|secret|token", ...)   → tìm potential leaks
8. Bash("grep -r 'TODO\\|FIXME' src/")  → tìm incomplete code
```

Mỗi bước build thêm context cho Claude để đưa ra analysis chính xác hơn.

---

## 4. Permission Model

### Ba cấp độ permission

Claude Code xử lý mỗi tool call theo một trong ba cách:

**Auto-approve** — Claude tự thực thi, không hỏi
- Read-only operations: `Read`, `Glob`, `Grep`, `LS`
- Được configure trong `allowedTools` trong settings

**Confirm** — Claude hỏi bạn trước khi thực thi
- Default cho `Bash`, `Write`, `Edit`
- Bạn thấy command/action và có thể approve hoặc deny

**Deny** — Block hoàn toàn
- Configure trong `disallowedTools`
- Claude không thể dùng tool này

### Tại sao permission model này quan trọng?

Một ví dụ thực tế đáng sợ:

```bash
# Claude được yêu cầu "clean up temp files"
# Nếu không có permission control:
Bash("rm -rf /tmp/myapp")  → OK
Bash("rm -rf ./logs")      → OK  
Bash("rm -rf .")           → ??? 
```

Permission model bảo vệ bạn khỏi những accidents như vậy. Với Java projects, bạn thường muốn:

```json
// .claude/settings.json
{
  "allowedTools": ["Read", "Grep", "Glob", "LS"],
  "permissions": {
    "allow": [
      "Bash(mvn test)",
      "Bash(mvn verify)", 
      "Bash(git diff)",
      "Bash(git log)"
    ],
    "deny": [
      "Bash(git push)",
      "Bash(rm -rf*)",
      "Write(src/main/resources/application-prod.yml)"
    ]
  }
}
```

---

## 5. Ba Built-in Subagents

Claude Code có 3 subagents được build sẵn. Đây là chi tiết **chính xác** về chúng — không phải từ docs mà từ behavior thực tế:

### Explore Agent (Haiku model, read-only)

**Mục đích**: Nhanh chóng hiểu codebase structure  
**Model**: Claude Haiku (nhanh, rẻ)  
**Tools**: Read, Grep, Glob, LS (read-only)  
**Khi nào được dùng**: Claude Code tự động spawn Explore khi cần survey codebase trước khi planning

**Ví dụ scenario**:
```
User: "Add pagination to all list endpoints"

Claude Code internally:
1. Spawn Explore agent → "Map out all REST controllers and their list methods"
2. Explore returns: "Found 12 controllers, 34 list methods, 3 already have pagination"
3. Claude Code uses this map to plan the actual implementation
```

**Tại sao Haiku?** Read-only survey không cần reasoning mạnh. Haiku nhanh và rẻ hơn 5-10x so với Sonnet.

### Plan Agent (Parent model, read-only)

**Mục đích**: Thiết kế approach trước khi implementation  
**Model**: Cùng model với parent (Sonnet/Opus)  
**Tools**: Read, Grep, Glob, LS (read-only)  
**Khi nào được dùng**: Với tasks phức tạp, trước khi viết code

**Ví dụ scenario**:
```
User: "Migrate from Hibernate to jOOQ"

Claude Code internally:
1. Spawn Plan agent → "Design migration strategy"
2. Plan agent reads all entities, repositories, queries
3. Plan agent returns: step-by-step migration plan, risks, file list
4. Claude Code presents plan to user for approval (nếu /plan mode)
   hoặc thực thi trực tiếp
```

### General Agent (Parent model, all tools)

**Mục đích**: Implementation — làm tất cả mọi thứ  
**Model**: Parent model  
**Tools**: Tất cả tools được phép  
**Khi nào được dùng**: Thực thi actual changes

### Automatic Subagent Selection

Claude Code **tự động** chọn subagent phù hợp dựa trên:
- Độ phức tạp của task
- Loại operation (read-only vs write)
- Current phase (exploration vs planning vs implementation)

Bạn không cần (và thường không nên) manually specify. Tuy nhiên, bạn có thể influence bằng cách viết rõ ràng trong prompt:
- "First explore the codebase, then..." → triggers Explore
- "Plan the approach before implementing" → triggers Plan

---

## 6. CLAUDE.md — Project Instructions File

### CLAUDE.md là gì?

`CLAUDE.md` là file markdown đặt ở root project (hoặc `~/.claude/CLAUDE.md` cho global). Claude Code đọc file này **tự động** ở đầu mỗi session và treat nó như system instructions cho project đó.

Đây là một trong những features mạnh nhất của Claude Code mà hầu hết người dùng không tận dụng đúng cách.

### Tại sao CLAUDE.md cực kỳ quan trọng?

Không có CLAUDE.md, Claude Code phải "guess" về:
- Project structure của bạn
- Coding conventions team bạn dùng
- Những gì được phép và không được phép làm
- Build commands, test commands
- Domain context (business rules, terminology)

Với CLAUDE.md tốt, Claude Code hoạt động như một **senior engineer đã onboard dự án của bạn**.

### Anatomy của một CLAUDE.md hiệu quả

```markdown
# Project: OrderManagement Service

## Context
E-commerce order management microservice. Part of larger platform with 
8 other services. Handles order lifecycle: creation → payment → fulfillment → delivery.
~150k orders/day peak load. PostgreSQL 15, Redis for caching.

## Tech Stack
- Java 21 (use virtual threads where appropriate)
- Spring Boot 3.2
- Spring Data JPA + Hibernate
- PostgreSQL 15
- Redis 7 (Lettuce client)
- Maven 3.9
- JUnit 5 + Mockito + Testcontainers

## Project Structure
src/
  main/java/com/company/orders/
    domain/          ← Domain entities and value objects (pure Java, no Spring)
    application/     ← Use cases / application services
    infrastructure/  ← Repository implementations, external adapters
    api/             ← REST controllers, DTOs, mappers
    config/          ← Spring configuration
  test/java/          ← Mirror structure of main/
    unit/             ← Fast tests, no Spring context
    integration/      ← @SpringBootTest, Testcontainers

## Coding Conventions
- IMMUTABILITY: Never mutate objects. Use Builder pattern or record types.
- ERROR HANDLING: All exceptions must be handled. Use Result<T> type in domain layer.
- NULL SAFETY: Never return null. Use Optional<T> for nullable results.
- VALIDATION: Validate at domain layer, not just controller layer.
- LOGGING: Use structured logging (key=value format). Never log PII.

## Testing Requirements
- Minimum 80% line coverage
- TDD workflow: write test first, then implementation
- Unit tests: pure Java, no Spring, fast (<100ms each)
- Integration tests: use Testcontainers for DB/Redis
- NEVER skip tests to make deadlines

## Build Commands
- Build: mvn clean install -DskipTests
- Test: mvn test
- Integration test: mvn verify -P integration-test
- Code quality: mvn spotbugs:check pmd:check

## FORBIDDEN Operations
- NEVER push directly to main branch
- NEVER log passwords, tokens, or user PII
- NEVER use System.out.println (use SLF4J)
- NEVER disable Spring Security for "testing"
- NEVER use deprecated Spring APIs
- NEVER write raw SQL (use JPQL or QueryDSL)
```

### CLAUDE.md Best Practices

**1. Be specific about conventions, not just general principles**

```markdown
# Bad
Be careful with security.

# Good  
Security rules:
- All endpoints require authentication unless annotated @Public
- Authorization checks go in the service layer, NOT controller
- SQL queries MUST use parameterized queries (PreparedStatement/JPQL parameters)
- Passwords must be hashed with BCrypt (strength=12), never MD5/SHA1
```

**2. Include domain context**

```markdown
## Business Domain
- "Order" = customer purchase, can have multiple "OrderLines"
- "Fulfillment" = warehouse picking and packing (handled by separate service)
- "Settlement" = financial reconciliation (T+1 day batch process)
- Status flow: PENDING → CONFIRMED → PROCESSING → SHIPPED → DELIVERED
- CANCELLED is terminal state — never transition out of CANCELLED
```

**3. Specify what Claude should NOT do**

```markdown
## Do NOT
- Modify files in src/generated/ (auto-generated by protobuf)  
- Change database migration files (use a new migration instead)
- Alter the OrderStatus enum values (downstream dependencies)
- Add new dependencies without checking with team first
```

**4. Reference key files**

```markdown
## Key Files
- src/main/resources/application.yml: main config (never commit secrets here)
- src/main/java/.../domain/Order.java: central domain model
- src/main/java/.../config/SecurityConfig.java: security configuration
- docs/api-contract.md: API contract with other services
```

---

## 7. Settings Files

### Hierarchy

```
~/.claude/settings.json          ← Global (tất cả projects)
    ↓ (overridden by)
.claude/settings.json            ← Project-level
    ↓ (overridden by)
.claude/settings.local.json      ← Local developer overrides (gitignore này)
```

### Key settings

```json
// ~/.claude/settings.json — global defaults
{
  "model": "claude-sonnet-4-6",
  "allowedTools": ["Read", "Grep", "Glob", "LS"],
  "theme": "dark",
  "autoCompact": true,
  "permissions": {
    "allow": [],
    "deny": []
  }
}
```

```json
// .claude/settings.json — project-level
{
  "allowedTools": ["Read", "Grep", "Glob", "LS", "Edit", "MultiEdit"],
  "permissions": {
    "allow": [
      "Bash(mvn *)",
      "Bash(git diff*)",
      "Bash(git log*)",
      "Bash(git status)"
    ],
    "deny": [
      "Bash(git push*)",
      "Bash(rm -rf*)",
      "Write(*application-prod*)"
    ]
  }
}
```

### allowedTools vs permissions

- `allowedTools`: Tool types được auto-approve (không hỏi confirm)
- `permissions.allow`: Specific bash patterns được auto-approve
- `permissions.deny`: Patterns bị block hoàn toàn

---

## 8. Hooks System

Hooks là một trong những features ít người biết nhưng **cực kỳ mạnh** của Claude Code. Hooks cho phép bạn inject logic tại các điểm cụ thể trong agent loop.

### Ba loại hooks

**PreToolUse** — Chạy TRƯỚC khi Claude thực thi một tool call
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [{
          "type": "command",
          "command": "echo 'About to modify: ' $CLAUDE_TOOL_INPUT_FILE_PATH"
        }]
      }
    ]
  }
}
```

**PostToolUse** — Chạy SAU khi tool thực thi
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [{
          "type": "command",
          "command": "mvn spotless:apply -q 2>/dev/null || true"
        }]
      }
    ]
  }
}
```

**Stop** — Chạy khi Claude Code kết thúc session
```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [{
          "type": "command",
          "command": "mvn test -q && echo 'All tests pass'"
        }]
      }
    ]
  }
}
```

### Hook examples thực tế cho Java projects

**Auto-format sau khi edit Java file:**
```json
"PostToolUse": [{
  "matcher": "Write|Edit|MultiEdit",
  "hooks": [{
    "type": "command",
    "command": "if echo \"$CLAUDE_TOOL_INPUT_FILE_PATH\" | grep -q '\\.java$'; then mvn spotless:apply -pl $(dirname $CLAUDE_TOOL_INPUT_FILE_PATH | sed 's|src/.*||') -q 2>/dev/null; fi"
  }]
}]
```

**Block writes đến production config:**
```json
"PreToolUse": [{
  "matcher": "Write|Edit",
  "hooks": [{
    "type": "command",
    "command": "if echo \"$CLAUDE_TOOL_INPUT_FILE_PATH\" | grep -q 'application-prod'; then echo 'BLOCKED: Cannot modify production config' && exit 1; fi"
  }]
}]
```

**Chạy tests liên quan sau khi sửa file:**
```json
"PostToolUse": [{
  "matcher": "Write|Edit|MultiEdit",  
  "hooks": [{
    "type": "command",
    "command": "if echo \"$CLAUDE_TOOL_INPUT_FILE_PATH\" | grep -q 'Service\\.java$'; then mvn test -Dtest=$(basename $CLAUDE_TOOL_INPUT_FILE_PATH .java)Test -q 2>&1 | tail -5; fi"
  }]
}]
```

### Hook environment variables

Hooks có access đến:
- `$CLAUDE_TOOL_NAME` — tên tool được gọi (e.g., "Edit")
- `$CLAUDE_TOOL_INPUT_FILE_PATH` — file path (cho Write/Edit)
- `$CLAUDE_TOOL_INPUT_COMMAND` — bash command (cho Bash)
- `$CLAUDE_SESSION_ID` — current session ID

---

## 9. Exercise: Thiết lập Claude Code cho Spring Boot Project

### Mục tiêu
- Install và configure Claude Code
- Viết CLAUDE.md hiệu quả cho project của bạn
- Setup permission model phù hợp
- Test 5 tasks thực tế

### Bước 1: Install

```bash
# Install Claude Code
npm install -g @anthropic-ai/claude-code

# Login
claude login

# Verify
claude --version
```

### Bước 2: Viết CLAUDE.md

Tạo file `CLAUDE.md` ở root của Spring Boot project của bạn. Dùng template sau và điền thông tin thực:

```markdown
# Project: [Tên project của bạn]

## Context
[2-3 câu mô tả project: làm gì, scale thế nào, context business]

## Tech Stack
[List thực tế: Java version, Spring Boot version, DB, cache, message queue, build tool]

## Project Structure
[Mô tả cấu trúc package chính, giải thích ý nghĩa từng layer]

## Coding Conventions
[Conventions thực tế team bạn đang dùng]

## Testing Requirements
[Coverage target, test types, test framework]

## Build Commands
- Build: [lệnh build]
- Test: [lệnh test]
- Integration test: [lệnh integration test]

## FORBIDDEN Operations
[Những gì tuyệt đối không được làm]
```

### Bước 3: Configure .claude/settings.json

```bash
mkdir -p .claude
```

```json
// .claude/settings.json
{
  "allowedTools": ["Read", "Grep", "Glob", "LS"],
  "permissions": {
    "allow": [
      "Bash(mvn test*)",
      "Bash(mvn verify*)",
      "Bash(mvn compile*)",
      "Bash(git diff*)",
      "Bash(git log*)",
      "Bash(git status)"
    ],
    "deny": [
      "Bash(git push*)",
      "Bash(git reset --hard*)",
      "Bash(rm -rf*)"
    ]
  }
}
```

### Bước 4: Test 5 Tasks

Mở terminal trong project directory và chạy `claude`. Thử từng task:

**Task 1 — Codebase exploration:**
```
Explore this Spring Boot project and give me an overview: main entities, services, 
API endpoints, database schema, and any architectural patterns you notice.
```

**Task 2 — Security review:**
```
Review the authentication and authorization implementation. 
Look for common security issues: missing auth checks, hardcoded credentials, 
SQL injection risks, sensitive data in logs.
```

**Task 3 — Test coverage:**
```
Check which services have the lowest test coverage. 
List the top 3 services that need more tests and suggest what test cases to add.
```

**Task 4 — Code quality:**
```
Find all TODO and FIXME comments in the codebase. 
Categorize them by priority and estimate effort to fix each.
```

**Task 5 — Refactoring:**
```
Find one class that violates Single Responsibility Principle. 
Explain why it violates SRP and propose a refactoring plan (don't implement yet, just plan).
```

### Checklist hoàn thành

- [ ] Claude Code installed và logged in
- [ ] CLAUDE.md viết đầy đủ với context thực của project
- [ ] .claude/settings.json configured với đúng permissions
- [ ] Cả 5 tasks chạy thành công và cho kết quả hữu ích
- [ ] Ghi chú: Claude Code hiểu project của bạn ở mức nào? Nó bỏ sót gì?

---

## Tóm tắt

| Concept | Key takeaway |
|---------|-------------|
| Agent Loop | Think → Act → Observe → lặp lại cho đến khi done |
| Built-in Tools | Read/Edit/Bash là core. Bash là mạnh nhất, cần permission |
| Permission Model | Auto-approve chỉ safe operations. Confirm cho destructive ops |
| Built-in Subagents | Explore (fast survey), Plan (design), General (implement) |
| CLAUDE.md | Project context file — đây là ROI cao nhất cho setup time |
| Settings | Hierarchy: global → project → local |
| Hooks | Automation injection points: Pre/Post/Stop |

---

*Tiếp theo: [Lesson 02 — Custom Subagents: Build Your Agent Team](02-custom-subagents.md)*
