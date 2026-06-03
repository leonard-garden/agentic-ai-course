# Lesson 04: Claude Code Workflows & Automation

> **Thời lượng**: ~3 giờ đọc + thực hành
> **Level**: Intermediate → Advanced
> **Mục tiêu**: Master Claude Code slash commands, hooks system, CLAUDE.md nâng cao, và parallel worktrees để build automation pipeline hoàn chỉnh

---

## 1. Slash Commands

Slash commands là built-in shortcuts trong Claude Code CLI. Gõ `/` trong Claude Code để xem danh sách đầy đủ.

### Các commands quan trọng nhất

**`/help`** — Hiển thị tất cả available commands và shortkeys
```
Usage: /help
```

**`/clear`** — Xóa conversation history, giữ nguyên file context
```
Usage: /clear
Khi nào dùng: Context window gần đầy, muốn bắt đầu fresh conversation
              nhưng Claude đã biết structure của codebase.
```

**`/compact`** — Compress conversation history để tiết kiệm tokens
```
Usage: /compact
Khi nào dùng: Session dài, muốn tiếp tục mà không mất context quan trọng.
              Claude sẽ summarize những gì đã làm và tiếp tục từ đó.
```

**`/model`** — Đổi model đang dùng
```
Usage: /model claude-opus-4-5
       /model claude-haiku-4-5
Khi nào dùng: Task phức tạp cần Opus, hoặc muốn dùng Haiku cho simple tasks.
```

**`/agents`** — List tất cả available agents (built-in + custom)
```
Usage: /agents
Output: Danh sách agents với name và description
Hữu ích để verify custom agents đã được load đúng.
```

**`/hooks`** — Manage hooks trong session hiện tại
```
Usage: /hooks
Output: List hooks đang active, có thể enable/disable
```

**`/status`** — Hiển thị current session info
```
Usage: /status
Output: Model, session ID, context usage, active tools
```

**`/cost`** — Hiển thị token usage và estimated cost của session
```
Usage: /cost
Output: Input tokens, output tokens, total cost estimate
Quan trọng: check cost trước khi chạy expensive tasks
```

**`/plan`** — Toggle plan mode (read-only, chỉ plan không implement)
```
Usage: /plan
Khi nào dùng: Muốn Claude thiết kế approach trước khi viết code.
              Claude sẽ nói "Here's my plan" và hỏi confirm trước khi thực thi.
```

**`/memory`** — View và manage persistent memory
```
Usage: /memory
Output: Những gì Claude "nhớ" về project của bạn
```

### Workflow tips với slash commands

```
# Typical session flow cho large refactoring task:

1. /plan         → Bật plan mode
2. [describe task] → Claude tạo plan, không implement
3. [review plan]   → Bạn review, approve hoặc adjust
4. /plan         → Tắt plan mode  
5. [implement]   → Claude execute theo plan đã approved
6. /cost         → Check cost sau khi xong
7. /compact      → Nén history nếu tiếp tục làm việc
```

---

## 2. Hooks Deep Dive

Hooks là hệ thống automation mạnh nhất của Claude Code. Chúng cho phép bạn inject logic tại các điểm cụ thể trong agent loop mà không cần sửa Claude Code source code.

### Hooks Architecture

```
Agent Loop
    │
    ├── [PreToolUse Hook] ← inject TRƯỚC tool call
    │
    ▼
Tool Execution
    │
    ├── [PostToolUse Hook] ← inject SAU tool call
    │
    ▼
Result returned to Claude
    │
   ...
    │
Claude finishes session
    │
    └── [Stop Hook] ← inject khi kết thúc
```

### Hook Configuration Format

Hooks được configure trong `settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "ToolName|AnotherTool",
        "hooks": [
          {
            "type": "command",
            "command": "your-shell-command-here"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "your-shell-command-here"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "your-shell-command-here"
          }
        ]
      }
    ]
  }
}
```

### Environment Variables trong Hooks

Hooks có access đến các env vars sau:

| Variable | Có ở | Nội dung |
|----------|------|----------|
| `CLAUDE_TOOL_NAME` | Pre + Post | Tên tool: "Edit", "Bash", "Write" |
| `CLAUDE_TOOL_INPUT` | Pre + Post | JSON string của toàn bộ tool input |
| `CLAUDE_TOOL_INPUT_FILE_PATH` | Pre + Post | File path (cho Write/Edit) |
| `CLAUDE_TOOL_INPUT_COMMAND` | Pre + Post | Command string (cho Bash) |
| `CLAUDE_TOOL_RESULT` | Post only | Output từ tool execution |
| `CLAUDE_SESSION_ID` | Tất cả | Session ID hiện tại |

### Hook Exit Codes

- Exit code **0**: Hook thành công, Claude Code tiếp tục
- Exit code **1**: Hook fail, Claude Code nhận được error message
- Exit code **2**: Hook block tool call (chỉ có ý nghĩa với PreToolUse)

---

## 3. PreToolUse Hooks — Validate và Block

PreToolUse hooks chạy TRƯỚC khi tool được thực thi. Dùng để:
- Validate parameters
- Block dangerous operations
- Log intent trước khi thực hiện
- Modify tool input (advanced)

### Example 1: Block writes đến production config

```json
// .claude/settings.json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "python3 .claude/hooks/block_prod_config.py"
          }
        ]
      }
    ]
  }
}
```

```python
# .claude/hooks/block_prod_config.py
import os
import sys
import json

file_path = os.environ.get("CLAUDE_TOOL_INPUT_FILE_PATH", "")

PROTECTED_PATTERNS = [
    "application-prod",
    "application-production",
    "application-staging",
    "secrets",
    ".env.prod",
    "k8s/prod",
]

for pattern in PROTECTED_PATTERNS:
    if pattern in file_path:
        print(f"BLOCKED: Cannot modify protected file: {file_path}", file=sys.stderr)
        print(f"Reason: File matches protected pattern '{pattern}'", file=sys.stderr)
        print(f"To modify this file, do it manually outside Claude Code.", file=sys.stderr)
        sys.exit(2)  # Exit code 2 = block this tool call

sys.exit(0)  # Allow
```

### Example 2: Log tất cả file modifications

```bash
# .claude/hooks/log_file_changes.sh
#!/bin/bash

TOOL_NAME="$CLAUDE_TOOL_NAME"
FILE_PATH="$CLAUDE_TOOL_INPUT_FILE_PATH"
SESSION="$CLAUDE_SESSION_ID"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

if [[ "$TOOL_NAME" =~ ^(Write|Edit|MultiEdit)$ ]] && [[ -n "$FILE_PATH" ]]; then
    echo "$TIMESTAMP [$SESSION] $TOOL_NAME: $FILE_PATH" >> .claude/audit.log
fi

exit 0
```

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/log_file_changes.sh"
          }
        ]
      }
    ]
  }
}
```

### Example 3: Confirm trước khi chạy dangerous bash commands

```python
# .claude/hooks/confirm_dangerous_bash.py
import os
import sys

command = os.environ.get("CLAUDE_TOOL_INPUT_COMMAND", "")

DANGEROUS_PATTERNS = [
    "rm -rf",
    "drop table",
    "DELETE FROM",
    "truncate",
    "git reset --hard",
    "git push --force",
]

for pattern in DANGEROUS_PATTERNS:
    if pattern.lower() in command.lower():
        print(f"⚠️  DANGEROUS COMMAND DETECTED", file=sys.stderr)
        print(f"Command: {command}", file=sys.stderr)
        print(f"Pattern matched: '{pattern}'", file=sys.stderr)
        print(f"This command has been BLOCKED for safety.", file=sys.stderr)
        sys.exit(2)

sys.exit(0)
```

---

## 4. PostToolUse Hooks — Auto-format và Quality Checks

PostToolUse hooks chạy SAU khi tool thực thi. Dùng để:
- Auto-format code sau khi sửa
- Chạy linter ngay sau khi edit
- Trigger tests cho file vừa sửa
- Update documentation

### Example 4: Auto-format Java files với Spotless

```bash
# .claude/hooks/auto_format_java.sh
#!/bin/bash

FILE_PATH="$CLAUDE_TOOL_INPUT_FILE_PATH"
TOOL_NAME="$CLAUDE_TOOL_NAME"

# Only trigger on Java file edits
if [[ ! "$FILE_PATH" =~ \.java$ ]]; then
    exit 0
fi

# Find the Maven project root (directory containing pom.xml)
CURRENT="$FILE_PATH"
while [[ "$CURRENT" != "/" ]]; do
    CURRENT=$(dirname "$CURRENT")
    if [[ -f "$CURRENT/pom.xml" ]]; then
        # Run Spotless on the module containing this file
        cd "$CURRENT" && mvn spotless:apply -q 2>/dev/null
        exit 0
    fi
done

exit 0
```

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/auto_format_java.sh"
          }
        ]
      }
    ]
  }
}
```

### Example 5: Chạy unit test sau khi sửa Service class

```bash
# .claude/hooks/run_related_tests.sh
#!/bin/bash

FILE_PATH="$CLAUDE_TOOL_INPUT_FILE_PATH"

# Only for Service classes
if [[ ! "$FILE_PATH" =~ Service\.java$ ]]; then
    exit 0
fi

# Extract class name, find corresponding test
CLASS_NAME=$(basename "$FILE_PATH" .java)
TEST_CLASS="${CLASS_NAME}Test"

# Find Maven module root
CURRENT="$FILE_PATH"
while [[ "$CURRENT" != "/" ]]; do
    CURRENT=$(dirname "$CURRENT")
    if [[ -f "$CURRENT/pom.xml" ]]; then
        echo "Running tests for $CLASS_NAME..."
        cd "$CURRENT" && mvn test -Dtest="$TEST_CLASS" -q 2>&1 | tail -10
        TEST_EXIT=$?
        if [[ $TEST_EXIT -ne 0 ]]; then
            echo "⚠️  Tests failed for $CLASS_NAME after edit!"
        else
            echo "✅ Tests pass for $CLASS_NAME"
        fi
        exit 0
    fi
done

exit 0
```

### Example 6: CheckStyle sau khi edit

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'if echo \"$CLAUDE_TOOL_INPUT_FILE_PATH\" | grep -q \"\\.java$\"; then cd $(dirname \"$CLAUDE_TOOL_INPUT_FILE_PATH\" | sed \"s|/src/.*$||g\") && mvn checkstyle:check -q 2>&1 | grep -E \"ERROR|WARNING\" | head -5; fi'"
          }
        ]
      }
    ]
  }
}
```

---

## 5. Stop Hooks — Final Verification và Notifications

Stop hooks chạy khi Claude Code kết thúc session (user gõ Ctrl+C hoặc task complete). Dùng để:
- Run full test suite để verify không có regression
- Auto-commit khi task complete
- Send notification (Slack, email, etc.)
- Generate report

### Example 7: Chạy full test suite khi kết thúc

```bash
# .claude/hooks/verify_on_stop.sh
#!/bin/bash

echo ""
echo "=== Running verification suite ==="

# Check if any Java files were modified in this session
if [[ -f ".claude/audit.log" ]]; then
    SESSION_ID="$CLAUDE_SESSION_ID"
    MODIFIED=$(grep "$SESSION_ID" .claude/audit.log | grep -E "Write|Edit" | wc -l)

    if [[ "$MODIFIED" -gt 0 ]]; then
        echo "Files modified this session: $MODIFIED"
        echo "Running mvn verify..."
        mvn verify -q 2>&1 | tail -20

        if [[ $? -eq 0 ]]; then
            echo "✅ All tests pass"
        else
            echo "❌ Tests FAILED — review before committing"
        fi
    else
        echo "No files modified, skipping tests."
    fi
fi
```

### Example 8: Slack notification khi task complete

```python
# .claude/hooks/notify_slack.py
import os
import sys
import json
import urllib.request
import urllib.parse

SLACK_WEBHOOK = os.environ.get("SLACK_WEBHOOK_URL")
if not SLACK_WEBHOOK:
    sys.exit(0)  # Skip if no webhook configured

session_id = os.environ.get("CLAUDE_SESSION_ID", "unknown")

# Read audit log to summarize what was done
audit_log = ".claude/audit.log"
files_modified = []
if os.path.exists(audit_log):
    with open(audit_log) as f:
        lines = [l for l in f if session_id in l]
        files_modified = list(set(
            l.split(": ")[-1].strip() for l in lines
        ))

message = {
    "text": f"Claude Code session completed",
    "blocks": [
        {
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": f"*Claude Code task finished* (session: `{session_id[:8]}...`)"
            }
        },
        {
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": f"Files modified: {len(files_modified)}\n" +
                        "\n".join(f"• `{f}`" for f in files_modified[:5])
            }
        }
    ]
}

data = json.dumps(message).encode("utf-8")
req = urllib.request.Request(SLACK_WEBHOOK, data=data,
                              headers={"Content-Type": "application/json"})
try:
    urllib.request.urlopen(req, timeout=5)
except Exception:
    pass  # Don't fail session for notification errors

sys.exit(0)
```

### Example 9: Auto-commit sau khi task complete

```bash
# .claude/hooks/auto_commit.sh
#!/bin/bash
# Auto-commit nếu có file changes và tất cả tests pass

# Check if there are staged/unstaged changes
if ! git diff --quiet || ! git diff --cached --quiet; then
    echo "=== Auto-commit: Running tests first ==="
    mvn test -q 2>&1 | tail -5

    if [[ $? -eq 0 ]]; then
        git add -A
        # Use Claude session ID as part of commit message
        git commit -m "chore: claude code session ${CLAUDE_SESSION_ID:0:8} changes

Auto-committed by Claude Code stop hook.
All tests pass."
        echo "✅ Changes committed"
    else
        echo "❌ Tests failed — not committing. Review changes manually."
        exit 1
    fi
fi

exit 0
```

---

## 6. Complete Hooks Configuration cho Spring Boot Project

Đây là cấu hình hooks hoàn chỉnh cho một Spring Boot team:

```json
// .claude/settings.json
{
  "model": "claude-sonnet-4-6",
  "allowedTools": ["Read", "Grep", "Glob", "LS"],
  "permissions": {
    "allow": [
      "Bash(mvn test*)",
      "Bash(mvn verify*)",
      "Bash(mvn compile*)",
      "Bash(mvn spotless:apply*)",
      "Bash(mvn checkstyle:check*)",
      "Bash(git diff*)",
      "Bash(git log*)",
      "Bash(git status)",
      "Bash(git add*)",
      "Bash(git commit*)"
    ],
    "deny": [
      "Bash(git push*)",
      "Bash(git reset --hard*)",
      "Bash(rm -rf*)",
      "Bash(mvn deploy*)",
      "Write(*application-prod*)",
      "Write(*application-staging*)",
      "Write(*.env.prod)"
    ]
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "python3 .claude/hooks/block_prod_config.py && bash .claude/hooks/log_file_changes.sh"
          }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 .claude/hooks/confirm_dangerous_bash.py"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/auto_format_java.sh && bash .claude/hooks/run_related_tests.sh"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/verify_on_stop.sh"
          }
        ]
      }
    ]
  }
}
```

---

## 7. CLAUDE.md Best Practices cho Java Projects — Nâng Cao

Ở Lesson 01, bạn đã học structure cơ bản của CLAUDE.md. Đây là những techniques nâng cao cho Java projects.

### Technique 1: Layer-specific Instructions

```markdown
## Architecture Layers — Rules per Layer

### Domain Layer (`domain/`)
- ZERO Spring annotations (no @Component, @Service, @Repository)
- ZERO infrastructure imports (no JPA, no Redis, no HTTP clients)
- All classes are immutable (Builder pattern or records)
- Business logic lives HERE, not in services or controllers
- Throw domain exceptions (OrderException, not RuntimeException)

### Application Layer (`application/`)
- Orchestrates domain objects and infrastructure
- One use case = one method (createOrder, cancelOrder, not processOrder)
- @Transactional goes here, not in domain
- Input validation using javax.validation annotations
- Never expose domain objects directly — map to DTOs

### Infrastructure Layer (`infrastructure/`)
- Implements interfaces defined in domain/application
- All @Repository, @Component for adapters go here
- Testcontainers for integration tests (never mock the DB)
- Each adapter in its own package (jpa/, redis/, kafka/, http/)

### API Layer (`api/`)
- Request/Response DTOs — never expose domain objects
- Controllers are thin — only validation + delegation to application services
- Consistent error format: {"error": "message", "code": "ERROR_CODE"}
- API versioning via URL prefix: /api/v1/
```

### Technique 2: Domain-specific Business Rules

```markdown
## Order Domain Rules — CRITICAL

These rules are invariants. Breaking them causes bugs that are hard to detect.

### Order Status Transitions
Valid transitions ONLY:
- PENDING → CONFIRMED (when payment authorized)
- PENDING → CANCELLED (customer cancellation, within 1 hour)
- CONFIRMED → PROCESSING (warehouse picks up)
- CONFIRMED → CANCELLED (admin override, before processing)
- PROCESSING → SHIPPED (tracking number assigned)
- SHIPPED → DELIVERED (delivery confirmed)

NEVER transition backwards. NEVER skip states.
CANCELLED and DELIVERED are terminal — no transitions allowed.

### Financial Rules
- Order total = sum(line.quantity * line.unitPrice) * (1 - discount%) + shippingFee
- Discounts are applied BEFORE tax calculation
- Refunds CANNOT exceed original payment amount
- Currency is always stored as Long (cents), never Double or BigDecimal for storage

### Data Integrity
- OrderLine.quantity must be > 0
- Order must have at least 1 OrderLine
- CustomerEmail must be validated (RFC 5322 basic validation)
- ShippingAddress cannot be null for CONFIRMED orders
```

### Technique 3: Explicit Anti-patterns

```markdown
## Anti-patterns — DO NOT DO THESE

### The ones we've seen cause real bugs:

❌ **Never use Optional.get() without isPresent() check**
```java
// WRONG
User user = userRepository.findById(id).get(); // throws if not found

// CORRECT  
User user = userRepository.findById(id)
    .orElseThrow(() -> new UserNotFoundException(id));
```

❌ **Never catch Exception broadly**
```java
// WRONG
try { ... } catch (Exception e) { log.error("error", e); }

// CORRECT
try { ... } catch (OrderNotFoundException e) { 
    throw e; // let it propagate to error handler
} catch (DataAccessException e) {
    throw new OrderPersistenceException("Failed to save order", e);
}
```

❌ **Never put business logic in @Controller**
```java
// WRONG
@PostMapping("/orders/{id}/cancel")
public ResponseEntity<?> cancel(@PathVariable Long id) {
    Order order = orderRepo.findById(id).orElseThrow();
    if (order.getStatus() != OrderStatus.PENDING) throw new BadRequestException();
    order.setStatus(OrderStatus.CANCELLED); // business logic in controller!
    orderRepo.save(order);
    return ResponseEntity.ok().build();
}

// CORRECT — controller delegates to service
@PostMapping("/orders/{id}/cancel")
public ResponseEntity<?> cancel(@PathVariable Long id) {
    orderService.cancelOrder(id);
    return ResponseEntity.ok().build();
}
```
```

### Technique 4: Test Conventions

```markdown
## Test Conventions

### Naming
Method name format: `should_[expectedBehavior]_when_[condition]()`

Examples:
- `should_throwOrderNotFoundException_when_orderIdDoesNotExist()`
- `should_returnEmptyList_when_noOrdersExistForCustomer()`
- `should_calculateCorrectTotal_when_discountApplied()`

### Test Data Builders
We have ObjectMother classes for test data. USE THEM:
```java
// WRONG — raw constructors
Order order = new Order();
order.setId(1L);
order.setCustomerEmail("test@example.com");
// ... 20 more setters

// CORRECT — use OrderMother
Order order = OrderMother.aConfirmedOrder()
    .withCustomerEmail("specific@test.com")
    .build();
```
ObjectMother classes: `src/test/java/com/example/testdata/`

### What needs Testcontainers vs Mockito
- Database operations → Testcontainers (PostgreSQL container)
- External HTTP APIs → Mockito or WireMock
- Redis → Testcontainers (Redis container)
- Kafka → Testcontainers (Kafka container)
- Pure business logic → no Spring, no mocks, just Java
```

---

## 8. Parallel Worktrees

Worktrees là Git feature cho phép bạn checkout nhiều branches đồng thời trong các thư mục riêng biệt. Kết hợp với Claude Code, bạn có thể chạy **nhiều Claude Code instances song song trên các branches khác nhau**.

### Setup Parallel Worktrees

```bash
# Giả sử bạn đang ở main project: /projects/order-service

# Tạo worktree cho feature branch
git worktree add /projects/order-service-feature-payment feature/payment-refactor

# Tạo worktree cho hotfix
git worktree add /projects/order-service-hotfix hotfix/null-pointer-fix

# List worktrees
git worktree list
# /projects/order-service          abc1234 [main]
# /projects/order-service-feature  def5678 [feature/payment-refactor]
# /projects/order-service-hotfix   ghi9012 [hotfix/null-pointer-fix]
```

### Chạy Claude Code trên multiple worktrees

```bash
# Terminal 1 — main branch
cd /projects/order-service
claude "Review PR #234 — payment service refactor"

# Terminal 2 — feature branch (đồng thời)
cd /projects/order-service-feature-payment
claude "Implement the payment refactor plan"

# Terminal 3 — hotfix (đồng thời)
cd /projects/order-service-hotfix
claude "Fix the NullPointerException in OrderProcessor line 142"
```

Ba Claude Code instances chạy **độc lập**, mỗi cái trên branch riêng, không conflict.

### Worktree workflow cho common scenarios

**Scenario 1: Review PR trong khi tiếp tục develop**

```bash
# Đang làm việc trên feature branch
cd /projects/order-service-feature

# Có urgent PR cần review
git worktree add /tmp/pr-review-456 pr/feature-456
cd /tmp/pr-review-456
claude "@java-security-auditor Review this PR for security issues"

# Trở lại làm việc, PR review chạy background
cd /projects/order-service-feature
# tiếp tục develop...
```

**Scenario 2: Parallel implementation + testing**

```bash
# Worktree 1: Implement feature
cd /projects/order-service-impl
claude "Implement OrderRefundService with all business rules"

# Worktree 2 (đồng thời): Write tests cho feature đó
cd /projects/order-service-tests
claude "@java-test-writer Write comprehensive tests for OrderRefundService"
```

**Scenario 3: Investigate production issue song song**

```bash
# Worktree 1: Debug trong version hiện tại
git worktree add /tmp/debug-prod v2.3.1
cd /tmp/debug-prod
claude "@spring-boot-debugger Debug the memory leak reported in v2.3.1"

# Worktree 2: Prepare fix trên main
cd /projects/order-service
claude "Based on the memory leak pattern, find and fix it in current codebase"
```

### Cleanup worktrees

```bash
# Xóa worktree sau khi xong
git worktree remove /projects/order-service-hotfix

# Force remove nếu có uncommitted changes
git worktree remove --force /tmp/pr-review-456

# List và verify
git worktree list
git worktree prune  # Clean up stale references
```

---

## 9. Full Automation Setup: CI/CD Pipeline với Claude Code

Kết hợp tất cả những gì đã học để setup một complete automation pipeline.

### Architecture

```
Developer pushes code
        │
        ▼
Pre-push hook (local)
├── Claude Code: Quick security scan (30 sec)
├── If CRITICAL found → block push
└── If clean → allow push
        │
        ▼
GitHub Actions / Jenkins
├── Agent SDK: Full security audit
├── Agent SDK: Code quality review  
├── Agent SDK: Test coverage check
└── Aggregate report → PR comment
        │
        ▼
Nightly Job
└── Agent SDK: Deep architecture review → Slack report
```

### Step 1: Git Pre-push Hook

```bash
#!/bin/bash
# .git/hooks/pre-push
# Install: cp .claude/hooks/pre-push .git/hooks/pre-push && chmod +x .git/hooks/pre-push

echo "Running Claude Code pre-push security check..."

# Get changed files in this push
CHANGED=$(git diff --name-only origin/main...HEAD | grep '\.java$')

if [[ -z "$CHANGED" ]]; then
    echo "No Java files changed, skipping check."
    exit 0
fi

# Run quick security scan using Agent SDK
python3 .claude/hooks/quick_security_check.py "$PWD" "$CHANGED"
EXIT_CODE=$?

if [[ $EXIT_CODE -ne 0 ]]; then
    echo ""
    echo "❌ Pre-push check FAILED — critical security issues found"
    echo "Fix the issues above before pushing."
    echo "To bypass (emergency only): git push --no-verify"
    exit 1
fi

echo "✅ Pre-push check passed"
exit 0
```

```python
# .claude/hooks/quick_security_check.py
#!/usr/bin/env python3
import anthropic
import sys
import os

def quick_scan(project_path: str, changed_files_str: str) -> bool:
    """Quick 30-second security scan. Returns True if safe to push."""
    client = anthropic.Anthropic()
    files = changed_files_str.strip().split('\n')

    if not files or files == ['']:
        return True

    files_list = '\n'.join(f'- {f}' for f in files[:10])  # Max 10 files

    result = client.beta.claude_code.run(
        prompt=f"""QUICK SCAN — you have 30 seconds.
        
        Scan these changed files for CRITICAL security issues only:
        {files_list}
        
        CRITICAL issues: SQL injection, hardcoded secrets, auth bypass, RCE.
        IGNORE: style, warnings, suggestions.
        
        Respond with ONLY:
        SAFE
        OR
        CRITICAL: <file>:<line>: <one-line description>
        """,
        tools=["Read", "Grep"],
        cwd=project_path
    )

    output = result.output.strip()
    if output.startswith("SAFE"):
        return True
    else:
        print(output, file=sys.stderr)
        return False

if __name__ == "__main__":
    project_path = sys.argv[1]
    changed_files = sys.argv[2] if len(sys.argv) > 2 else ""
    safe = quick_scan(project_path, changed_files)
    sys.exit(0 if safe else 1)
```

### Step 2: GitHub Actions Workflow

```yaml
# .github/workflows/ai-review.yml
name: AI Code Review

on:
  pull_request:
    branches: [main, develop]
    paths: ['**.java', 'pom.xml']

jobs:
  ai-review:
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for git diff

      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Setup Node (for Claude Code CLI)
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Install Python dependencies
        run: pip install anthropic

      - name: Run AI Review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          python3 .claude/ci/pr_review.py \
            --project-path "$GITHUB_WORKSPACE" \
            --base-branch origin/main \
            --output review-report.json

      - name: Post Review Comment
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const report = JSON.parse(fs.readFileSync('review-report.json', 'utf8'));
            
            let body = `## AI Code Review\n\n`;
            body += `**Status**: ${report.passed ? '✅ Passed' : '❌ Failed'}\n\n`;
            
            if (report.critical_issues?.length > 0) {
              body += `### Critical Issues\n`;
              report.critical_issues.forEach(issue => {
                body += `- **${issue.file}:${issue.line}** — ${issue.issue}\n`;
              });
            }
            
            if (report.warnings?.length > 0) {
              body += `\n### Warnings\n`;
              report.warnings.slice(0, 5).forEach(w => {
                body += `- ${w.file}: ${w.issue}\n`;
              });
            }
            
            body += `\n*Generated by Claude Code AI Review*`;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: body
            });

      - name: Fail if critical issues
        run: |
          PASSED=$(python3 -c "import json; d=json.load(open('review-report.json')); print(str(d['passed']).lower())")
          if [[ "$PASSED" == "false" ]]; then
            echo "❌ Critical issues found — failing build"
            exit 1
          fi
```

---

## 10. Exercise: Setup Hooks cho Spring Boot Project

### Mục tiêu

Setup complete hooks configuration cho dự án Spring Boot của bạn và automate testing pipeline.

### Bước 1: Tạo hooks directory structure

```bash
mkdir -p .claude/hooks
touch .claude/hooks/block_prod_config.py
touch .claude/hooks/log_file_changes.sh
touch .claude/hooks/auto_format_java.sh
touch .claude/hooks/run_related_tests.sh
touch .claude/hooks/verify_on_stop.sh
chmod +x .claude/hooks/*.sh
```

### Bước 2: Implement hooks

Copy và adapt các hooks examples từ trên vào project của bạn. Adjust:
- Protected file patterns phù hợp với project của bạn
- Module structure cho Maven multi-module (nếu có)
- Test naming conventions

### Bước 3: Configure settings.json

```json
// .claude/settings.json — điều chỉnh theo project của bạn
{
  "allowedTools": ["Read", "Grep", "Glob", "LS"],
  "permissions": {
    "allow": ["Bash(mvn *)","Bash(git diff*)","Bash(git log*)","Bash(git status)"],
    "deny": ["Bash(git push*)","Bash(rm -rf*)"]
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [{"type": "command","command": "python3 .claude/hooks/block_prod_config.py"}]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [{"type": "command","command": "bash .claude/hooks/auto_format_java.sh"}]
      }
    ],
    "Stop": [
      {
        "hooks": [{"type": "command","command": "bash .claude/hooks/verify_on_stop.sh"}]
      }
    ]
  }
}
```

### Bước 4: Test hooks

```bash
# Test PreToolUse hook — thử edit production config
claude "Edit application-prod.yml to add a new property"
# Kỳ vọng: bị block bởi hook

# Test PostToolUse hook — edit một Service file
claude "Add a log statement to the beginning of UserService.findById()"
# Kỳ vọng: auto-format chạy sau khi edit

# Test Stop hook — làm một thay đổi nhỏ và kết thúc session
claude "Add a comment to UserService"
# Ctrl+C hoặc /exit
# Kỳ vọng: verify_on_stop chạy và báo cáo test results
```

### Bước 5: Setup parallel worktree workflow

```bash
# Tạo worktree cho một feature đang pending
BRANCH_NAME="feature/$(date +%Y%m%d)-ai-experiment"
git checkout -b $BRANCH_NAME
git worktree add /tmp/ai-experiment $BRANCH_NAME

# Test: chạy Claude Code trên worktree
cd /tmp/ai-experiment
claude "Explore this codebase and suggest 3 improvements"
```

### Checklist hoàn thành

- [ ] .claude/hooks/ directory tạo với tất cả hook scripts
- [ ] .claude/settings.json có đầy đủ permissions + hooks config
- [ ] PreToolUse hook block production config files
- [ ] PostToolUse hook auto-format Java files sau edit
- [ ] Stop hook chạy test suite khi kết thúc session
- [ ] Test tất cả 3 loại hooks và xác nhận hoạt động
- [ ] Tạo ít nhất 1 worktree và test parallel workflow
- [ ] (Bonus) Setup pre-push git hook với quick security scan

---

## Tổng kết Module 02

Bạn đã học xong toàn bộ Module 02. Nhìn lại những gì đã học:

| Lesson | Key skills |
|--------|-----------|
| 01 — Architecture | Agent loop, tools, permissions, CLAUDE.md, settings |
| 02 — Subagents | 5 specialized agents, description quality, least privilege |
| 03 — Agent SDK | Sessions, multi-turn, parallel agents, Java integration |
| 04 — Workflows | Slash commands, 3 hook types, CLAUDE.md advanced, worktrees |

### Bước tiếp theo

1. **Áp dụng ngay**: Setup Claude Code trên dự án Spring Boot chính của bạn
2. **Build agent team**: Deploy 3-5 custom agents phù hợp với domain của bạn
3. **Automate một workflow**: Chọn một pain point (code review, test writing, security scan) và automate hoàn toàn
4. **Module 03**: MCP (Model Context Protocol) — connect Claude với databases, APIs, và internal tools của công ty bạn

---

*Kết thúc Module 02 — Tiến sang [Module 03: MCP](../../module-03-mcp/README.md)*
