# Lesson 02: Model Routing — Cost vs Capability

> **Module 04 — Multi-Agent Systems** | Tuần 2 | ~90 phút

---

## Mục tiêu bài học

Sau lesson này, bạn có thể:
- Giải thích trade-off cost/capability giữa Haiku, Sonnet, và Opus
- Thiết kế routing logic để tự động chọn model phù hợp
- Tính toán monthly cost savings khi áp dụng model routing
- Implement `ModelRouter` utility class trong Java
- Sử dụng prompt caching để giảm thêm 80-90% input token cost

---

## 1. Vấn đề: Cost Runaway trong Multi-Agent Systems

Trong Lesson 01, chúng ta build Java Code Review Pipeline với 4 agents. Giả sử pipeline này được chạy **50 lần/ngày** (mỗi PR một lần):

**Nếu tất cả agents dùng Opus:**
```
SecurityAuditor    (Opus): 50 calls × 8,000 tokens × $0.015/1K = $6.00/ngày
PerformanceAnalyzer(Opus): 50 calls × 8,000 tokens × $0.015/1K = $6.00/ngày
TestCoverageChecker(Opus): 50 calls × 4,000 tokens × $0.015/1K = $3.00/ngày
ReportSynthesizer  (Opus): 50 calls × 3,000 tokens × $0.015/1K = $2.25/ngày
─────────────────────────────────────────────────────────────────────────────
Total: ~$17.25/ngày × 30 ngày = $517.50/tháng
```

**Với model routing phù hợp:**
```
SecurityAuditor    (Sonnet): 50 calls × 8,000 tokens × $0.003/1K = $1.20/ngày
PerformanceAnalyzer(Sonnet): 50 calls × 8,000 tokens × $0.003/1K = $1.20/ngày
TestCoverageChecker(Haiku):  50 calls × 4,000 tokens × $0.00025/1K = $0.05/ngày
ReportSynthesizer  (Haiku):  50 calls × 3,000 tokens × $0.00025/1K = $0.04/ngày
─────────────────────────────────────────────────────────────────────────────
Total: ~$2.49/ngày × 30 ngày = $74.70/tháng
```

**Tiết kiệm: $442.80/tháng (~85% giảm)** mà chất lượng output không thay đổi đáng kể, vì Haiku và Sonnet đã đủ tốt cho các tasks đó.

> Giá tham khảo (sẽ thay đổi). Kiểm tra pricing hiện tại tại https://anthropic.com/pricing

---

## 2. 3-Tier Model Hierarchy

### Bảng so sánh

| Model | Relative Cost | Best For | Tránh dùng khi |
|-------|--------------|----------|----------------|
| **Claude Haiku 4.5** | ~1x (baseline) | Classification, extraction, summarization, formatting, text transformation, lightweight workers | Complex reasoning, cross-file analysis, creative problem solving |
| **Claude Sonnet 4.6** | ~5x | Code generation, standard analysis, API integration, debugging single files, security audit | Simple classification tasks (overkill + cost), architectural planning |
| **Claude Opus 4.8** | ~25x | Multi-file architectural analysis, complex debugging, long-horizon planning, lead orchestrator, research synthesis | Frequent/cheap tasks, simple formatting, single-file review |

### Khi nào dùng từng model

**Haiku 4.5 — "The Workhorse"**

Haiku có ~90% capability của Sonnet cho các tasks không đòi hỏi deep reasoning. Dùng Haiku khi:

- Task có **output format rõ ràng** (extract JSON từ text, classify vào categories cố định)
- Task **không đòi hỏi suy luận đa bước** (formatting, summarization, translation)
- Task được gọi **rất thường xuyên** (>10 calls/phút)
- Task là **lightweight worker** trong pipeline (không phải orchestrator)

Ví dụ Haiku phù hợp:
```
- "Is this log line an error or info?" → CLASSIFICATION → Haiku
- "Summarize this 500-line stack trace in 3 sentences" → SUMMARIZATION → Haiku
- "Extract all TODO comments from this file" → EXTRACTION → Haiku
- "Format these findings into a markdown table" → FORMATTING → Haiku
- "Translate this error message to Vietnamese" → TRANSLATION → Haiku
```

**Sonnet 4.6 — "The Engineer"**

Sonnet là model tốt nhất cho engineering tasks. Dùng Sonnet khi:

- **Code generation** — viết implementation, tests, refactoring
- **Standard analysis** — review một file cụ thể cho bugs hoặc security issues
- **API integration** — viết code gọi external APIs với error handling
- **Debugging single files** — trace logic errors trong một class
- **Worker agents** yêu cầu code understanding

Ví dụ Sonnet phù hợp:
```
- "Write JUnit 5 tests for this UserService class" → CODE_GENERATION → Sonnet
- "Find SQL injection vulnerabilities in this repository file" → SECURITY_AUDIT → Sonnet
- "Refactor this method to use Java Streams" → CODE_REFACTORING → Sonnet
- "Debug why this Spring Boot endpoint returns 500" → DEBUGGING → Sonnet
```

**Opus 4.8 — "The Architect"**

Opus cho deep reasoning và complex, multi-step problems. Dùng Opus khi:

- **Cross-file architectural analysis** — hiểu cách 20 files interact với nhau
- **Long-horizon planning** — thiết kế migration strategy qua nhiều sprints
- **Complex debugging** — trace bug qua nhiều layers (request → service → repository → DB)
- **Lead orchestrator** — khi orchestrator cần hiểu toàn bộ context để decompose task tốt
- **Security architecture review** — đánh giá threat model của toàn bộ system

Ví dụ Opus phù hợp:
```
- "Analyze our entire auth flow and identify architectural weaknesses" → ARCHITECTURE → Opus
- "Plan migration from monolith to microservices" → PLANNING → Opus
- "Why does this distributed transaction fail under high load?" → COMPLEX_DEBUGGING → Opus
- "Orchestrate the review of this 50-file PR" → ORCHESTRATION → Opus
```

---

## 3. Practical Routing Rules

### Rule 1: Frequency-Based Routing
```
Frequency > 10 calls/minute → Haiku (cost protection)
Frequency 1-10 calls/minute → Sonnet (standard quality)
Frequency < 1 call/minute → Opus (acceptable cost)
```

### Rule 2: Task Complexity Routing
```
Single fact extraction → Haiku
Single file analysis → Sonnet
Cross-file analysis → Opus
Full codebase analysis → Opus (orchestrator) + Sonnet/Haiku (workers)
```

### Rule 3: Output Complexity Routing
```
Structured extraction (JSON/CSV) → Haiku
Code generation → Sonnet
Architectural recommendations → Opus
```

### Rule 4: Risk-Based Routing
```
Low stakes (formatting, translation) → Haiku
Medium stakes (code generation, testing) → Sonnet
High stakes (security audit, architectural decisions) → Sonnet or Opus
```

### Decision Tree

```
Task received
    │
    ├─ Is this pure classification/extraction/formatting?
    │       YES → Haiku
    │
    ├─ Does this require generating or deeply analyzing code?
    │       YES → Sonnet (single file) or Opus (multi-file/architectural)
    │
    ├─ Does this require planning, architecture, or multi-step reasoning?
    │       YES → Opus
    │
    └─ Default → Sonnet (safe middle ground)
```

---

## 4. Java Implementation: ModelRouter

### TaskType Enum

```java
package com.example.aiengineering.routing;

public enum TaskType {
    // Haiku tasks — lightweight, high-frequency
    CLASSIFICATION,
    SUMMARIZATION,
    FORMATTING,
    EXTRACTION,
    TRANSLATION,
    
    // Sonnet tasks — code and analysis
    CODE_GENERATION,
    CODE_REFACTORING,
    SINGLE_FILE_ANALYSIS,
    SECURITY_AUDIT,
    STANDARD_DEBUGGING,
    TEST_GENERATION,
    API_INTEGRATION,
    
    // Opus tasks — architecture and complex reasoning
    ARCHITECTURAL_ANALYSIS,
    COMPLEX_DEBUGGING,
    PLANNING,
    ORCHESTRATION,
    SECURITY_ARCHITECTURE,
    CROSS_FILE_ANALYSIS
}
```

### ModelRouter Utility Class

```java
package com.example.aiengineering.routing;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.Set;
import java.util.EnumSet;

/**
 * Routes tasks to the appropriate Claude model based on task complexity and cost efficiency.
 *
 * Cost hierarchy (approximate, verify current pricing at anthropic.com/pricing):
 * - Haiku 4.5:  ~$0.00025/1K tokens  (1x baseline)
 * - Sonnet 4.6: ~$0.003/1K tokens    (~12x Haiku)
 * - Opus 4.8:   ~$0.015/1K tokens    (~60x Haiku)
 */
public class ModelRouter {

    private static final Logger log = LoggerFactory.getLogger(ModelRouter.class);

    // Model identifiers — update when new versions release
    public static final String HAIKU   = "claude-haiku-4-5-20251001";
    public static final String SONNET  = "claude-sonnet-4-6-20251001";
    public static final String OPUS    = "claude-opus-4-8-20251001";

    private static final Set<TaskType> HAIKU_TASKS = EnumSet.of(
        TaskType.CLASSIFICATION,
        TaskType.SUMMARIZATION,
        TaskType.FORMATTING,
        TaskType.EXTRACTION,
        TaskType.TRANSLATION
    );

    private static final Set<TaskType> SONNET_TASKS = EnumSet.of(
        TaskType.CODE_GENERATION,
        TaskType.CODE_REFACTORING,
        TaskType.SINGLE_FILE_ANALYSIS,
        TaskType.SECURITY_AUDIT,
        TaskType.STANDARD_DEBUGGING,
        TaskType.TEST_GENERATION,
        TaskType.API_INTEGRATION
    );

    private static final Set<TaskType> OPUS_TASKS = EnumSet.of(
        TaskType.ARCHITECTURAL_ANALYSIS,
        TaskType.COMPLEX_DEBUGGING,
        TaskType.PLANNING,
        TaskType.ORCHESTRATION,
        TaskType.SECURITY_ARCHITECTURE,
        TaskType.CROSS_FILE_ANALYSIS
    );

    /**
     * Route a task to the appropriate model.
     *
     * @param taskType the type of task to perform
     * @return Claude model identifier string
     */
    public static String route(TaskType taskType) {
        String model;

        if (HAIKU_TASKS.contains(taskType)) {
            model = HAIKU;
        } else if (SONNET_TASKS.contains(taskType)) {
            model = SONNET;
        } else if (OPUS_TASKS.contains(taskType)) {
            model = OPUS;
        } else {
            // Safe default — Sonnet covers most edge cases
            log.warn("Unknown task type {}, defaulting to Sonnet", taskType);
            model = SONNET;
        }

        log.debug("Routing task {} → model {}", taskType, model);
        return model;
    }

    /**
     * Override routing for high-frequency paths (>10 calls/min).
     * Forces Haiku regardless of task type as cost protection.
     */
    public static String routeHighFrequency(TaskType taskType) {
        log.debug("High-frequency route: task {} → Haiku (cost protection)", taskType);
        return HAIKU;
    }

    /**
     * Get estimated cost per 1000 tokens for a given model.
     * Values approximate — check anthropic.com/pricing for current rates.
     *
     * @return cost in USD per 1000 tokens (output tokens)
     */
    public static double estimatedCostPer1kTokens(String model) {
        return switch (model) {
            case HAIKU   -> 0.00025;
            case SONNET  -> 0.003;
            case OPUS    -> 0.015;
            default -> throw new IllegalArgumentException("Unknown model: " + model);
        };
    }

    /**
     * Calculate estimated cost for a batch of calls.
     *
     * @param taskType     type of task
     * @param callCount    number of API calls
     * @param avgTokens    average tokens per call (input + output)
     * @return estimated cost in USD
     */
    public static double estimateMonthlyCost(TaskType taskType, int callsPerDay, int avgTokens) {
        String model = route(taskType);
        double costPer1k = estimatedCostPer1kTokens(model);
        double dailyCost = (callsPerDay * avgTokens / 1000.0) * costPer1k;
        return dailyCost * 30;
    }
}
```

### ModelRouterTest

```java
package com.example.aiengineering.routing;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class ModelRouterTest {

    @Test
    void classification_routesToHaiku() {
        assertThat(ModelRouter.route(TaskType.CLASSIFICATION))
            .isEqualTo(ModelRouter.HAIKU);
    }

    @Test
    void codeGeneration_routesToSonnet() {
        assertThat(ModelRouter.route(TaskType.CODE_GENERATION))
            .isEqualTo(ModelRouter.SONNET);
    }

    @Test
    void architecturalAnalysis_routesToOpus() {
        assertThat(ModelRouter.route(TaskType.ARCHITECTURAL_ANALYSIS))
            .isEqualTo(ModelRouter.OPUS);
    }

    @Test
    void costEstimate_sonnetCheaperThanOpus() {
        double sonnetCost = ModelRouter.estimateMonthlyCost(
            TaskType.CODE_GENERATION, 100, 5000
        );
        double opusCost = ModelRouter.estimateMonthlyCost(
            TaskType.ARCHITECTURAL_ANALYSIS, 100, 5000
        );
        assertThat(sonnetCost).isLessThan(opusCost);
    }

    @Test
    void highFrequency_alwaysRoutesToHaiku() {
        // Even code generation routes to Haiku under high-frequency override
        assertThat(ModelRouter.routeHighFrequency(TaskType.CODE_GENERATION))
            .isEqualTo(ModelRouter.HAIKU);
    }
}
```

---

## 5. Token Budget Management

Mỗi agent trong pipeline cần có **hard token limits** để tránh cost runaway.

### Token Budget Strategy

```java
package com.example.aiengineering.routing;

/**
 * Defines token budgets per agent type.
 * These are hard limits — agents that exceed these are killed.
 */
public enum AgentTokenBudget {

    // Haiku workers — cheap, short output
    SUMMARIZER(512, 1024),
    CLASSIFIER(256, 512),
    FORMATTER(512, 2048),

    // Sonnet workers — moderate output
    SECURITY_AUDITOR(8192, 4096),
    PERFORMANCE_ANALYZER(8192, 4096),
    TEST_COVERAGE_CHECKER(4096, 2048),
    CODE_GENERATOR(8192, 8192),

    // Opus — expensive, highest budget justified
    ORCHESTRATOR(16384, 4096),
    ARCHITECT(32768, 8192);

    public final int maxInputTokens;
    public final int maxOutputTokens;

    AgentTokenBudget(int maxInputTokens, int maxOutputTokens) {
        this.maxInputTokens = maxInputTokens;
        this.maxOutputTokens = maxOutputTokens;
    }
}
```

### Monitoring Token Usage

```java
import com.anthropic.client.AnthropicClient;
import com.anthropic.models.Message;
import com.anthropic.models.Usage;

public class TokenAwareAgentRunner {

    private final AnthropicClient client;
    private final TokenUsageTracker usageTracker;

    public AgentResult run(AgentConfig config, String userMessage) {
        Message response = client.messages().create(
            MessageCreateParams.builder()
                .model(config.model())
                .maxTokens(config.budget().maxOutputTokens)
                .system(config.systemPrompt())
                .addUserMessage(userMessage)
                .build()
        );

        Usage usage = response.usage();
        int inputTokens = usage.inputTokens();
        int outputTokens = usage.outputTokens();
        int totalTokens = inputTokens + outputTokens;

        // Track usage cho monitoring
        usageTracker.record(
            config.agentName(),
            config.model(),
            inputTokens,
            outputTokens,
            estimateCost(config.model(), totalTokens)
        );

        // Alert nếu usage cao bất thường
        if (totalTokens > config.budget().maxInputTokens * 0.8) {
            log.warn("Agent {} used {}% of token budget ({}/{})",
                config.agentName(),
                (totalTokens * 100) / config.budget().maxInputTokens,
                totalTokens,
                config.budget().maxInputTokens
            );
        }

        return AgentResult.success(response.content().get(0).text(), usage);
    }

    private double estimateCost(String model, int tokens) {
        return (tokens / 1000.0) * ModelRouter.estimatedCostPer1kTokens(model);
    }
}
```

---

## 6. Prompt Caching trong Multi-Agent Systems

Đây là một optimization ít được biết đến nhưng cực kỳ hiệu quả.

### Vấn đề

Trong pipeline của chúng ta, mỗi worker agent nhận **toàn bộ file contents** như là phần input. Nếu code review pipeline chạy nhiều lần trên cùng một codebase, chúng ta đang gửi lại cùng một file contents hàng trăm lần.

### Giải pháp: Prompt Caching

Anthropic hỗ trợ **prompt caching** — Claude cache phần system prompt và có thể tái sử dụng cho các requests tiếp theo. Cache hit giảm input token cost đến **90%**.

```python
import anthropic

client = anthropic.AsyncAnthropic()

# Cached system prompt — chỉ tính full cost lần đầu tiên
# Các lần sau: cache hit, ~10% cost
CACHED_CODE_CONTEXT = """
<file path="src/main/java/UserService.java">
[... toàn bộ nội dung file ...]
</file>
<file path="src/main/java/UserController.java">
[... toàn bộ nội dung file ...]
</file>
"""

async def run_with_caching(agent_task: str):
    response = await client.messages.create(
        model="claude-sonnet-4-6-20251001",
        max_tokens=4096,
        system=[
            {
                "type": "text",
                "text": SECURITY_AUDITOR_SYSTEM_PROMPT,
                "cache_control": {"type": "ephemeral"}  # Cache this block
            },
            {
                "type": "text",
                "text": CACHED_CODE_CONTEXT,
                "cache_control": {"type": "ephemeral"}  # Cache the code too
            }
        ],
        messages=[
            {"role": "user", "content": agent_task}
        ]
    )

    # Check cache performance
    usage = response.usage
    cache_read_tokens = getattr(usage, 'cache_read_input_tokens', 0)
    cache_creation_tokens = getattr(usage, 'cache_creation_input_tokens', 0)

    if cache_read_tokens > 0:
        savings_pct = (cache_read_tokens / usage.input_tokens) * 90
        print(f"Cache hit: saved ~{savings_pct:.0f}% input cost")

    return response.content[0].text
```

### Cache Cost Savings Calculation

```
Scenario: 50 review pipeline runs/ngày, average 8,000 input tokens/agent

WITHOUT caching:
  4 agents × 50 runs × 8,000 tokens × $0.003/1K = $4.80/ngày

WITH caching (after first run):
  First run: full cost = $0.096
  Remaining 49 runs: ~10% of input cost = $4.80 × 0.1 × (49/50) = $0.47
  Total: $0.096 + $0.47 = $0.57/ngày

Savings: $4.80 - $0.57 = $4.23/ngày (~88% savings on input tokens)
Monthly savings: $4.23 × 30 = $126.90/tháng
```

### Khi nào cache phát huy tác dụng

Prompt caching hiệu quả nhất khi:
1. **System prompt dài** (> 1024 tokens) và ít thay đổi
2. **Cùng codebase** được review nhiều lần trong ngày
3. **Nhiều agent** nhận cùng input context (file contents)
4. **Cache TTL**: ephemeral cache tồn tại ~5 phút — đủ cho pipeline trong một session

---

## 7. Cost Calculation Worked Example

### Scenario: Team 10 developers, 50 PRs/ngày

**Baseline (không có routing, tất cả Sonnet):**

| Agent | Calls/ngày | Avg tokens | Model | Cost/ngày |
|-------|-----------|------------|-------|-----------|
| SecurityAuditor | 50 | 10,000 | Sonnet | $1.50 |
| PerformanceAnalyzer | 50 | 10,000 | Sonnet | $1.50 |
| TestCoverageChecker | 50 | 6,000 | Sonnet | $0.90 |
| ReportSynthesizer | 50 | 4,000 | Sonnet | $0.60 |
| **Total** | | | | **$4.50/ngày** |

**Optimized (model routing + caching):**

| Agent | Calls/ngày | Avg tokens | Model | Cache? | Cost/ngày |
|-------|-----------|------------|-------|--------|-----------|
| SecurityAuditor | 50 | 10,000 | Sonnet | ✓ (8,000 cached) | $0.45 |
| PerformanceAnalyzer | 50 | 10,000 | Sonnet | ✓ (8,000 cached) | $0.45 |
| TestCoverageChecker | 50 | 6,000 | Haiku | ✓ (5,000 cached) | $0.025 |
| ReportSynthesizer | 50 | 4,000 | Haiku | ✗ (dynamic) | $0.05 |
| **Total** | | | | | **$0.975/ngày** |

**Kết quả:**
- Không có routing: $4.50/ngày × 30 = **$135/tháng**
- Với routing + caching: $0.975/ngày × 30 = **$29.25/tháng**
- **Tiết kiệm: $105.75/tháng (78%)**

> Lưu ý: Đây là ước tính minh họa. Giá thực tế phụ thuộc vào Anthropic pricing tại thời điểm bạn đọc bài này.

---

## 8. Anti-Patterns cần tránh

### Anti-Pattern 1: Dùng Opus cho mọi thứ

```python
# WRONG — cost nightmare
async def run_agent(task):
    return await client.messages.create(
        model="claude-opus-4-8",  # Always Opus
        ...
    )
```

```python
# RIGHT — route based on task type
async def run_agent(task_type: TaskType, task_content: str):
    model = ModelRouter.route(task_type)
    return await client.messages.create(model=model, ...)
```

### Anti-Pattern 2: Không set max_tokens

```python
# WRONG — agent có thể generate vô hạn tokens
response = await client.messages.create(
    model="claude-opus-4-8",
    # max_tokens not set!
    messages=[...]
)
```

```python
# RIGHT — hard limit
response = await client.messages.create(
    model="claude-opus-4-8",
    max_tokens=4096,  # Hard limit
    messages=[...]
)
```

### Anti-Pattern 3: Bỏ qua cache_control cho repeated content

```python
# WRONG — gửi lại toàn bộ code context mỗi lần
for agent in agents:
    response = await client.messages.create(
        system=f"{agent.system_prompt}\n\nCode context:\n{full_code_context}",
        ...
    )
```

```python
# RIGHT — cache code context
CACHED_SYSTEM = [
    {"type": "text", "text": agent.system_prompt},
    {"type": "text", "text": full_code_context, "cache_control": {"type": "ephemeral"}}
]

for agent in agents:
    response = await client.messages.create(
        system=CACHED_SYSTEM,  # Cache hit from second agent onwards
        ...
    )
```

---

## 9. Exercise

### Bài tập: Routing Optimization Analysis

**Part 1: Analysis**

Nhìn vào Java Code Review Pipeline từ Lesson 01. Xác định:

1. Mỗi agent hiện đang dùng model gì?
2. Đề xuất model routing mới dựa trên task type của mỗi agent
3. Tính monthly cost savings với routing mới (giả sử 50 calls/ngày)

**Part 2: Implementation**

1. Thêm `ModelRouter.java` vào project của bạn
2. Cập nhật pipeline để dùng `ModelRouter.route()` thay vì hardcoded model strings
3. Thêm token usage logging vào mỗi agent call
4. Implement prompt caching cho file contents

**Part 3: Measurement**

Chạy pipeline 5 lần và report:
- Average cost per run (before và after optimization)
- Cache hit rate (% of input tokens served from cache)
- Output quality comparison (subjective — có routing ảnh hưởng đến chất lượng không?)

**Deliverable:**
- Updated pipeline code với routing + caching
- Cost analysis spreadsheet (hoặc simple markdown table)
- Nhận xét về trade-off quality vs cost

---

## Tóm tắt

| Quyết định | Guidance |
|-----------|----------|
| Task là classification/extraction/formatting | → Haiku |
| Task là code generation hoặc single-file analysis | → Sonnet |
| Task là architectural analysis hoặc orchestration | → Opus |
| High-frequency path (>10 calls/min) | → Haiku (forced) |
| Repeated content across many calls | → Enable prompt caching |
| Không biết dùng model gì | → Sonnet (safe default) |

**Bài tiếp theo:** [Lesson 03 — Failure Modes & Production](./03-failure-modes-and-production.md) — Những gì có thể xảy ra sai trong production và cách phòng chống.
