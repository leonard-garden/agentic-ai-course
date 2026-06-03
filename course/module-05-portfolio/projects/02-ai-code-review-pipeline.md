# Project 02: AI Code Review Pipeline

> **Tên project:** JavaGuard — Automated AI Code Review for Java Projects
> **Tagline:** Multi-agent code review that posts findings directly to your PRs
> **Độ khó:** Intermediate-Advanced
> **Thời gian build:** 3-4 ngày
> **Stack:** Java (Spring Boot webhook receiver), Python (Claude Agent SDK orchestrator), GitHub Actions
> **Portfolio value:** ⭐⭐⭐⭐⭐ — Team-level impact

---

## Tại sao đây là portfolio piece mạnh?

**1. End-to-end agentic system**
JavaGuard không chỉ gọi Claude một lần. Nó orchestrate 4 specialized agents, tổng hợp kết quả, và post findings lên GitHub. Đây là production multi-agent system thực sự.

**2. Solves a universal problem**
Mọi development team đều cần code review. Mọi team đều thiếu reviewer bandwidth. Khi demo JavaGuard, không cần giải thích business value — nó self-evident.

**3. Production thinking là điểm sáng**
Cost guardrails, rate limiting, false positive tracking, engineer acceptance rate — đây là những thứ enterprise hiring managers muốn thấy. Không phải "tôi build tính năng", mà "tôi nghĩ về production implications."

**4. GitHub integration = enterprise standard**
Mọi công ty đều dùng GitHub (hoặc GitLab/Bitbucket với tương tự). Integrating vào workflow sẵn có là cách AI tools get adopted at scale.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        GitHub                                    │
│  Developer opens PR → GitHub Actions triggers workflow          │
└────────────────────────┬────────────────────────────────────────┘
                         │ workflow calls
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                   GitHub Actions Runner                          │
│                                                                 │
│  1. Checkout PR diff                                            │
│  2. Call JavaGuard Python orchestrator                          │
│  3. Post results as PR comment                                  │
└────────────────────────┬────────────────────────────────────────┘
                         │ subprocess
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              JavaGuard Python Orchestrator                       │
│              (Claude Agent SDK)                                 │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐    │
│  │                  Orchestrator Agent                     │    │
│  │  - Receive PR diff                                      │    │
│  │  - Split into file chunks                               │    │
│  │  - Dispatch to workers in parallel                      │    │
│  │  - Collect findings                                     │    │
│  │  - Call Synthesis agent                                 │    │
│  └────────────────┬────────────────────────────────────┬──┘    │
│                   │ parallel                            │       │
│    ┌──────────────┼──────────────┐                     │       │
│    ▼              ▼              ▼                      ▼       │
│  ┌──────┐  ┌──────────┐  ┌──────────┐         ┌──────────────┐ │
│  │Securi│  │Performan │  │ Test     │         │ Synthesis    │ │
│  │tyAudt│  │ceAnalysis│  │Coverage  │         │ Agent        │ │
│  └──────┘  └──────────┘  └──────────┘         └──────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                         │
                         ▼
              Formatted PR Comment
              posted to GitHub
```

**Flow chi tiết:**
1. Developer mở PR → GitHub Actions trigger
2. GitHub Actions checkout PR diff, gọi Python orchestrator
3. Orchestrator phân tích diff, chia thành file chunks
4. 3 worker agents chạy song song (SecurityAudit, PerformanceAnalysis, TestCoverage)
5. Synthesis agent tổng hợp findings từ 3 workers
6. Orchestrator format kết quả thành Markdown
7. GitHub Actions post comment lên PR

---

## GitHub Actions Workflow

```yaml
# .github/workflows/ai-code-review.yml
name: JavaGuard AI Code Review

on:
  pull_request:
    types: [opened, synchronize]
    paths:
      - '**/*.java'

jobs:
  ai-review:
    runs-on: ubuntu-latest
    # Skip draft PRs
    if: github.event.pull_request.draft == false

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Need full history for diff

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install JavaGuard dependencies
        run: |
          pip install anthropic>=0.40.0 PyGithub requests

      - name: Generate PR diff
        id: diff
        run: |
          git diff origin/${{ github.base_ref }}...HEAD -- '*.java' > pr_diff.txt
          echo "diff_size=$(wc -l < pr_diff.txt)" >> $GITHUB_OUTPUT

      - name: Check diff size (cost guard)
        run: |
          DIFF_LINES=${{ steps.diff.outputs.diff_size }}
          echo "PR diff: $DIFF_LINES lines"
          if [ "$DIFF_LINES" -gt 3000 ]; then
            echo "::warning::PR diff too large ($DIFF_LINES lines). Reviewing first 3000 lines only."
            head -3000 pr_diff.txt > pr_diff_truncated.txt
            mv pr_diff_truncated.txt pr_diff.txt
          fi

      - name: Run JavaGuard review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
          REPO_FULL_NAME: ${{ github.repository }}
          MAX_TOKENS_PER_REVIEW: "50000"   # Cost guard: ~$0.50 max
        run: |
          python .github/scripts/javaguard_review.py \
            --diff-file pr_diff.txt \
            --pr-number $PR_NUMBER \
            --repo $REPO_FULL_NAME

      - name: Upload review artifacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: javaguard-review-${{ github.event.pull_request.number }}
          path: review_output.json
          retention-days: 30
```

---

## Python Orchestrator

```python
# .github/scripts/javaguard_review.py
"""
JavaGuard — Multi-agent Java code review orchestrator
Uses Claude Agent SDK to run specialized review agents in parallel
"""

import anthropic
import argparse
import json
import os
import re
import sys
from pathlib import Path
from typing import Optional
import requests


def load_diff(diff_file: str) -> str:
    """Load PR diff, respect MAX_TOKENS_PER_REVIEW limit."""
    content = Path(diff_file).read_text()
    max_tokens = int(os.getenv("MAX_TOKENS_PER_REVIEW", "50000"))
    # Rough estimate: 1 token ≈ 4 chars
    max_chars = max_tokens * 4
    if len(content) > max_chars:
        content = content[:max_chars]
        print(f"Warning: Diff truncated to {max_chars} chars for cost management")
    return content


def run_security_audit(client: anthropic.Anthropic, diff: str) -> dict:
    """Agent 1: Security vulnerability analysis."""
    response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=4000,
        system="""You are a Java security expert specializing in OWASP Top 10 vulnerabilities.
Review the provided Java code diff and identify security issues ONLY.

Focus on:
- SQL injection (string concatenation in queries, missing parameterization)
- XSS vulnerabilities (unescaped user input in responses)  
- Authentication/authorization bypasses
- Sensitive data exposure (logging passwords, tokens in responses)
- Insecure deserialization
- Hard-coded credentials or API keys
- Path traversal vulnerabilities
- SSRF vulnerabilities

Format each finding as JSON:
{
  "severity": "CRITICAL|HIGH|MEDIUM|LOW",
  "file": "filename",
  "line_hint": "approximate line or method name",
  "issue": "brief description",
  "recommendation": "specific fix"
}

Return a JSON array of findings. Empty array if no issues found.
Only report actual security issues, not style or performance concerns.""",
        messages=[{
            "role": "user",
            "content": f"Review this Java PR diff for security vulnerabilities:\n\n```diff\n{diff}\n```"
        }]
    )

    try:
        text = response.content[0].text
        # Extract JSON array from response
        json_match = re.search(r'\[.*\]', text, re.DOTALL)
        findings = json.loads(json_match.group()) if json_match else []
        return {"agent": "SecurityAudit", "findings": findings, "tokens": response.usage.input_tokens}
    except Exception as e:
        return {"agent": "SecurityAudit", "findings": [], "error": str(e), "tokens": 0}


def run_performance_analysis(client: anthropic.Anthropic, diff: str) -> dict:
    """Agent 2: Performance issue detection."""
    response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=4000,
        system="""You are a Java performance expert. Review the provided Java code diff for performance issues ONLY.

Focus on:
- N+1 query problems (loops with database calls inside)
- Missing database indexes (based on query patterns)
- Inefficient collection operations (O(n²) when O(n) possible)
- Unnecessary object creation in hot paths
- Missing pagination on list endpoints
- Synchronous operations that should be async
- Missing caching for expensive operations
- String concatenation in loops (should use StringBuilder)
- Unoptimized JPQL/HQL queries

Format each finding as JSON:
{
  "severity": "HIGH|MEDIUM|LOW",
  "file": "filename", 
  "line_hint": "approximate line or method name",
  "issue": "brief description",
  "recommendation": "specific fix",
  "estimated_impact": "brief performance impact estimate"
}

Return a JSON array. Only report genuine performance problems.""",
        messages=[{
            "role": "user",
            "content": f"Review this Java PR diff for performance issues:\n\n```diff\n{diff}\n```"
        }]
    )

    try:
        text = response.content[0].text
        json_match = re.search(r'\[.*\]', text, re.DOTALL)
        findings = json.loads(json_match.group()) if json_match else []
        return {"agent": "PerformanceAnalysis", "findings": findings, "tokens": response.usage.input_tokens}
    except Exception as e:
        return {"agent": "PerformanceAnalysis", "findings": [], "error": str(e), "tokens": 0}


def run_test_coverage_check(client: anthropic.Anthropic, diff: str) -> dict:
    """Agent 3: Test coverage and quality analysis."""
    response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=4000,
        system="""You are a Java testing expert specializing in JUnit 5, Mockito, and Spring Boot Test.
Review the provided Java code diff for testing gaps ONLY.

Focus on:
- New public methods without corresponding tests
- Edge cases not covered (null inputs, empty collections, boundary values)
- Missing exception handling tests
- Tests without assertions (assertTrue(true) anti-pattern)
- Over-mocking (mocking things that should be tested)
- Missing integration tests for new endpoints
- Test isolation issues (tests that depend on order)
- Missing @Transactional rollback in test setup

Format each finding as JSON:
{
  "severity": "HIGH|MEDIUM|LOW",
  "file": "filename",
  "line_hint": "method or class name",
  "issue": "what test is missing or inadequate",
  "recommendation": "what test to add or fix"
}

Return a JSON array. Be practical — not every private method needs a test.""",
        messages=[{
            "role": "user",
            "content": f"Review this Java PR diff for test coverage gaps:\n\n```diff\n{diff}\n```"
        }]
    )

    try:
        text = response.content[0].text
        json_match = re.search(r'\[.*\]', text, re.DOTALL)
        findings = json.loads(json_match.group()) if json_match else []
        return {"agent": "TestCoverage", "findings": findings, "tokens": response.usage.input_tokens}
    except Exception as e:
        return {"agent": "TestCoverage", "findings": [], "error": str(e), "tokens": 0}


def run_synthesis(client: anthropic.Anthropic, all_findings: list[dict]) -> str:
    """Agent 4: Synthesis — create final PR comment."""
    findings_json = json.dumps(all_findings, indent=2)

    response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=3000,
        system="""You are a senior Java engineer writing a helpful, constructive code review.
Given findings from specialized review agents, create a well-structured PR comment in Markdown.

Guidelines:
- Lead with CRITICAL and HIGH severity findings
- Group findings by file
- Be specific and actionable — include code examples for fixes when helpful
- Tone: collegial, not condescending. "Consider adding X" not "You forgot X"
- End with a brief positive note if the PR looks generally good
- Use emojis sparingly for severity: 🔴 CRITICAL, 🟠 HIGH, 🟡 MEDIUM, 🟢 LOW

Structure:
## JavaGuard AI Review

### Summary
(1-2 sentence overview)

### Issues Found

#### 🔴 Critical / 🟠 High Priority
(actionable items that should block merge)

#### 🟡 Medium Priority  
(should fix but won't block)

#### 🟢 Low Priority / Suggestions
(nice to have)

### ✅ Looks Good
(brief positive note)

---
*JavaGuard AI Review — findings are suggestions, human review required*""",
        messages=[{
            "role": "user",
            "content": f"Create PR comment from these findings:\n\n{findings_json}"
        }]
    )

    return response.content[0].text


def post_pr_comment(comment: str, pr_number: int, repo: str, token: str) -> bool:
    """Post review comment to GitHub PR."""
    url = f"https://api.github.com/repos/{repo}/issues/{pr_number}/comments"
    headers = {
        "Authorization": f"token {token}",
        "Accept": "application/vnd.github.v3+json"
    }

    # Check if JavaGuard already commented (avoid duplicates on re-review)
    existing = requests.get(url, headers=headers).json()
    for comment_obj in existing:
        if "JavaGuard AI Review" in comment_obj.get("body", ""):
            # Update existing comment instead of creating new
            update_url = f"https://api.github.com/repos/{repo}/issues/comments/{comment_obj['id']}"
            resp = requests.patch(update_url, headers=headers, json={"body": comment})
            return resp.status_code == 200

    resp = requests.post(url, headers=headers, json={"body": comment})
    return resp.status_code == 201


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--diff-file", required=True)
    parser.add_argument("--pr-number", type=int, required=True)
    parser.add_argument("--repo", required=True)
    args = parser.parse_args()

    client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])
    github_token = os.environ["GITHUB_TOKEN"]

    print(f"JavaGuard: Reviewing PR #{args.pr_number} in {args.repo}")

    diff = load_diff(args.diff_file)
    if not diff.strip():
        print("No Java changes detected. Skipping review.")
        sys.exit(0)

    print("Running 3 specialized agents in parallel...")

    # Run agents (in production: use asyncio for true parallelism)
    # For simplicity here, sequential with clear structure
    results = []

    print("  → SecurityAudit agent...")
    results.append(run_security_audit(client, diff))

    print("  → PerformanceAnalysis agent...")
    results.append(run_performance_analysis(client, diff))

    print("  → TestCoverage agent...")
    results.append(run_test_coverage_check(client, diff))

    # Count findings
    all_findings = []
    total_tokens = 0
    for r in results:
        all_findings.extend(r.get("findings", []))
        total_tokens += r.get("tokens", 0)
        if "error" in r:
            print(f"  Warning: {r['agent']} error: {r['error']}")

    print(f"Total findings: {len(all_findings)} | Tokens used: {total_tokens:,}")

    # Synthesis
    print("Running Synthesis agent...")
    if all_findings:
        pr_comment = run_synthesis(client, all_findings)
    else:
        pr_comment = "## JavaGuard AI Review\n\n✅ No significant issues found in this PR.\n\n*JavaGuard AI Review — human review still recommended*"

    # Save output artifact
    output = {
        "pr_number": args.pr_number,
        "repo": args.repo,
        "findings_count": len(all_findings),
        "findings": all_findings,
        "tokens_used": total_tokens,
        "comment_preview": pr_comment[:500]
    }
    Path("review_output.json").write_text(json.dumps(output, indent=2))

    # Post to GitHub
    print("Posting review to GitHub...")
    success = post_pr_comment(pr_comment, args.pr_number, args.repo, github_token)

    if success:
        print(f"Review posted successfully. {len(all_findings)} findings reported.")
    else:
        print("Failed to post review comment.")
        sys.exit(1)


if __name__ == "__main__":
    main()
```

---

## Cost Guardrails

Cost management là một trong những điều quan trọng nhất ở production AI systems. JavaGuard có 3 tầng bảo vệ:

### Tầng 1: Diff truncation (GitHub Actions)

```yaml
# Trong workflow: nếu diff > 3000 lines, truncate
if [ "$DIFF_LINES" -gt 3000 ]; then
  head -3000 pr_diff.txt > pr_diff_truncated.txt
fi
```

### Tầng 2: Token limit per review

```python
# Trong Python: MAX_TOKENS_PER_REVIEW environment variable
max_tokens = int(os.getenv("MAX_TOKENS_PER_REVIEW", "50000"))
max_chars = max_tokens * 4
```

### Tầng 3: Monthly budget tracking

```python
# Thêm vào review script: track spending in DynamoDB/S3
def track_usage(tokens: int, pr_number: int, repo: str):
    """Track token usage for monthly budget enforcement."""
    # Approximate cost: claude-sonnet-4-5 = $3/M input tokens
    estimated_cost = (tokens / 1_000_000) * 3.0
    
    # Log to your metrics system
    print(f"METRIC pr_review_cost repo={repo} pr={pr_number} "
          f"tokens={tokens} cost_usd={estimated_cost:.4f}")
    
    # Check monthly budget (implement with your storage of choice)
    # If over budget: skip review, post warning comment
```

**Kết quả thực tế sau optimization:**
- Before: $2-3 per PR review (full file analysis)
- After: $0.30-0.50 per PR review (diff-only, truncation, model selection)
- Cost reduction: ~85%

---

## Rate Limiting và Error Handling

```python
# Thêm retry logic với exponential backoff
import time
from anthropic import RateLimitError, APIError

def run_agent_with_retry(agent_fn, client, diff, max_retries=3):
    for attempt in range(max_retries):
        try:
            return agent_fn(client, diff)
        except RateLimitError:
            wait_time = 2 ** attempt * 10  # 10s, 20s, 40s
            print(f"Rate limited. Waiting {wait_time}s...")
            time.sleep(wait_time)
        except APIError as e:
            if attempt == max_retries - 1:
                return {"agent": agent_fn.__name__, "findings": [],
                        "error": f"API error after {max_retries} retries: {e}"}
            time.sleep(5)
    return {"agent": agent_fn.__name__, "findings": [], "error": "Max retries exceeded"}
```

---

## Sample PR Comment Output

```markdown
## JavaGuard AI Review

### Summary
PR adds user authentication module với 3 new endpoints và UserService implementation.
2 security issues cần fix trước khi merge.

---

### Issues Found

#### 🔴 Critical / 🟠 High Priority

**🔴 SQL Injection — `UserRepository.java`**
Method `findByUsername()` concatenate user input trực tiếp vào SQL query:
```java
// VULNERABLE:
String sql = "SELECT * FROM users WHERE username = '" + username + "'";

// FIX:
String sql = "SELECT * FROM users WHERE username = ?";
jdbcTemplate.queryForObject(sql, UserMapper, username);
```

**🟠 Sensitive Data in Logs — `AuthService.java`**
`login()` method log raw password trước khi hashing:
```java
// REMOVE this line:
log.debug("Login attempt for user: {} with password: {}", username, password);
```

---

#### 🟡 Medium Priority

**N+1 Query — `UserController.java`**
`getAllUsersWithRoles()` gọi `roleService.getRolesForUser(user.getId())` trong loop.
Consider dùng JOIN query hoặc `@EntityGraph` trong JPA.

---

#### 🟢 Low Priority / Suggestions

**Test Coverage — `AuthService.java`**
`validateToken()` method chưa có test cho expired token case.

---

### ✅ Looks Good
Overall structure tốt. Exception handling đầy đủ. DTO pattern được implement correctly.

---
*JavaGuard AI Review · 12 findings · 42,150 tokens · ~$0.13*
*Findings are suggestions — human review required before merge*
```

---

## Metrics to Track

Sau khi deploy, track những metrics này để improve system theo thời gian:

```python
# Metrics schema (save vào database hoặc metrics system)
{
    "pr_number": 123,
    "repo": "org/repo",
    "timestamp": "2026-06-03T10:00:00Z",
    "findings_total": 8,
    "findings_by_severity": {"CRITICAL": 1, "HIGH": 2, "MEDIUM": 3, "LOW": 2},
    "findings_by_agent": {"SecurityAudit": 2, "PerformanceAnalysis": 3, "TestCoverage": 3},
    "tokens_used": 42150,
    "cost_usd": 0.13,
    "engineer_feedback": null,  # filled later: "accepted" | "rejected" | "partial"
    "findings_accepted": null,  # filled later: number
    "false_positives": null      # filled later: number
}
```

**Key KPIs:**
- **Findings per PR:** baseline sau 50 PRs → track trend
- **False positive rate:** target < 15% (ask engineers to react với 👎 on bad findings)
- **Engineer acceptance rate:** target > 60% of findings addressed
- **Cost per PR:** target < $0.50
- **Review latency:** target < 3 minutes from PR open to comment

---

## Demo Setup

```bash
# 1. Fork một open-source Spring Boot project (e.g., spring-petclinic)
git clone https://github.com/spring-projects/spring-petclinic
cd spring-petclinic

# 2. Add intentional issues vào một branch
git checkout -b demo/add-security-issues

# 3. Thêm code có vấn đề (để demo detection)
# Ví dụ: thêm SQL injection, N+1 query, missing tests

# 4. Open PR, watch JavaGuard review appear

# 5. Show comment với findings highlighted
```

**Pro tip cho demo:** Có sẵn một "before/after" PR screenshot. Hiring managers không luôn có thời gian xem live demo — screenshot rõ ràng với 2-3 findings được highlight là đủ mạnh.

---

## How to Present in Portfolio

**One-liner:** "Multi-agent code review system tích hợp GitHub Actions — SecurityAudit, PerformanceAnalysis, và TestCoverage agents chạy song song, post findings trực tiếp vào PRs."

**Questions to expect:**
- Tại sao 4 agents thay vì 1? (specialization → better findings, cheaper per agent)
- Làm thế nào handle hallucinations / false positives? (tracking + engineer feedback loop)
- Cost structure là gì? (explain 3 layers of cost control)
- Làm thế nào improve accuracy over time? (feedback loop, prompt versioning, eval dataset)
