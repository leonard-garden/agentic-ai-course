# Lesson 03: Agent SDK — Programmatic Control

> **Thời lượng**: ~3 giờ đọc + thực hành
> **Level**: Intermediate → Advanced
> **Mục tiêu**: Dùng Agent SDK để lập trình hóa và orchestrate multi-agent workflows từ code

---

## 1. Agent SDK là gì?

### Định nghĩa

**Agent SDK** (còn gọi là Claude Code SDK) là package Python/TypeScript cho phép bạn **lập trình hóa** toàn bộ Claude Code agent loop — không cần giao diện interactive CLI nữa.

Thay vì gõ lệnh trong terminal, bạn viết code điều khiển Claude như một library:

```python
import anthropic

client = anthropic.Anthropic()
result = client.beta.claude_code.run(
    prompt="Review AuthService.java for security vulnerabilities",
    tools=["Read", "Grep", "Glob"]
)
print(result.output)
```

Đây là **paradigm shift** quan trọng: từ "dùng AI tool" sang "tích hợp AI vào software system của bạn."

### Khác gì Messages API?

Bạn đã dùng Messages API ở Module 01. Đây là sự khác biệt cốt lõi:

| Feature | Messages API | Agent SDK |
|---------|-------------|-----------|
| **Abstraction level** | Low — bạn manage mọi thứ | High — SDK manage agent loop |
| **Context** | Bạn tự maintain conversation history | SDK tự maintain qua session |
| **File access** | Không có built-in | Đọc/ghi file trực tiếp trên filesystem |
| **Tool execution** | Bạn tự implement tool handlers | Tools built-in (Read, Bash, etc.) |
| **Multi-turn** | Bạn tự gửi lại history | SDK tự handle qua session_id |
| **Subagents** | Không có | Có thể spawn specialized subagents |
| **Use case** | Chatbot, Q&A, text processing | Code automation, codebase analysis |

**Khi nào dùng Messages API**: NLP tasks, chatbot, text extraction, không cần file access.

**Khi nào dùng Agent SDK**: Code review automation, CI/CD integration, codebase analysis, bất cứ thứ gì cần đọc/ghi files hoặc chạy commands.

---

## 2. Installation & Setup

### Python SDK

```bash
pip install anthropic

# Verify
python3 -c "import anthropic; print(anthropic.__version__)"
```

### TypeScript/Node SDK

```bash
npm install @anthropic-ai/sdk

# Verify
node -e "const a = require('@anthropic-ai/sdk'); console.log('ok')"
```

### API Key

```bash
export ANTHROPIC_API_KEY="sk-ant-..."

# Hoặc trong .env file
echo "ANTHROPIC_API_KEY=sk-ant-..." >> .env
```

### Verify Claude Code CLI (required by SDK)

Agent SDK gọi Claude Code CLI internally. Claude Code phải được cài và authenticated:

```bash
claude --version     # phải có output
claude whoami        # phải show account info
```

---

## 3. Core Concepts

### Sessions

Session là đơn vị context trong Agent SDK. Một session duy trì:
- Conversation history (những gì đã nói)
- File context (files đã đọc)
- Tool execution history (những gì đã chạy)

```python
import anthropic

client = anthropic.Anthropic()

# Lần chạy đầu — tạo session mới
result = client.beta.claude_code.run(
    prompt="Read UserService.java and summarize its responsibilities",
    tools=["Read", "Grep"]
)

session_id = result.session_id
print(f"Session: {session_id}")
print(result.output)

# Lần chạy tiếp — resume session, Claude "nhớ" đã đọc gì
result2 = client.beta.claude_code.run(
    prompt="Now find all methods that don't have proper null checks",
    session_id=session_id  # Tiếp tục từ context trước
)
print(result2.output)
# Claude không cần đọc lại UserService.java — đã có trong context
```

**Khi nào reset session vs resume:**

| Tình huống | Action |
|-----------|--------|
| Task mới, không liên quan đến context cũ | Reset — không truyền `session_id` |
| Follow-up question về cùng codebase | Resume — truyền `session_id` |
| Multi-step workflow (review → fix → verify) | Resume — một session cho cả workflow |
| New codebase hoặc new day | Reset |
| Session quá cũ (context stale) | Reset |

### Tools Parameter

Chỉ định tools Claude được phép dùng trong session này:

```python
# Read-only audit
result = client.beta.claude_code.run(
    prompt="...",
    tools=["Read", "Grep", "Glob", "LS"]
)

# Can write files
result = client.beta.claude_code.run(
    prompt="...",
    tools=["Read", "Grep", "Glob", "Write", "Edit"]
)

# Full access
result = client.beta.claude_code.run(
    prompt="...",
    tools=["Read", "Grep", "Glob", "Write", "Edit", "Bash"]
)
```

### System Prompt

Tương tự custom subagent — define role và behavior:

```python
result = client.beta.claude_code.run(
    prompt="Review the payment module",
    system_prompt="""You are a PCI-DSS compliance specialist reviewing Java code.
    Focus on: cardholder data protection, encryption standards, access controls.
    Format findings as: [SEVERITY] Finding — File:Line — Description — Fix.""",
    tools=["Read", "Grep", "Glob"]
)
```

---

## 4. Python Examples — Từ Cơ Bản Đến Nâng Cao

### Example 1: Basic Single-Run

```python
#!/usr/bin/env python3
"""
Basic Agent SDK usage — single run, no session management
"""
import anthropic
import os

def security_audit(project_path: str) -> str:
    """Run security audit on a Java project."""
    client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

    result = client.beta.claude_code.run(
        prompt=f"""
        You are in directory: {project_path}
        
        Perform a security audit of this Spring Boot project:
        1. Check for SQL injection vulnerabilities
        2. Check for hardcoded secrets
        3. Check Spring Security configuration
        4. List findings with severity: CRITICAL/HIGH/MEDIUM/LOW
        """,
        system_prompt="You are a senior application security engineer specializing in Java.",
        tools=["Read", "Grep", "Glob", "LS"],
        cwd=project_path  # Set working directory for file operations
    )

    return result.output


if __name__ == "__main__":
    import sys
    project = sys.argv[1] if len(sys.argv) > 1 else "."
    findings = security_audit(project)
    print(findings)
```

### Example 2: Multi-Turn Session (Review → Fix → Verify)

```python
#!/usr/bin/env python3
"""
Multi-turn workflow: Review code → Fix issues → Verify fixes
Demonstrates session continuation for coherent multi-step work.
"""
import anthropic
import os
import json
from dataclasses import dataclass
from typing import Optional

@dataclass
class WorkflowResult:
    review_output: str
    fix_output: str
    verify_output: str
    session_id: str
    issues_found: int
    issues_fixed: int

def code_review_and_fix_workflow(
    project_path: str,
    target_file: str,
    auto_fix: bool = False
) -> WorkflowResult:
    """
    Three-phase workflow:
    1. Review: find issues (read-only)
    2. Fix: apply fixes (requires write permission)
    3. Verify: confirm fixes work (read-only + bash for tests)
    """
    client = anthropic.Anthropic()

    # --- Phase 1: Review (read-only) ---
    print("Phase 1: Reviewing code...")
    review_result = client.beta.claude_code.run(
        prompt=f"""
        Review {target_file} for:
        1. Code quality issues (naming, complexity, duplication)
        2. Potential bugs (null pointers, race conditions, resource leaks)
        3. Performance issues (N+1 queries, missing indexes)
        4. Missing error handling
        
        For each issue, output in JSON format:
        {{
          "issues": [
            {{
              "severity": "HIGH|MEDIUM|LOW",
              "type": "bug|quality|performance|error-handling",
              "line": 42,
              "description": "...",
              "fix": "..."
            }}
          ]
        }}
        """,
        system_prompt="You are a senior Java engineer performing a thorough code review.",
        tools=["Read", "Grep", "Glob"],
        cwd=project_path
    )

    session_id = review_result.session_id

    # Parse issues from review output
    # In practice, you'd parse the JSON from review_result.output
    print(f"Review complete. Session: {session_id}")
    print(review_result.output[:500] + "...")

    fix_output = ""
    issues_fixed = 0

    if auto_fix:
        # --- Phase 2: Fix (write access, same session) ---
        print("\nPhase 2: Applying fixes...")
        fix_result = client.beta.claude_code.run(
            prompt="""
            Based on the issues you just found, fix all HIGH and MEDIUM severity issues.
            
            For each fix:
            1. Apply the change
            2. Confirm what was changed
            3. If a test exists for this code, check if it still passes
            
            Do NOT fix LOW severity issues — list them for manual review.
            """,
            session_id=session_id,  # Continue from review context
            tools=["Read", "Grep", "Glob", "Edit", "MultiEdit"],
            cwd=project_path
        )
        fix_output = fix_result.output
        print(f"Fixes applied.\n{fix_output[:300]}...")

        # --- Phase 3: Verify (run tests) ---
        print("\nPhase 3: Verifying fixes...")
        verify_result = client.beta.claude_code.run(
            prompt="""
            Verify the fixes you applied:
            1. Read the modified files to confirm changes look correct
            2. Run the relevant unit tests: Bash("mvn test -Dtest=*Test -q")
            3. Confirm no regressions introduced
            4. Summary: how many issues fixed, are all tests passing?
            """,
            session_id=session_id,
            tools=["Read", "Grep", "Bash"],
            cwd=project_path
        )

        return WorkflowResult(
            review_output=review_result.output,
            fix_output=fix_output,
            verify_output=verify_result.output,
            session_id=session_id,
            issues_found=0,  # Parse from review_result.output in real impl
            issues_fixed=0   # Parse from fix_result.output in real impl
        )
    else:
        return WorkflowResult(
            review_output=review_result.output,
            fix_output="(auto-fix disabled)",
            verify_output="(auto-fix disabled)",
            session_id=session_id,
            issues_found=0,
            issues_fixed=0
        )


if __name__ == "__main__":
    result = code_review_and_fix_workflow(
        project_path="/path/to/spring-boot-project",
        target_file="src/main/java/com/example/service/OrderService.java",
        auto_fix=True
    )
    print("\n=== WORKFLOW COMPLETE ===")
    print(f"Session ID: {result.session_id}")
    print(f"\n--- REVIEW ---\n{result.review_output}")
    print(f"\n--- FIXES ---\n{result.fix_output}")
    print(f"\n--- VERIFICATION ---\n{result.verify_output}")
```

### Example 3: Parallel Subagents

Subagents cho phép chạy specialized workers song song trên cùng codebase. Đây là pattern mạnh nhất của Agent SDK.

```python
#!/usr/bin/env python3
"""
Parallel subagent pattern: spawn multiple specialists simultaneously.
All subagents analyze the same codebase from different angles.
Results are aggregated into a comprehensive report.
"""
import anthropic
import asyncio
from concurrent.futures import ThreadPoolExecutor, as_completed
from dataclasses import dataclass
from typing import List

@dataclass
class AgentResult:
    agent_name: str
    output: str
    session_id: str
    error: Optional[str] = None

def run_subagent(
    client: anthropic.Anthropic,
    agent_name: str,
    system_prompt: str,
    task_prompt: str,
    tools: List[str],
    project_path: str,
    parent_session_id: Optional[str] = None
) -> AgentResult:
    """Run a single subagent. Designed to be called in a thread pool."""
    try:
        kwargs = {
            "prompt": task_prompt,
            "system_prompt": system_prompt,
            "tools": tools,
            "cwd": project_path
        }
        # Subagents can optionally share parent context
        if parent_session_id:
            kwargs["parent_session_id"] = parent_session_id

        result = client.beta.claude_code.run(**kwargs)
        return AgentResult(
            agent_name=agent_name,
            output=result.output,
            session_id=result.session_id
        )
    except Exception as e:
        return AgentResult(
            agent_name=agent_name,
            output="",
            session_id="",
            error=str(e)
        )

def comprehensive_codebase_analysis(project_path: str) -> dict:
    """
    Run 4 specialist agents in parallel, then aggregate results.
    
    Architecture:
    
    Orchestrator
        ├── security-agent    (read-only, Sonnet)
        ├── performance-agent (read-only, Sonnet)  
        ├── test-coverage-agent (read-only, Haiku)
        └── tech-debt-agent   (read-only, Haiku)
    
    All run in parallel, no dependencies between them.
    """
    client = anthropic.Anthropic()

    AGENTS = [
        {
            "name": "security",
            "system": "You are a Java security specialist. Be concise and specific.",
            "prompt": """Scan this Spring Boot project for security vulnerabilities.
                        Focus on top 5 most critical issues. Format as JSON array.""",
            "tools": ["Read", "Grep", "Glob"],
            "model_note": "use claude-sonnet-4-6"
        },
        {
            "name": "performance",
            "system": "You are a Java performance engineer. Identify the top bottlenecks.",
            "prompt": """Find the top 3 performance issues in this codebase.
                        Focus on N+1 queries, missing caching, inefficient algorithms.
                        Format as JSON array.""",
            "tools": ["Read", "Grep", "Glob"],
        },
        {
            "name": "test-coverage",
            "system": "You are a TDD advocate. Be brief and actionable.",
            "prompt": """Assess test coverage: which services/classes have no tests?
                        List top 5 most critical untested areas. Format as JSON array.""",
            "tools": ["Read", "Glob", "Grep"],
        },
        {
            "name": "tech-debt",
            "system": "You are a software quality engineer. Be specific and quantitative.",
            "prompt": """Identify technical debt: TODO comments, deprecated APIs,
                        code duplication, overly complex methods.
                        List top 5 items. Format as JSON array.""",
            "tools": ["Read", "Grep", "Glob"],
        }
    ]

    results = {}

    # Run all agents in parallel using ThreadPoolExecutor
    # (Agent SDK calls are blocking, so we use threads)
    with ThreadPoolExecutor(max_workers=4) as executor:
        futures = {
            executor.submit(
                run_subagent,
                client=client,
                agent_name=agent["name"],
                system_prompt=agent["system"],
                task_prompt=agent["prompt"],
                tools=agent["tools"],
                project_path=project_path
            ): agent["name"]
            for agent in AGENTS
        }

        for future in as_completed(futures):
            agent_name = futures[future]
            result = future.result()

            if result.error:
                print(f"[{agent_name}] ERROR: {result.error}")
                results[agent_name] = {"error": result.error}
            else:
                print(f"[{agent_name}] Complete (session: {result.session_id[:8]}...)")
                results[agent_name] = {
                    "output": result.output,
                    "session_id": result.session_id
                }

    # Aggregate results with a coordinator call
    print("\nAggregating results...")
    summary_prompt = f"""
    You have received analysis from 4 specialist agents:
    
    SECURITY FINDINGS:
    {results.get('security', {}).get('output', 'N/A')}
    
    PERFORMANCE FINDINGS:
    {results.get('performance', {}).get('output', 'N/A')}
    
    TEST COVERAGE GAPS:
    {results.get('test-coverage', {}).get('output', 'N/A')}
    
    TECHNICAL DEBT:
    {results.get('tech-debt', {}).get('output', 'N/A')}
    
    Create an executive summary:
    1. Top 3 most critical issues across all categories
    2. Recommended action plan (priority order)
    3. Estimated total effort (story points)
    """

    summary = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2000,
        messages=[{"role": "user", "content": summary_prompt}]
    )

    results["summary"] = summary.content[0].text
    return results


if __name__ == "__main__":
    import json
    import sys

    project = sys.argv[1] if len(sys.argv) > 1 else "."
    analysis = comprehensive_codebase_analysis(project)

    print("\n" + "="*60)
    print("COMPREHENSIVE CODEBASE ANALYSIS REPORT")
    print("="*60)
    print(analysis.get("summary", "No summary generated"))
```

### Example 4: CI/CD Integration Script

```python
#!/usr/bin/env python3
"""
CI/CD integration: Run as pre-merge check in GitHub Actions / Jenkins.
Returns exit code 0 (pass) or 1 (fail) based on findings.
"""
import anthropic
import sys
import json
import os
import subprocess

def get_changed_files() -> list[str]:
    """Get list of Java files changed in this PR/commit."""
    try:
        result = subprocess.run(
            ["git", "diff", "--name-only", "origin/main...HEAD"],
            capture_output=True, text=True, check=True
        )
        return [f for f in result.stdout.splitlines() if f.endswith(".java")]
    except subprocess.CalledProcessError:
        return []

def run_pr_checks(project_path: str, changed_files: list[str]) -> tuple[bool, str]:
    """
    Run automated checks on changed files.
    Returns (passed: bool, report: str)
    """
    if not changed_files:
        return True, "No Java files changed."

    client = anthropic.Anthropic()
    files_list = "\n".join(f"- {f}" for f in changed_files)

    result = client.beta.claude_code.run(
        prompt=f"""
        Review these changed Java files for PR merge readiness:
        {files_list}
        
        Check for:
        1. CRITICAL security issues (SQL injection, auth bypass, secret exposure)
        2. Obvious bugs (null pointer risks, unclosed resources, race conditions)
        3. Missing tests (is there a corresponding test file for each changed file?)
        4. Broken imports or compilation issues (look for obvious errors)
        
        OUTPUT FORMAT — respond with ONLY valid JSON:
        {{
            "passed": true/false,
            "critical_issues": [
                {{"file": "path", "line": 42, "issue": "description"}}
            ],
            "warnings": [
                {{"file": "path", "line": 10, "issue": "description"}}
            ],
            "missing_tests": ["Service1.java", "Service2.java"],
            "summary": "One sentence summary"
        }}
        
        IMPORTANT: passed=false only for CRITICAL issues. Warnings don't fail the build.
        """,
        system_prompt="You are a strict but fair code reviewer. Be precise and actionable.",
        tools=["Read", "Grep", "Glob"],
        cwd=project_path
    )

    # Parse JSON from output
    # In practice, use a JSON extraction function with fallback
    output = result.output
    try:
        # Find JSON block in output
        start = output.find('{')
        end = output.rfind('}') + 1
        if start >= 0 and end > start:
            data = json.loads(output[start:end])
            passed = data.get("passed", True)
            report = json.dumps(data, indent=2)
            return passed, report
    except json.JSONDecodeError:
        pass

    # Fallback: treat as passed if we can't parse
    return True, output


def main():
    project_path = os.environ.get("WORKSPACE", ".")
    changed_files = get_changed_files()

    print(f"Checking {len(changed_files)} changed Java files...")

    passed, report = run_pr_checks(project_path, changed_files)

    print("\n=== PR CHECK REPORT ===")
    print(report)

    if not passed:
        print("\n❌ PR check FAILED — critical issues found")
        sys.exit(1)
    else:
        print("\n✅ PR check PASSED")
        sys.exit(0)


if __name__ == "__main__":
    main()
```

---

## 5. Java Integration Patterns

Agent SDK là Python/TypeScript. Java engineer cần tích hợp như thế nào?

### Pattern 1: Subprocess Call

Java gọi Python script như một subprocess. Đơn giản nhất, phù hợp cho CI/CD pipelines.

```java
// AgentSDKRunner.java
import java.io.*;
import java.nio.file.*;
import java.util.concurrent.TimeUnit;

@Service
public class AgentSDKRunner {

    private final String pythonPath;
    private final String scriptDir;

    public AgentSDKRunner(
        @Value("${agent.python-path:python3}") String pythonPath,
        @Value("${agent.script-dir}") String scriptDir
    ) {
        this.pythonPath = pythonPath;
        this.scriptDir = scriptDir;
    }

    public AgentResult runSecurityAudit(String projectPath) throws Exception {
        return runScript("security_audit.py", projectPath);
    }

    public AgentResult runCodeReview(String projectPath, String targetFile) throws Exception {
        return runScript("code_review.py", projectPath, targetFile);
    }

    private AgentResult runScript(String scriptName, String... args) throws Exception {
        // Build command: python3 scripts/security_audit.py /path/to/project
        String[] command = new String[2 + args.length];
        command[0] = pythonPath;
        command[1] = scriptDir + "/" + scriptName;
        System.arraycopy(args, 0, command, 2, args.length);

        ProcessBuilder pb = new ProcessBuilder(command);
        pb.environment().put("ANTHROPIC_API_KEY", System.getenv("ANTHROPIC_API_KEY"));
        pb.redirectErrorStream(true);

        Process process = pb.start();

        // Read output
        StringBuilder output = new StringBuilder();
        try (BufferedReader reader = new BufferedReader(
                new InputStreamReader(process.getInputStream()))) {
            String line;
            while ((line = reader.readLine()) != null) {
                output.append(line).append("\n");
            }
        }

        boolean completed = process.waitFor(5, TimeUnit.MINUTES);
        if (!completed) {
            process.destroyForcibly();
            throw new RuntimeException("Agent script timed out after 5 minutes");
        }

        int exitCode = process.exitValue();
        return new AgentResult(exitCode == 0, output.toString(), exitCode);
    }
}

// AgentResult.java
public record AgentResult(boolean success, String output, int exitCode) {}
```

### Pattern 2: REST Wrapper Service

Wrap Agent SDK trong một microservice. Phù hợp khi nhiều Java services cần dùng agents.

```python
# agent_service.py — FastAPI wrapper around Agent SDK
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional, List
import anthropic
import os

app = FastAPI(title="Agent Service", version="1.0")
client = anthropic.Anthropic()

class AgentRequest(BaseModel):
    prompt: str
    system_prompt: Optional[str] = None
    tools: List[str] = ["Read", "Grep", "Glob"]
    project_path: str
    session_id: Optional[str] = None

class AgentResponse(BaseModel):
    output: str
    session_id: str
    success: bool

@app.post("/agent/run", response_model=AgentResponse)
async def run_agent(request: AgentRequest):
    try:
        kwargs = {
            "prompt": request.prompt,
            "tools": request.tools,
            "cwd": request.project_path
        }
        if request.system_prompt:
            kwargs["system_prompt"] = request.system_prompt
        if request.session_id:
            kwargs["session_id"] = request.session_id

        result = client.beta.claude_code.run(**kwargs)
        return AgentResponse(
            output=result.output,
            session_id=result.session_id,
            success=True
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health():
    return {"status": "ok"}

# Run: uvicorn agent_service:app --host 0.0.0.0 --port 8080
```

```java
// AgentServiceClient.java — Java client cho REST wrapper
@Service
public class AgentServiceClient {

    private final RestTemplate restTemplate;
    private final String agentServiceUrl;

    public AgentServiceClient(
        RestTemplate restTemplate,
        @Value("${agent.service.url:http://localhost:8080}") String agentServiceUrl
    ) {
        this.restTemplate = restTemplate;
        this.agentServiceUrl = agentServiceUrl;
    }

    public AgentResponse runSecurityAudit(String projectPath) {
        AgentRequest request = AgentRequest.builder()
            .prompt("Perform security audit. Find CRITICAL and HIGH severity issues. Return JSON.")
            .systemPrompt("You are a Java security specialist.")
            .tools(List.of("Read", "Grep", "Glob"))
            .projectPath(projectPath)
            .build();

        return restTemplate.postForObject(
            agentServiceUrl + "/agent/run",
            request,
            AgentResponse.class
        );
    }

    // Multi-turn: review then fix using same session
    public String reviewAndFix(String projectPath, String targetFile) {
        // Step 1: Review
        AgentRequest reviewRequest = AgentRequest.builder()
            .prompt("Review " + targetFile + " for bugs and quality issues.")
            .tools(List.of("Read", "Grep"))
            .projectPath(projectPath)
            .build();

        AgentResponse reviewResponse = restTemplate.postForObject(
            agentServiceUrl + "/agent/run", reviewRequest, AgentResponse.class
        );

        // Step 2: Fix using same session
        AgentRequest fixRequest = AgentRequest.builder()
            .prompt("Now fix all HIGH severity issues you found.")
            .tools(List.of("Read", "Edit", "MultiEdit"))
            .projectPath(projectPath)
            .sessionId(reviewResponse.getSessionId())  // Continue session!
            .build();

        AgentResponse fixResponse = restTemplate.postForObject(
            agentServiceUrl + "/agent/run", fixRequest, AgentResponse.class
        );

        return fixResponse.getOutput();
    }
}
```

### Pattern 3: Async Job Queue

Cho production workloads với nhiều requests và cần tracking:

```java
// AgentJobService.java
@Service
public class AgentJobService {

    private final AgentServiceClient agentClient;
    private final AgentJobRepository jobRepository;

    @Async("agentExecutor")
    public CompletableFuture<AgentJob> submitAudit(String projectPath, String jobId) {
        AgentJob job = jobRepository.save(AgentJob.builder()
            .id(jobId)
            .status(JobStatus.RUNNING)
            .projectPath(projectPath)
            .startedAt(Instant.now())
            .build());

        try {
            AgentResponse response = agentClient.runSecurityAudit(projectPath);
            job = job.toBuilder()
                .status(JobStatus.COMPLETED)
                .output(response.getOutput())
                .completedAt(Instant.now())
                .build();
        } catch (Exception e) {
            job = job.toBuilder()
                .status(JobStatus.FAILED)
                .error(e.getMessage())
                .completedAt(Instant.now())
                .build();
        }

        return CompletableFuture.completedFuture(jobRepository.save(job));
    }
}

// AgentJobController.java
@RestController
@RequestMapping("/api/agent-jobs")
public class AgentJobController {

    private final AgentJobService jobService;
    private final AgentJobRepository jobRepository;

    @PostMapping("/security-audit")
    public ResponseEntity<Map<String, String>> submitAudit(
        @RequestParam String projectPath
    ) {
        String jobId = UUID.randomUUID().toString();
        jobService.submitAudit(projectPath, jobId);

        return ResponseEntity.accepted().body(Map.of(
            "jobId", jobId,
            "status", "submitted",
            "pollUrl", "/api/agent-jobs/" + jobId
        ));
    }

    @GetMapping("/{jobId}")
    public ResponseEntity<AgentJob> getJob(@PathVariable String jobId) {
        return jobRepository.findById(jobId)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
}
```

---

## 6. Cost Management

Agent SDK calls tốn tiền. Chiến lược quản lý cost:

### Estimate trước khi run

```python
def estimate_cost(project_path: str) -> dict:
    """Estimate token usage before running expensive agents."""
    import os

    java_files = []
    for root, dirs, files in os.walk(project_path):
        # Skip hidden dirs, target/, .git/
        dirs[:] = [d for d in dirs if not d.startswith('.') and d != 'target']
        java_files.extend(
            os.path.join(root, f) for f in files if f.endswith('.java')
        )

    total_size = sum(os.path.getsize(f) for f in java_files)
    estimated_tokens = total_size // 4  # Rough: 4 bytes per token

    # Pricing (as of 2025)
    haiku_cost = estimated_tokens * 0.25 / 1_000_000
    sonnet_cost = estimated_tokens * 3.0 / 1_000_000
    opus_cost = estimated_tokens * 15.0 / 1_000_000

    return {
        "files": len(java_files),
        "estimated_tokens": estimated_tokens,
        "cost_haiku": f"${haiku_cost:.4f}",
        "cost_sonnet": f"${sonnet_cost:.4f}",
        "cost_opus": f"${opus_cost:.4f}",
        "recommendation": (
            "haiku" if estimated_tokens < 50_000
            else "sonnet" if estimated_tokens < 500_000
            else "consider splitting into smaller scopes"
        )
    }
```

### Cost optimization tips

1. **Scope tightly** — Đừng run agent trên toàn bộ project khi chỉ cần review 1 module
2. **Haiku cho survey** — Dùng Haiku để explore, Sonnet để analyze
3. **Cache results** — Lưu audit results, không re-run nếu files không thay đổi
4. **Targeted prompts** — "Review AuthService.java" tốt hơn "Review everything"
5. **Read-only agents trước** — Không waste tokens trên write operations cho exploratory tasks

---

## 7. Subagent Constraints

Điều quan trọng cần hiểu: **subagents không thể spawn sub-subagents**.

```
Orchestrator (bạn call SDK)
    ├── Security Agent (subagent)    ← OK
    ├── Performance Agent (subagent) ← OK
    └── Test Agent (subagent)        ← OK
            └── Sub-sub-agent?       ← KHÔNG được
```

Nếu cần deep nesting, orchestrate từ Python code của bạn, không phải từ agent prompts.

---

## 8. Exercise: Viết Code Review Workflow cho Java Project

### Mục tiêu

Viết Python script dùng Agent SDK để automate code review workflow, có thể chạy từ command line hoặc tích hợp vào CI/CD.

### Requirements

Script của bạn phải:
1. Nhận project path và (optional) list of files để review
2. Nếu không có file list → review files đã thay đổi trong git (so với main)
3. Chạy 2 review agents song song: security + code quality
4. Aggregate kết quả thành một report
5. Exit code 0 nếu không có CRITICAL issues, exit code 1 nếu có
6. Output report dưới dạng Markdown file

### Template

```python
#!/usr/bin/env python3
"""
Java Code Review Workflow
Usage: python3 review.py [project_path] [--files file1.java file2.java]
"""
import anthropic
import argparse
import subprocess
import sys
import os
from concurrent.futures import ThreadPoolExecutor, as_completed
from datetime import datetime

def parse_args():
    parser = argparse.ArgumentParser(description="Java Code Review Workflow")
    parser.add_argument("project_path", nargs="?", default=".",
                       help="Path to Java project")
    parser.add_argument("--files", nargs="+",
                       help="Specific files to review (default: git changed files)")
    parser.add_argument("--output", default="review-report.md",
                       help="Output report file")
    return parser.parse_args()

def get_changed_files(project_path: str) -> list[str]:
    # TODO: implement using git diff
    pass

def run_security_review(client, project_path, files) -> str:
    # TODO: implement using Agent SDK
    pass

def run_quality_review(client, project_path, files) -> str:
    # TODO: implement using Agent SDK
    pass

def generate_report(security: str, quality: str, files: list) -> str:
    # TODO: aggregate into markdown report
    pass

def main():
    args = parse_args()
    # TODO: implement main workflow
    pass

if __name__ == "__main__":
    main()
```

### Checklist hoàn thành

- [ ] Script chạy được từ command line
- [ ] Lấy đúng changed files từ git diff
- [ ] 2 agents chạy song song (không sequential)
- [ ] Report output là valid Markdown
- [ ] Exit code đúng: 0 nếu pass, 1 nếu có critical issues
- [ ] Test trên Spring Boot project thực của bạn
- [ ] Thử tích hợp vào git pre-push hook

---

## Tóm tắt

| Concept | Key takeaway |
|---------|-------------|
| Agent SDK vs Messages API | SDK: file access, multi-turn, tool execution. API: text processing |
| Sessions | Dùng session_id để resume context. Reset khi task mới |
| Parallel agents | ThreadPoolExecutor cho nhiều agents song song |
| Java integration | Subprocess (simple) hoặc REST wrapper (production) |
| Cost management | Scope tightly, Haiku for survey, cache results |
| Subagent limit | Không có sub-sub-agents. Orchestrate từ Python code |

---

*Tiếp theo: [Lesson 04 — Claude Code Workflows & Automation](04-claude-code-workflows.md)*
