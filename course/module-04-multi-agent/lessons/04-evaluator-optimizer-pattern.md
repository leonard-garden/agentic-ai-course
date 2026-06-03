# Lesson 04: Evaluator-Optimizer Pattern

> **Module 04 — Multi-Agent Systems** | Tuần 4 | ~90 phút

---

## Mục tiêu bài học

Sau lesson này, bạn có thể:
- Giải thích Evaluator-Optimizer pattern và khi nào nên dùng
- Thiết kế scoring rubric và feedback schema cho một domain cụ thể
- Implement feedback loop với max iterations và early exit
- Áp dụng pattern cho Java test generation và Spring Boot API documentation
- Phòng tránh các anti-patterns phổ biến (score inflation, vague feedback)

---

## 1. Pattern Overview

### Vấn đề cần giải quyết

Khi bạn yêu cầu AI generate code hoặc documentation, lần đầu tiên kết quả thường không đạt yêu cầu:
- JUnit test thiếu edge cases
- API documentation thiếu error response examples
- Code refactoring không cover tất cả code paths

Bạn có thể prompt lại thủ công — nhưng điều này không scalable và không consistent.

**Evaluator-Optimizer giải quyết điều này bằng cách tự động hóa vòng lặp cải thiện.**

### Pattern Diagram

```
┌─────────────┐     input      ┌──────────────────┐
│    User     │ ─────────────→ │   Orchestrator   │
└─────────────┘                └──────────────────┘
                                         │
                               ┌─────────▼──────────┐
                               │  GeneratorAgent    │
                               │  (Sonnet/Opus)     │
                               └─────────┬──────────┘
                                         │ output v1
                               ┌─────────▼──────────┐
                               │  EvaluatorAgent    │
                               │  (Sonnet)          │
                               └─────────┬──────────┘
                                         │ score + feedback
                                    ┌────▼────┐
                                    │score≥8? │
                                    └────┬────┘
                               NO ◄──────┴──────► YES
                               │                   │
                    ┌──────────▼──────┐    ┌───────▼──────┐
                    │ Generator gets  │    │  Return to   │
                    │ feedback, retry │    │    User      │
                    └─────────────────┘    └──────────────┘
```

### Khi nào dùng Evaluator-Optimizer

**Phù hợp:**
- Code generation với quality bar rõ ràng (coverage, style, correctness)
- Document writing với completeness checklist
- Test case generation với coverage metrics
- API documentation với accuracy requirements
- Prompt refinement với response quality scoring

**Không phù hợp:**
- Tasks có single correct answer (translation, extraction) — dùng direct generation
- Tasks không có measurable quality metric
- Khi latency quan trọng hơn quality (real-time APIs)
- Khi cost là constraint nghiêm ngặt (mỗi iteration tốn thêm tokens)

---

## 2. Key Design Decisions

Trước khi implement, bạn cần trả lời 4 câu hỏi:

### Decision 1: Quality Score là gì?

Lựa chọn scoring approach:

| Approach | Ưu | Nhược | Dùng khi |
|---------|-----|-------|---------|
| **0-10 numeric** | Dễ threshold, track trend | Subjective, inconsistent | General quality assessment |
| **Pass/Fail** | Binary, clear | No gradient for improvement | Strict compliance checks |
| **Rubric-based** | Consistent, explainable | Phức tạp hơn | Multi-dimensional quality |
| **Weighted criteria** | Flexible prioritization | Cần tune weights | Complex documents |

Ví dụ rubric-based cho JUnit tests:
```
Score = weighted_average(
    coverage_score     × 0.35,   # Line/branch coverage
    edge_cases_score   × 0.25,   # Boundary conditions, null, empty
    assertion_quality  × 0.25,   # AssertEquals vs assertTrue(x != null)
    test_isolation     × 0.15    # No shared state, deterministic
)
```

### Decision 2: Evaluator feedback như thế nào?

Feedback phải là **specific và actionable**, không phải vague:

```
VAGUE (useless):
"The tests could be improved. Add more test cases."

SPECIFIC (actionable):
"Missing tests for: (1) null input to createUser() — should throw NullPointerException,
(2) duplicate email — should throw DuplicateEmailException,
(3) email format validation — should reject 'not-an-email'.
Also: line 47 uses assertTrue(result != null) — replace with assertNotNull(result)."
```

### Decision 3: Max iterations là bao nhiêu?

Thực tế cho thấy:
- **Iteration 1 → 2**: Improvement lớn nhất (~30-40% score improvement)
- **Iteration 2 → 3**: Moderate improvement (~15-20%)
- **Iteration 3 → 4**: Diminishing returns (~5-10%)
- **Iteration 4+**: Marginal improvement, không đáng chi phí

**Recommendation: 3-5 iterations cho hầu hết use cases.**

### Decision 4: Khi nào escalate to human?

```
if score < 4 after max_iterations:
    # Fundamental problem — task description unclear, or generator struggling
    → Escalate to human: "Cannot achieve quality bar. Human review needed."

if score >= threshold:
    → Return to user automatically

if threshold - 2 <= score < threshold after max_iterations:
    → Return with warning: "Best effort result, did not reach quality threshold"
```

---

## 3. Python Implementation với Anthropic SDK

```python
import asyncio
import json
from anthropic import AsyncAnthropic
from dataclasses import dataclass
from typing import Optional

client = AsyncAnthropic()

@dataclass
class EvaluationResult:
    score: float          # 0-10
    feedback: str         # Specific, actionable
    passed: bool          # score >= threshold
    details: dict         # Rubric breakdown

@dataclass
class OptimizationResult:
    final_output: str
    final_score: float
    iterations: int
    history: list[dict]   # [{iteration, score, feedback}]
    converged: bool       # True if reached threshold, False if hit max_iterations


async def evaluator_optimizer(
    task: str,
    generator_system_prompt: str,
    evaluator_system_prompt: str,
    quality_threshold: float = 8.0,
    max_iterations: int = 5,
    generator_model: str = "claude-sonnet-4-6-20251001",
    evaluator_model: str = "claude-sonnet-4-6-20251001"
) -> OptimizationResult:
    """
    Generic evaluator-optimizer loop.

    Args:
        task: The generation task description
        generator_system_prompt: System prompt for the generator agent
        evaluator_system_prompt: System prompt for the evaluator agent
        quality_threshold: Score (0-10) at which we stop iterating
        max_iterations: Hard cap to prevent infinite loops
        generator_model: Claude model for generation
        evaluator_model: Claude model for evaluation
    """

    history = []
    current_feedback = None
    best_output = None
    best_score = 0.0

    for iteration in range(1, max_iterations + 1):
        print(f"\n--- Iteration {iteration}/{max_iterations} ---")

        # === GENERATION PHASE ===
        generator_input = task
        if current_feedback:
            generator_input = f"""
Original task:
{task}

Previous attempt feedback (iteration {iteration - 1}):
Score: {history[-1]['score']}/10
Feedback: {current_feedback}

Generate an improved version that addresses ALL the feedback points above.
"""

        gen_response = await client.messages.create(
            model=generator_model,
            max_tokens=4096,
            system=generator_system_prompt,
            messages=[{"role": "user", "content": generator_input}]
        )
        generated_output = gen_response.content[0].text
        print(f"Generated output ({len(generated_output)} chars)")

        # === EVALUATION PHASE ===
        eval_input = f"""
Evaluate the following output for this task:

## Task
{task}

## Output to Evaluate
{generated_output}

Provide your evaluation as JSON.
"""

        eval_response = await client.messages.create(
            model=evaluator_model,
            max_tokens=2048,
            system=evaluator_system_prompt,
            messages=[{"role": "user", "content": eval_input}]
        )

        evaluation = json.loads(eval_response.content[0].text)
        score = float(evaluation["score"])
        feedback = evaluation["feedback"]

        print(f"Score: {score}/10")
        print(f"Feedback: {feedback[:200]}...")

        # Track history
        history.append({
            "iteration": iteration,
            "score": score,
            "feedback": feedback,
            "output_length": len(generated_output)
        })

        # Track best result
        if score > best_score:
            best_score = score
            best_output = generated_output

        # Check convergence
        if score >= quality_threshold:
            print(f"Quality threshold {quality_threshold} reached. Stopping.")
            return OptimizationResult(
                final_output=generated_output,
                final_score=score,
                iterations=iteration,
                history=history,
                converged=True
            )

        # Prepare feedback for next iteration
        current_feedback = feedback

    # Hit max iterations without converging
    print(f"Max iterations ({max_iterations}) reached. Best score: {best_score}")
    return OptimizationResult(
        final_output=best_output,
        final_score=best_score,
        iterations=max_iterations,
        history=history,
        converged=False
    )
```

---

## 4. Java Test Generation Example

### Use Case
Generate JUnit 5 tests cho một Spring Boot service class, với evaluator kiểm tra coverage, edge cases, và assertion quality.

### Generator System Prompt

```python
JAVA_TEST_GENERATOR_PROMPT = """
You are a senior Java test engineer specializing in JUnit 5 and Mockito.

Generate comprehensive unit tests following these standards:
- JUnit 5 with @ExtendWith(MockitoExtension.class)
- Mockito for mocking dependencies
- AssertJ for assertions (assertThat, not assertTrue)
- Test method names: methodName_scenario_expectedBehavior
- AAA pattern: Arrange, Act, Assert (with comments)
- Each test method tests exactly ONE behavior
- Cover: happy path, null inputs, boundary values, error cases

Output ONLY the Java test class code, no explanation.
Start with package declaration.
"""
```

### Evaluator System Prompt

```python
JAVA_TEST_EVALUATOR_PROMPT = """
You are a senior Java code reviewer evaluating JUnit 5 test quality.

Evaluate the tests on these criteria (score each 0-10):
1. coverage_score (weight 35%): Are all public methods tested?
   Are null inputs, boundary values, error cases covered?
2. assertion_quality (weight 25%): Do tests use assertThat() with specific matchers?
   (Penalize assertTrue(x != null), use assertNotNull or assertThat(x).isNotNull())
3. test_isolation (weight 25%): Are all dependencies mocked?
   No static state, no file I/O, deterministic?
4. naming_clarity (weight 15%): Do method names follow methodName_scenario_expected pattern?

Output JSON ONLY:
{
  "score": <weighted average 0-10, one decimal>,
  "breakdown": {
    "coverage_score": <0-10>,
    "assertion_quality": <0-10>,
    "test_isolation": <0-10>,
    "naming_clarity": <0-10>
  },
  "missing_tests": ["list of specific test cases that should be added"],
  "assertion_issues": ["list of specific assertions to improve"],
  "feedback": "<3-5 sentences of specific, actionable improvement suggestions>"
}
"""
```

### Full Example Run

```python
USER_SERVICE_CODE = """
package com.example;

@Service
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;

    public User createUser(String email, String name) {
        if (email == null || email.isBlank()) {
            throw new IllegalArgumentException("Email cannot be blank");
        }
        if (userRepository.existsByEmail(email)) {
            throw new DuplicateEmailException("Email already registered: " + email);
        }
        User user = new User(UUID.randomUUID().toString(), email, name);
        User saved = userRepository.save(user);
        emailService.sendWelcomeEmail(saved.getEmail(), saved.getName());
        return saved;
    }

    public Optional<User> findById(String id) {
        return userRepository.findById(id);
    }

    public List<User> findAll(int page, int size) {
        if (page < 0) throw new IllegalArgumentException("Page must be >= 0");
        if (size <= 0 || size > 100) throw new IllegalArgumentException("Size must be 1-100");
        return userRepository.findAll(PageRequest.of(page, size)).getContent();
    }
}
"""

async def main():
    task = f"""
Generate complete JUnit 5 test class for UserService.
The service source code is:

```java
{USER_SERVICE_CODE}
```

The test class should be named UserServiceTest.
"""

    result = await evaluator_optimizer(
        task=task,
        generator_system_prompt=JAVA_TEST_GENERATOR_PROMPT,
        evaluator_system_prompt=JAVA_TEST_EVALUATOR_PROMPT,
        quality_threshold=8.0,
        max_iterations=4
    )

    print(f"\n=== FINAL RESULT ===")
    print(f"Score: {result.final_score}/10")
    print(f"Iterations: {result.iterations}")
    print(f"Converged: {result.converged}")
    print(f"\nScore progression: {[h['score'] for h in result.history]}")
    print(f"\n--- Generated Tests ---")
    print(result.final_output)

    # Save to file
    with open("UserServiceTest.java", "w") as f:
        f.write(result.final_output)
    print("\nSaved to UserServiceTest.java")

asyncio.run(main())
```

### Typical Score Progression

```
Iteration 1: 5.2/10
  Missing: null name test, page=-1 test, size=101 test
  Weak assertions: assertTrue(user != null)

Iteration 2: 7.4/10
  Added null name test ✓, boundary tests ✓
  Still using assertTrue for some assertions

Iteration 3: 8.6/10  ← Threshold reached, stop
  All assertions use assertThat ✓
  All boundary conditions covered ✓
```

---

## 5. Spring Boot API Documentation Example

### Use Case
Generate OpenAPI descriptions cho REST endpoints, evaluator kiểm tra completeness và accuracy.

### Generator Prompt

```python
API_DOC_GENERATOR_PROMPT = """
You are a technical writer specializing in REST API documentation.

Generate OpenAPI 3.0 YAML documentation for the given endpoint(s).
Requirements:
- Every endpoint must have: summary, description, all parameters documented
- Request body: schema with all fields, required fields marked, example provided
- Responses: 200 success with schema + example, 400 with error schema, 401, 404 where applicable
- Use clear, developer-friendly language in descriptions
- Include a realistic example for every schema

Output ONLY valid YAML, starting with 'paths:'.
"""
```

### Evaluator Prompt

```python
API_DOC_EVALUATOR_PROMPT = """
You are a developer experience (DevEx) engineer reviewing API documentation quality.

Evaluate on these criteria:
1. completeness (weight 40%): Are all endpoints covered? All parameters documented?
   All response codes (200, 400, 401, 404, 500) documented where applicable?
2. accuracy (weight 30%): Does the schema match the actual request/response structure?
   Are required fields correctly marked?
3. examples_quality (weight 20%): Is there a realistic example for request AND response?
   Are examples valid JSON/YAML?
4. clarity (weight 10%): Are descriptions clear to a developer unfamiliar with the system?

Output JSON ONLY:
{
  "score": <weighted average 0-10>,
  "breakdown": {
    "completeness": <0-10>,
    "accuracy": <0-10>,
    "examples_quality": <0-10>,
    "clarity": <0-10>
  },
  "missing_items": ["specific items that are missing"],
  "feedback": "<specific, actionable improvements>"
}
"""
```

---

## 6. Java Implementation: EvaluatorOptimizerService

```java
package com.example.aiengineering.evaluator;

import com.anthropic.client.AnthropicClient;
import com.anthropic.models.*;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

public class EvaluatorOptimizerService {

    private static final Logger log = LoggerFactory.getLogger(EvaluatorOptimizerService.class);
    private static final ObjectMapper objectMapper = new ObjectMapper();

    private final AnthropicClient client;
    private final double qualityThreshold;
    private final int maxIterations;

    public EvaluatorOptimizerService(AnthropicClient client) {
        this(client, 8.0, 5);
    }

    public EvaluatorOptimizerService(
        AnthropicClient client,
        double qualityThreshold,
        int maxIterations
    ) {
        this.client = client;
        this.qualityThreshold = qualityThreshold;
        this.maxIterations = maxIterations;
    }

    public OptimizationResult optimize(OptimizationRequest request) {
        List<IterationRecord> history = new ArrayList<>();
        String currentFeedback = null;
        String bestOutput = null;
        double bestScore = 0.0;

        for (int iteration = 1; iteration <= maxIterations; iteration++) {
            log.info("Starting iteration {}/{}", iteration, maxIterations);

            // Build generator input
            String generatorInput = buildGeneratorInput(
                request.task(), currentFeedback, iteration, history
            );

            // Generate
            String generatedOutput = callAgent(
                request.generatorModel(),
                request.generatorSystemPrompt(),
                generatorInput,
                request.generatorMaxTokens()
            );

            // Evaluate
            String evaluatorInput = buildEvaluatorInput(request.task(), generatedOutput);
            String evaluationJson = callAgent(
                request.evaluatorModel(),
                request.evaluatorSystemPrompt(),
                evaluatorInput,
                2048
            );

            EvaluationResult evaluation = parseEvaluation(evaluationJson);
            double score = evaluation.score();

            log.info("Iteration {}: score={}/10, feedback={}",
                iteration, score, evaluation.feedback().substring(0, Math.min(100, evaluation.feedback().length())));

            history.add(new IterationRecord(iteration, score, evaluation.feedback()));

            if (score > bestScore) {
                bestScore = score;
                bestOutput = generatedOutput;
            }

            if (score >= qualityThreshold) {
                log.info("Quality threshold {} reached at iteration {}", qualityThreshold, iteration);
                return OptimizationResult.converged(generatedOutput, score, iteration, history);
            }

            currentFeedback = evaluation.feedback();
        }

        log.warn("Max iterations {} reached. Best score: {}", maxIterations, bestScore);
        return OptimizationResult.maxIterationsReached(bestOutput, bestScore, maxIterations, history);
    }

    private String buildGeneratorInput(
        String task, String feedback, int iteration, List<IterationRecord> history
    ) {
        if (feedback == null) {
            return task;
        }

        IterationRecord last = history.get(history.size() - 1);
        return String.format("""
            Original task:
            %s

            Previous attempt (iteration %d, score: %.1f/10):
            Feedback: %s

            Generate an improved version that addresses ALL feedback points.
            """,
            task, iteration - 1, last.score(), feedback
        );
    }

    private String buildEvaluatorInput(String task, String output) {
        return String.format("""
            Evaluate the following output for this task:

            ## Task
            %s

            ## Output to Evaluate
            %s

            Provide evaluation as JSON.
            """,
            task, output
        );
    }

    private String callAgent(String model, String systemPrompt, String userMessage, int maxTokens) {
        Message response = client.messages().create(
            MessageCreateParams.builder()
                .model(model)
                .maxTokens(maxTokens)
                .system(systemPrompt)
                .addUserMessage(userMessage)
                .build()
        );
        return response.content().get(0).text();
    }

    private EvaluationResult parseEvaluation(String json) {
        try {
            Map<String, Object> parsed = objectMapper.readValue(json, Map.class);
            double score = ((Number) parsed.get("score")).doubleValue();
            String feedback = (String) parsed.get("feedback");
            return new EvaluationResult(score, feedback, parsed);
        } catch (Exception e) {
            log.error("Failed to parse evaluation JSON: {}", json, e);
            return new EvaluationResult(0.0, "Evaluation parse failed: " + e.getMessage(), Map.of());
        }
    }
}
```

```java
// Supporting records
public record OptimizationRequest(
    String task,
    String generatorModel,
    String generatorSystemPrompt,
    int generatorMaxTokens,
    String evaluatorModel,
    String evaluatorSystemPrompt
) {}

public record EvaluationResult(double score, String feedback, Map<String, Object> rawData) {}

public record IterationRecord(int iteration, double score, String feedback) {}

public record OptimizationResult(
    String finalOutput,
    double finalScore,
    int iterations,
    List<IterationRecord> history,
    boolean converged
) {
    public static OptimizationResult converged(
        String output, double score, int iterations, List<IterationRecord> history
    ) {
        return new OptimizationResult(output, score, iterations, history, true);
    }

    public static OptimizationResult maxIterationsReached(
        String output, double score, int iterations, List<IterationRecord> history
    ) {
        return new OptimizationResult(output, score, iterations, history, false);
    }

    public String scoreProgression() {
        return history.stream()
            .map(r -> String.format("%.1f", r.score()))
            .reduce((a, b) -> a + " → " + b)
            .orElse("no iterations");
    }
}
```

---

## 7. Anti-Patterns

### Anti-Pattern 1: Vague Evaluator Feedback

```
# BAD — Generator không biết phải cải thiện gì
{
  "score": 5,
  "feedback": "The tests need to be better. Add more coverage."
}

# GOOD — Generator có roadmap cụ thể
{
  "score": 5,
  "feedback": "Three specific issues: (1) No test for null email in createUser() — add testCreateUser_nullEmail_throwsIllegalArgumentException. (2) Line 34: assertTrue(result.isPresent()) should be assertThat(result).isPresent(). (3) findAll() is not tested at all — add tests for page=0, page=-1, size=0, size=100, size=101."
}
```

### Anti-Pattern 2: Score Inflation

Evaluator bị "nice" và cho score cao ngay từ đầu:

```python
# BAD evaluator prompt — không có teeth
"Evaluate the quality of these tests. Give a score from 1-10."

# GOOD evaluator prompt — strict criteria
"Evaluate with these STRICT criteria:
- Deduct 2 points for each missing edge case test
- Deduct 1 point for each assertTrue() that should use assertThat()
- Deduct 1 point for each undocumented test scenario
A score of 10 means production-ready tests. Be strict."
```

### Anti-Pattern 3: Generator Ignoring Feedback

Xảy ra khi feedback không được included trong generator prompt rõ ràng:

```python
# BAD — feedback bị chôn vùi
generator_input = f"Task: {task}\nContext: {previous_output}\n{feedback}"

# GOOD — feedback được emphasize
generator_input = f"""
Task: {task}

CRITICAL FEEDBACK FROM PREVIOUS ATTEMPT (you MUST address all these points):
{feedback}

Previous attempt (for reference only, do not copy):
{previous_output}

Generate a new, improved version that explicitly addresses every feedback point.
"""
```

### Anti-Pattern 4: No Max Iterations Guard

```python
# CATASTROPHIC — infinite loop nếu score không bao giờ đạt threshold
while True:
    output = generate(task)
    score = evaluate(output)
    if score >= 8:
        break
    # Nếu score luôn là 7.8, loop forever → $$$

# CORRECT — always bounded
for i in range(MAX_ITERATIONS):
    ...
    if score >= threshold:
        break
```

---

## 8. When to Use Each Model Tier in E-O Loops

| Role | Recommended Model | Reasoning |
|------|------------------|-----------|
| Generator (simple text) | Sonnet | Good quality, reasonable cost |
| Generator (code generation) | Sonnet | Best coding model at reasonable cost |
| Generator (complex architecture) | Opus | Depth needed for first attempt quality |
| Evaluator (scoring+feedback) | Sonnet | Needs enough intelligence to find real issues |
| Evaluator (lightweight pass/fail) | Haiku | If criteria are very clear and simple |

**Cost tip**: Dùng Sonnet cho cả generator và evaluator. Opus cho generator chỉ khi task rất phức tạp (full architecture docs, complex algorithm). Haiku cho evaluator chỉ khi criteria đơn giản và rõ ràng.

---

## 9. Exercise

### Bài tập: Build Evaluator-Optimizer cho Java Code Generation

**Phần 1: JUnit Test Generator**

Lấy một Service class trong project của bạn (hoặc dùng Spring PetClinic), build evaluator-optimizer để:
1. Generate JUnit 5 tests cho tất cả public methods
2. Evaluate với rubric: coverage (35%), assertion quality (25%), isolation (25%), naming (15%)
3. Loop tối đa 4 iterations, stop khi score >= 8
4. Print score progression sau mỗi run

**Phần 2: Measure Improvement**

Chạy pipeline 3 lần với cùng input và report:
- Average score tại iteration 1 (baseline quality)
- Average score tại convergence
- Average iterations to convergence
- Có lần nào không converge trong 4 iterations không?

**Phần 3: Tune the Evaluator**

Thử 2 phiên bản evaluator:
- **Strict**: Penalize mạnh mọi thiếu sót nhỏ
- **Lenient**: Chỉ penalize major issues

So sánh: Phiên bản nào cho final output tốt hơn? Phiên bản nào converge nhanh hơn?

**Deliverable:**
- Code evaluator-optimizer hoàn chỉnh
- Sample output showing score progression (ít nhất 3 iterations)
- Nhận xét về strict vs lenient evaluator trade-off

---

## Tóm tắt

| Quyết định | Guidance |
|-----------|---------|
| Quality metric | Rubric-based với weighted criteria cho best consistency |
| Feedback style | Specific, actionable, list cụ thể những gì cần thêm/thay đổi |
| Max iterations | 3-5 cho hầu hết use cases |
| Threshold | 8/10 là reasonable cho code quality |
| Generator model | Sonnet (code), Opus (complex architecture) |
| Evaluator model | Sonnet (reliable judgment) |
| When to escalate | score < 4 after max iterations |

**Bài tiếp theo:** [Project — Production Java Code Review Pipeline](../exercises/project-java-review-pipeline.md) — Tổng hợp tất cả lessons vào một production-grade system.
