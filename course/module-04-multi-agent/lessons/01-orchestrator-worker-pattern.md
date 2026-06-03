# Lesson 01: Orchestrator-Worker Pattern

> **Module 04 — Multi-Agent Systems** | Tuần 1 | ~90 phút

---

## Mục tiêu bài học

Sau lesson này, bạn có thể:
- Giải thích tại sao multi-agent tốt hơn single agent cho complex tasks
- Thiết kế Orchestrator-Worker architecture cho một bài toán thực tế
- Implement Java Code Review Pipeline với 4 specialized agents
- Sử dụng Claude Code Agent Teams primitives (TaskCreate, SendMessage, v.v.)

---

## 1. Tại sao Multi-Agent?

Hãy tưởng tượng bạn đang review một pull request lớn: 20 files, 800 dòng code thay đổi. Nếu bạn tự làm một mình, bạn phải:

1. Check security vulnerabilities
2. Phân tích performance bottlenecks
3. Verify test coverage
4. Format báo cáo

Làm tuần tự mất 2 giờ. Nhưng nếu bạn có 4 người chuyên sâu làm song song, xong trong 30 phút.

**Multi-agent AI hoạt động theo đúng nguyên tắc đó.**

### 4 lý do cốt lõi

**1. Parallelization — Chạy song song**

```
Single agent (sequential):
Task A → Task B → Task C → Task D = 4x latency

Multi-agent (parallel):
Task A ─┐
Task B ─┤─→ Aggregate = 1x latency + overhead
Task C ─┤
Task D ─┘
```

Anthropic's production Research feature (April 2025) cho thấy up to **90% time reduction** trên complex research queries nhờ parallelization — 3-5 concurrent agents, mỗi agent chạy 3+ tools đồng thời.

**2. Specialization — Chuyên môn hóa**

Một agent với system prompt chuyên về security sẽ tìm được nhiều lỗ hổng hơn một generalist agent. Tương tự như việc bạn thuê một penetration tester thay vì nhờ backend dev "kiêm" security review.

```
Generic agent system prompt:
"You are a helpful assistant that reviews Java code."

Specialist agent system prompt:
"You are a senior security engineer specializing in Java application security.
Focus exclusively on: SQL injection, XSS, authentication bypass, insecure
deserialization, OWASP Top 10. For each issue found, provide: severity
(CRITICAL/HIGH/MEDIUM/LOW), CWE ID, affected line numbers, and remediation
code. Do NOT comment on style or performance."
```

**3. Context Isolation — Tránh nhiễu context**

Context window của một agent là hữu hạn và dễ bị "nhiễu". Khi bạn giao cho một agent làm quá nhiều việc khác nhau, nó bắt đầu "quên" thông tin từ đầu conversation hoặc bị phân tâm bởi các instruction không liên quan.

Multi-agent giải quyết điều này bằng cách cho mỗi worker một context sạch, chỉ chứa thông tin cần thiết cho task của nó.

**4. Scale — Mở rộng dễ dàng**

Cần review nhanh hơn? Thêm worker. Cần cover nhiều ngôn ngữ hơn? Thêm specialized agent. Architecture tự nhiên horizontal scaling.

---

## 2. Analogy: Microservices Team

Bạn đã quen với microservices architecture. Multi-agent cũng có cấu trúc tương tự:

| Microservices | Multi-Agent |
|--------------|-------------|
| API Gateway | Orchestrator |
| Service A, B, C | Worker Agents |
| Message Queue | Task Queue / SendMessage |
| Service Contract (API spec) | Structured Output (JSON schema) |
| Circuit Breaker | Agent timeout + retry |
| Health Check | Agent status monitoring |

**Tech Lead = Orchestrator**

Tech Lead nhận requirements, phân rã thành subtasks, giao cho các specialist engineers, track progress, và tổng hợp kết quả. Khi một engineer bị block, Tech Lead intervene. Khi tất cả xong, Tech Lead viết summary cho stakeholders.

**Specialist Engineers = Workers**

Mỗi engineer có domain expertise riêng (backend, frontend, DevOps, security). Họ làm việc độc lập, báo cáo kết quả qua ticket/PR, không cần biết người khác đang làm gì.

---

## 3. Anthropic Production Example: Research Feature (April 2025)

Anthropic đã deploy một multi-agent research system với architecture như sau:

```
LeadResearcher (Opus 4.8) — Orchestrator
├── Nhận research query từ user
├── Phân rã thành 3-5 subtopics
├── Spawn 3-5 ResearchAgent (Sonnet 4.6) đồng thời
│   ├── Agent 1: SubTopic A
│   │   ├── web_search tool
│   │   ├── read_url tool  
│   │   └── extract_facts tool
│   ├── Agent 2: SubTopic B (parallel)
│   │   └── [same tools]
│   └── Agent 3: SubTopic C (parallel)
│       └── [same tools]
├── Thu thập kết quả từ tất cả agents
└── Synthesize thành comprehensive report
```

**Dual Parallelization:**
- **Agent-level**: 3-5 agents chạy đồng thời
- **Tool-level**: Mỗi agent chạy 3+ tools đồng thời (search + fetch + extract)

**Kết quả:** Up to 90% time reduction trên complex research queries so với single-agent sequential approach. Đây là ceiling của best-case scenario — average improvement thực tế thấp hơn nhưng vẫn rất đáng kể.

---

## 4. Pattern Structure

### Orchestrator

```
Orchestrator (Claude Opus 4.8)
│
├── Task Decomposition
│   └── Phân tích input, xác định subtasks độc lập
│
├── Worker Assignment  
│   └── Map mỗi subtask đến đúng specialized worker
│
├── Progress Tracking
│   └── Monitor TaskStatus: PENDING → IN_PROGRESS → DONE/FAILED
│
├── Result Aggregation
│   └── Collect structured outputs từ tất cả workers
│
└── Final Synthesis
    └── Tổng hợp thành output coherent cho user
```

### Workers

```
Workers (Claude Sonnet 4.6 hoặc Haiku 4.5)
│
├── Specialized System Prompt
│   └── Chi tiết về domain, output format, constraints
│
├── Restricted Tool Set
│   └── Chỉ có tools cần thiết cho task này (least privilege)
│
├── Independent Context
│   └── Không shared state với workers khác
│
└── Structured Output (JSON)
    └── Well-defined schema để orchestrator dễ aggregate
```

### Communication Flow

```
1. Orchestrator phân tích task
2. Orchestrator tạo subtasks (TaskCreate)
3. Workers nhận subtasks và bắt đầu thực thi
4. Workers gửi kết quả (SendMessage hoặc return value)
5. Orchestrator theo dõi (TaskList) và đợi completion
6. Orchestrator aggregate và synthesize
7. Orchestrator return kết quả cho user
```

---

## 5. Claude Code Agent Teams Primitives

Claude Code v2.1.32+ có experimental multi-agent primitives. Để enable:

```bash
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```

### Các primitives chính

**TaskCreate — Giao việc cho worker**

```python
# Tạo một task cho worker agent
task = await TaskCreate(
    name="security_audit",
    description="Audit Java files for OWASP Top 10 vulnerabilities",
    agent_config={
        "model": "claude-sonnet-4-6",
        "system_prompt": SECURITY_AUDITOR_PROMPT,
        "tools": ["read_file", "grep", "list_files"],
        "max_tokens": 4096
    },
    input_data={
        "files": ["src/main/java/com/example/UserService.java"],
        "context": "Spring Boot REST API"
    }
)
```

**TaskUpdate — Cập nhật trạng thái task**

```python
# Worker tự update status khi hoàn thành
await TaskUpdate(
    task_id=task.id,
    status="COMPLETED",
    output={
        "vulnerabilities": [...],
        "severity_summary": {"CRITICAL": 1, "HIGH": 2}
    }
)
```

**TaskList — Kiểm tra trạng thái tất cả tasks**

```python
# Orchestrator poll để biết workers đã xong chưa
tasks = await TaskList(filter={"status": "IN_PROGRESS"})
all_done = all(t.status == "COMPLETED" for t in tasks)
```

**SendMessage — Giao tiếp giữa agents**

```python
# Orchestrator gửi thêm context cho worker đang chạy
await SendMessage(
    target_agent_id=security_worker.id,
    content="Also check the AuthController.java file, it was added recently"
)
```

> **Lưu ý**: Các primitives này là **experimental** và có thể thay đổi. Xem [Claude Code docs](https://docs.anthropic.com/en/docs/claude-code/agent-teams) để cập nhật mới nhất.

---

## 6. AWS Reference Implementation

AWS cung cấp một reference implementation tại:
**github.com/aws-samples/sample-claude-code-agent-team**

Architecture của nó:

```
fullstack-agent (Orchestrator — Sonnet)
│
├── coding-agent (Worker)
│   ├── Viết feature code
│   ├── Tools: read_file, write_file, bash
│   └── Output: implemented files + summary
│
├── devops-agent (Worker)  
│   ├── Viết IaC, CI/CD configs
│   ├── Tools: read_file, write_file, bash
│   └── Output: deployment configs + summary
│
└── review-agent (Worker)
    ├── Review code quality, security, best practices
    ├── Tools: read_file, grep
    └── Output: review comments + approve/reject
```

Key learnings từ implementation này:
1. **Workers nên có focused, narrow system prompts** — không cần biết global context
2. **Structured JSON output là bắt buộc** — orchestrator cần parse kết quả programmatically
3. **Timeouts là critical** — mỗi worker cần hard timeout để tránh stuck
4. **Orchestrator phải handle partial failures** — một worker fail không nên crash toàn bộ pipeline

---

## 7. Java Code Review Pipeline — Full Worked Example

Đây là hệ thống chúng ta sẽ build trong module này. Architecture:

```
JavaReviewOrchestrator (Claude Opus 4.8)
│
├── SecurityAuditor (Claude Sonnet 4.6)
│   ├── Focus: OWASP Top 10, injection attacks, auth issues
│   ├── Tools: read_file, grep, list_files
│   └── Output: SecurityReport JSON
│
├── PerformanceAnalyzer (Claude Sonnet 4.6)
│   ├── Focus: N+1 queries, missing indexes, memory leaks, O(n²) algorithms
│   ├── Tools: read_file, grep
│   └── Output: PerformanceReport JSON
│
├── TestCoverageChecker (Claude Haiku 4.5)
│   ├── Focus: Untested code paths, missing edge cases, weak assertions
│   ├── Tools: read_file, list_files
│   └── Output: CoverageReport JSON
│
└── ReportSynthesizer (Claude Haiku 4.5)
    ├── Focus: Format final markdown report, prioritize findings
    ├── Tools: (none — pure text processing)
    └── Output: Final Markdown Report
```

### Subagent YAML Definitions

Đây là cách define mỗi worker agent:

```yaml
# security-auditor.yaml
name: SecurityAuditor
model: claude-sonnet-4-6-20251001
description: OWASP Top 10 security vulnerability scanner for Java code
system_prompt: |
  You are a senior application security engineer specializing in Java security.
  
  Your ONLY job is to find security vulnerabilities. Do NOT comment on:
  - Code style
  - Performance
  - Test coverage
  - Architecture (unless it's a security architecture issue)
  
  For each vulnerability found, output in this exact JSON structure:
  {
    "vulnerabilities": [
      {
        "id": "SEC-001",
        "type": "SQL_INJECTION",
        "severity": "CRITICAL",
        "cwe_id": "CWE-89",
        "file": "src/main/java/UserRepository.java",
        "line": 42,
        "code_snippet": "String query = \"SELECT * FROM users WHERE id = \" + userId;",
        "description": "Direct string concatenation in SQL query allows injection",
        "remediation": "Use parameterized queries: PreparedStatement ps = conn.prepareStatement(\"SELECT * FROM users WHERE id = ?\"); ps.setInt(1, userId);"
      }
    ],
    "summary": {
      "critical": 1,
      "high": 0,
      "medium": 2,
      "low": 3,
      "files_scanned": 5
    }
  }
  
  Security categories to check:
  - SQL/NoSQL/LDAP injection (CWE-89, CWE-943)
  - XSS in server-rendered output (CWE-79)
  - Insecure deserialization (CWE-502)
  - Broken authentication / JWT issues (CWE-287)
  - Sensitive data exposure - logging passwords, tokens (CWE-532)
  - Insecure direct object references (CWE-639)
  - Missing authorization checks (CWE-862)
  - Hardcoded credentials (CWE-798)
  - Path traversal (CWE-22)
  - SSRF (CWE-918)

tools:
  - read_file
  - grep
  - list_files
max_tokens: 4096
timeout_seconds: 120
```

```yaml
# performance-analyzer.yaml
name: PerformanceAnalyzer
model: claude-sonnet-4-6-20251001
description: Performance bottleneck analyzer for Java/Spring Boot code
system_prompt: |
  You are a senior Java performance engineer with deep expertise in JVM tuning,
  database query optimization, and distributed systems performance.
  
  Your ONLY job is to find performance issues. Do NOT comment on security or style.
  
  Output JSON with this structure:
  {
    "issues": [
      {
        "id": "PERF-001",
        "type": "N_PLUS_ONE_QUERY",
        "severity": "HIGH",
        "file": "UserService.java",
        "line": 87,
        "description": "Loading User.orders in a loop causes N+1 SELECT queries",
        "estimated_impact": "For 100 users: 101 queries instead of 2",
        "remediation": "Add @BatchSize(size=20) or use JOIN FETCH in the query"
      }
    ],
    "summary": {
      "high": 1,
      "medium": 2,
      "low": 1,
      "estimated_total_impact": "~500ms added latency per request"
    }
  }
  
  Performance areas to check:
  - N+1 query patterns (Hibernate lazy loading in loops)
  - Missing database indexes (queries without WHERE clause coverage)
  - Unbounded queries (SELECT * without LIMIT)
  - Memory allocation in hot paths (object creation in loops)
  - Synchronized blocks on contended resources
  - ThreadLocal leaks
  - String concatenation in loops (use StringBuilder)
  - Missing caching on expensive operations
  - Blocking I/O in reactive context

tools:
  - read_file
  - grep
max_tokens: 4096
timeout_seconds: 120
```

```yaml
# test-coverage-checker.yaml
name: TestCoverageChecker
model: claude-haiku-4-5-20251001
description: Test coverage and quality checker
system_prompt: |
  You are a QA engineer specializing in Java test quality.
  
  Analyze the test files and corresponding source files to identify:
  1. Untested code paths
  2. Missing edge case tests
  3. Weak assertions (assertTrue(result != null) instead of assertEquals)
  4. Tests that don't actually test behavior (testing mocks, not logic)
  5. Missing error handling tests
  
  Output JSON:
  {
    "issues": [
      {
        "id": "TEST-001",
        "type": "MISSING_TEST",
        "file": "UserService.java",
        "method": "createUser",
        "description": "No test for duplicate email scenario",
        "suggested_test": "testCreateUser_duplicateEmail_throwsDuplicateException"
      }
    ],
    "coverage_estimate": {
      "well_tested": ["UserController", "AuthService"],
      "poorly_tested": ["PaymentService", "EmailService"],
      "untested": ["ReportGenerator"]
    }
  }

tools:
  - read_file
  - list_files
max_tokens: 2048
timeout_seconds: 90
```

```yaml
# report-synthesizer.yaml
name: ReportSynthesizer
model: claude-haiku-4-5-20251001
description: Synthesizes findings from all agents into final report
system_prompt: |
  You are a technical writer creating code review reports for engineering teams.
  
  You will receive JSON outputs from SecurityAuditor, PerformanceAnalyzer,
  and TestCoverageChecker. Your job is to synthesize these into a clear,
  actionable markdown report.
  
  Report structure:
  1. Executive Summary (3-4 sentences, suitable for engineering manager)
  2. Critical Issues (must fix before merge)
  3. High Priority Issues (should fix this sprint)
  4. Medium/Low Issues (tech debt backlog)
  5. Positive Findings (what's done well)
  6. Recommended Action Items (numbered, prioritized)
  
  Tone: Direct, professional, actionable. No fluff.

tools: []
max_tokens: 3072
timeout_seconds: 60
```

### Orchestrator Prompt Template

```python
ORCHESTRATOR_SYSTEM_PROMPT = """
You are the Lead Code Reviewer orchestrating a comprehensive Java code review.

Your workflow:
1. Receive the list of files to review
2. Spawn SecurityAuditor, PerformanceAnalyzer, and TestCoverageChecker in PARALLEL
3. Wait for all three to complete
4. Pass their outputs to ReportSynthesizer
5. Return the final report

Important:
- Always run the first three workers in PARALLEL, not sequentially
- If a worker fails, log the error and continue with available results
- Include a metadata section showing which workers succeeded/failed
- The entire pipeline should complete in under 3 minutes

Output format: The final markdown report from ReportSynthesizer, plus a metadata footer showing:
- Workers: SecurityAuditor ✓, PerformanceAnalyzer ✓, TestCoverageChecker ✓
- Total time: X seconds
- Total tokens used: X
"""
```

### Python Implementation với Anthropic SDK

```python
import asyncio
from anthropic import AsyncAnthropic
from dataclasses import dataclass
from typing import Optional
import json
import time

client = AsyncAnthropic()

@dataclass
class AgentResult:
    agent_name: str
    success: bool
    output: Optional[dict]
    error: Optional[str]
    duration_seconds: float
    tokens_used: int

async def run_worker_agent(
    agent_name: str,
    model: str,
    system_prompt: str,
    task_input: str,
    tools: list,
    max_tokens: int = 4096,
    timeout: int = 120
) -> AgentResult:
    """Run một worker agent và return kết quả."""
    start = time.time()
    
    try:
        response = await asyncio.wait_for(
            client.messages.create(
                model=model,
                max_tokens=max_tokens,
                system=system_prompt,
                messages=[{"role": "user", "content": task_input}]
            ),
            timeout=timeout
        )
        
        # Parse JSON output từ agent
        content = response.content[0].text
        output = json.loads(content)
        
        return AgentResult(
            agent_name=agent_name,
            success=True,
            output=output,
            error=None,
            duration_seconds=time.time() - start,
            tokens_used=response.usage.input_tokens + response.usage.output_tokens
        )
        
    except asyncio.TimeoutError:
        return AgentResult(
            agent_name=agent_name,
            success=False,
            output=None,
            error=f"Timeout after {timeout}s",
            duration_seconds=time.time() - start,
            tokens_used=0
        )
    except json.JSONDecodeError as e:
        return AgentResult(
            agent_name=agent_name,
            success=False,
            output=None,
            error=f"Invalid JSON output: {e}",
            duration_seconds=time.time() - start,
            tokens_used=0
        )

async def java_review_pipeline(files_to_review: list[str]) -> str:
    """
    Main orchestration function.
    Chạy 3 workers song song, sau đó synthesize kết quả.
    """
    pipeline_start = time.time()
    
    # Đọc nội dung files
    file_contents = {}
    for filepath in files_to_review:
        with open(filepath, 'r') as f:
            file_contents[filepath] = f.read()
    
    task_context = f"""
Review the following Java files:

{chr(10).join(f"### {path}:{chr(10)}```java{chr(10)}{content}{chr(10)}```" for path, content in file_contents.items())}
"""
    
    # Chạy 3 workers SONG SONG
    print("Starting parallel analysis...")
    security_task, performance_task, coverage_task = await asyncio.gather(
        run_worker_agent(
            agent_name="SecurityAuditor",
            model="claude-sonnet-4-6-20251001",
            system_prompt=SECURITY_AUDITOR_PROMPT,
            task_input=task_context,
            tools=[],  # simplified — real impl passes tool definitions
            max_tokens=4096,
            timeout=120
        ),
        run_worker_agent(
            agent_name="PerformanceAnalyzer", 
            model="claude-sonnet-4-6-20251001",
            system_prompt=PERFORMANCE_ANALYZER_PROMPT,
            task_input=task_context,
            tools=[],
            max_tokens=4096,
            timeout=120
        ),
        run_worker_agent(
            agent_name="TestCoverageChecker",
            model="claude-haiku-4-5-20251001",
            system_prompt=TEST_COVERAGE_PROMPT,
            task_input=task_context,
            tools=[],
            max_tokens=2048,
            timeout=90
        )
    )
    
    parallel_duration = time.time() - pipeline_start
    print(f"Parallel analysis complete in {parallel_duration:.1f}s")
    
    # Aggregate results
    results = [security_task, performance_task, coverage_task]
    
    synthesis_input = f"""
Synthesize the following code review findings into a final report:

## SecurityAuditor Results
{"ERROR: " + security_task.error if not security_task.success else json.dumps(security_task.output, indent=2)}

## PerformanceAnalyzer Results
{"ERROR: " + performance_task.error if not performance_task.success else json.dumps(performance_task.output, indent=2)}

## TestCoverageChecker Results
{"ERROR: " + coverage_task.error if not coverage_task.success else json.dumps(coverage_task.output, indent=2)}
"""
    
    # Synthesize
    synthesizer_result = await run_worker_agent(
        agent_name="ReportSynthesizer",
        model="claude-haiku-4-5-20251001",
        system_prompt=REPORT_SYNTHESIZER_PROMPT,
        task_input=synthesis_input,
        tools=[],
        max_tokens=3072,
        timeout=60
    )
    
    total_duration = time.time() - pipeline_start
    total_tokens = sum(r.tokens_used for r in results) + synthesizer_result.tokens_used
    
    # Build final output
    worker_status = " | ".join(
        f"{r.agent_name} {'✓' if r.success else '✗'}" for r in results
    )
    
    report = synthesizer_result.output.get("report", "Report generation failed") \
        if synthesizer_result.success else f"Synthesis failed: {synthesizer_result.error}"
    
    metadata = f"""
---
*Code Review Pipeline Metadata*
- Workers: {worker_status}
- Total time: {total_duration:.1f}s (parallel phase: {parallel_duration:.1f}s)
- Total tokens: {total_tokens:,}
- Files reviewed: {len(files_to_review)}
"""
    
    return report + metadata

# Entry point
if __name__ == "__main__":
    files = [
        "src/main/java/com/example/UserService.java",
        "src/main/java/com/example/UserController.java",
        "src/test/java/com/example/UserServiceTest.java"
    ]
    
    report = asyncio.run(java_review_pipeline(files))
    print(report)
    
    # Save report
    with open("code-review-report.md", "w") as f:
        f.write(report)
```

---

## 8. Điểm quan trọng cần nhớ

### DO — Những điều nên làm

- **Dùng structured JSON output** cho tất cả workers — orchestrator cần parse kết quả
- **Timeout mỗi worker** — đừng để một worker stuck block toàn bộ pipeline
- **Handle partial failures** — nếu một worker fail, tiếp tục với kết quả còn lại
- **Log worker decisions** — khi debug, bạn cần biết từng worker đã làm gì
- **Narrow system prompts** — mỗi worker chỉ nên biết đúng phần việc của nó

### DON'T — Những điều tránh

- **Đừng share mutable state** giữa workers — mỗi worker có context sạch
- **Đừng để workers communicate trực tiếp** — orchestrator là điểm kết nối duy nhất
- **Đừng dùng Opus cho mọi worker** — cost sẽ vượt tầm kiểm soát (xem Lesson 02)
- **Đừng chạy workers tuần tự** khi chúng independent — mất toàn bộ lợi thế parallelization

---

## 9. Exercise

### Bài tập: Implement Java Code Review Pipeline

**Yêu cầu:**

1. Clone sample repository: `git clone https://github.com/spring-projects/spring-petclinic`

2. Implement pipeline theo code ở trên với đầy đủ 4 agents

3. Thêm một **LlmConfig** class để externalize các model constants:
```java
public class LlmConfig {
    public static final String ORCHESTRATOR_MODEL = "claude-opus-4-8";
    public static final String WORKER_MODEL = "claude-sonnet-4-6-20251001";
    public static final String LIGHTWEIGHT_MODEL = "claude-haiku-4-5-20251001";
}
```

4. Run pipeline trên ít nhất 3 files từ petclinic

5. Measure và report:
   - Total wall clock time
   - Thời gian nếu chạy sequential (ước tính)
   - Speedup factor
   - Total tokens used

**Bonus:** Thêm một **CodeSmellDetector** agent (Haiku) chạy song song với 3 agent kia, tìm long methods, duplicate code, magic numbers.

**Deliverable:** 
- Code pipeline hoàn chỉnh
- Sample output report từ petclinic
- Comment trong code giải thích từng design decision

---

## Tóm tắt

| Concept | Key Point |
|---------|-----------|
| Parallelization | Chạy independent tasks đồng thời, giảm latency |
| Specialization | Narrow system prompts tốt hơn generic prompts |
| Context Isolation | Mỗi worker có context sạch, không nhiễu |
| Orchestrator role | Decompose → Assign → Track → Aggregate → Synthesize |
| Worker design | JSON output, timeout, restricted tools, focused prompt |

**Bài tiếp theo:** [Lesson 02 — Model Routing Strategy](./02-model-routing-strategy.md) — Chọn đúng model cho đúng task để tối ưu cost và performance.
