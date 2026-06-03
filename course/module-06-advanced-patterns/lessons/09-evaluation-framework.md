# Lesson 09: Evaluation Frameworks — Measuring Agent Quality

> **Module 06 — Advanced Agentic Patterns**
> Prerequisite: Lessons 01-08
> Thời lượng ước tính: 90 phút

---

## Tại Sao Evaluation Khó?

Khi bạn viết một function Java truyền thống, test dễ: `assertEquals(expected, actual)`. Deterministic input → deterministic output. Nhưng với agents:

- Cùng một prompt có thể cho kết quả khác nhau mỗi lần (non-deterministic)
- "Correct" output không có một đáp án duy nhất (open-ended)
- Agent có thể hoàn thành task theo nhiều cách khác nhau, tất cả đều valid
- Performance phụ thuộc vào context: độ phức tạp của task, chất lượng retrieved documents, tool availability

**Hậu quả thực tế**: Bạn demo agent và nó hoạt động hoàn hảo. Bạn deploy. Một tuần sau, người dùng báo cáo nó fail liên tục. Vấn đề? Bạn đã đánh giá chất lượng dựa trên 3-5 lần chạy thủ công, không có systematic measurement.

Evaluation framework giải quyết điều này bằng cách:
1. Định nghĩa rõ ràng "quality" nghĩa là gì
2. Đo lường nhiều lần để có statistical confidence
3. Tự động hóa để chạy liên tục (CI/CD)
4. Track regression khi bạn thay đổi prompts hoặc tools

---

## The pass@1 Problem

Từ nghiên cứu arXiv:2603.29231 (March 2026), "Single-Run Evaluation Bias in Autonomous Agents":

> An agent with 60% single-run pass rate may only maintain 25% consistent performance across 8 runs.

Điều này có nghĩa là nếu bạn chỉ chạy eval một lần và thấy agent "pass", bạn không biết liệu nó có thực sự reliable hay chỉ may mắn trong lần đó.

```
Agent A: Run 1=PASS, Run 2=FAIL, Run 3=PASS, Run 4=FAIL, Run 5=PASS
→ pass@1 = 100% (nếu bạn chỉ chạy Run 1)
→ pass@5 = 60% (thực tế)

Agent B: Run 1=PASS, Run 2=PASS, Run 3=PASS, Run 4=PASS, Run 5=FAIL
→ pass@1 = 100% (giống A trên mặt)
→ pass@5 = 80% (tốt hơn A đáng kể)
```

Moral: **Luôn dùng pass@k với k ≥ 5** cho production evaluation. pass@1 chỉ dùng cho quick sanity check trong development.

---

## Multi-Dimensional Evaluation Framework

Đánh giá agent chỉ bằng "pass/fail" quá thô. Framework này đo 5 dimensions:

```
┌───────────────────────────────────────────────────────────────┐
│              5-DIMENSION EVAL FRAMEWORK                        │
├─────────────────────┬─────────────────────────────────────────┤
│ Dimension 1         │ Task Completion Rate                    │
│                     │ Agent hoàn thành task không?            │
├─────────────────────┼─────────────────────────────────────────┤
│ Dimension 2         │ Output Quality                          │
│                     │ Output tốt đến mức nào?                 │
├─────────────────────┼─────────────────────────────────────────┤
│ Dimension 3         │ Reliability (pass@k)                    │
│                     │ Kết quả có consistent không?            │
├─────────────────────┼─────────────────────────────────────────┤
│ Dimension 4         │ Cost Efficiency                         │
│                     │ Bao nhiêu tokens per task?              │
├─────────────────────┼─────────────────────────────────────────┤
│ Dimension 5         │ Latency                                 │
│                     │ Mất bao lâu để complete?                │
└─────────────────────┴─────────────────────────────────────────┘
```

---

## Dimension 1: Task Completion Rate

### Binary vs Graded Completion

**Binary completion**: Task hoàn thành (1) hoặc không (0). Đơn giản nhất.

**Graded completion**: Partial credit cho partial completion. Phù hợp với complex tasks có nhiều sub-requirements.

```java
public record EvalResult(
    String taskId,
    String agentId,
    boolean completed,          // Binary: did it complete at all?
    float completionScore,      // Graded: 0.0 to 1.0
    String failureReason,       // Null if completed
    Duration duration,
    int inputTokenCount,
    int outputTokenCount,
    int totalTokenCount,        // input + output
    List<String> completedSubtasks,
    List<String> failedSubtasks,
    Instant timestamp
) {
    public int tokenCount() { return totalTokenCount; }

    public static EvalResult success(String taskId, String agentId,
                                      Duration duration, int tokens) {
        return new EvalResult(taskId, agentId, true, 1.0f, null,
            duration, 0, tokens, tokens,
            List.of(), List.of(), Instant.now());
    }

    public static EvalResult failure(String taskId, String agentId,
                                      String reason, Duration duration, int tokens) {
        return new EvalResult(taskId, agentId, false, 0.0f, reason,
            duration, 0, tokens, tokens,
            List.of(), List.of(), Instant.now());
    }

    public static EvalResult partial(String taskId, String agentId,
                                      float score, String reason,
                                      List<String> completed, List<String> failed,
                                      Duration duration, int tokens) {
        return new EvalResult(taskId, agentId, score > 0, score, reason,
            duration, 0, tokens, tokens,
            completed, failed, Instant.now());
    }
}
```

### Automated Completion Check

Với well-defined tasks, viết automated checker:

```java
@Component
public class TaskCompletionChecker {

    /**
     * Check task completion dựa trên expected outcomes.
     * Trả về graded score (0.0 - 1.0).
     */
    public float checkCompletion(String taskOutput, TaskExpectation expectation) {
        List<String> required = expectation.requiredElements();
        List<String> optional = expectation.optionalElements();

        long requiredMet = required.stream()
            .filter(req -> taskOutput.contains(req) || semanticallyContains(taskOutput, req))
            .count();

        long optionalMet = optional.stream()
            .filter(opt -> taskOutput.contains(opt) || semanticallyContains(taskOutput, opt))
            .count();

        // Required elements: 80% weight, optional: 20% weight
        float requiredScore = required.isEmpty() ? 1.0f :
            (float) requiredMet / required.size();
        float optionalScore = optional.isEmpty() ? 1.0f :
            (float) optionalMet / optional.size();

        return requiredScore * 0.8f + optionalScore * 0.2f;
    }
}
```

---

## Dimension 2: Output Quality

### Code Quality Evaluation

Với code generation agent, quality có thể đo tự động:

```java
@Service
public class CodeQualityEvaluator {

    private final JavaCompilerService compiler;
    private final TestRunner testRunner;
    private final CheckstyleRunner linter;

    /**
     * Đánh giá chất lượng code được generate.
     * Trả về score từ 0.0 (unusable) đến 1.0 (perfect).
     */
    public CodeQualityScore evaluate(String generatedCode,
                                      List<String> testCases,
                                      CodeQualityConfig config) {
        // Stage 1: Compilation (prerequisite)
        CompileResult compile = compiler.compile(generatedCode);
        if (!compile.success()) {
            return CodeQualityScore.builder()
                .compilationPassed(false)
                .overallScore(0.0f)
                .failureReason("Compilation failed: " + compile.errorMessage())
                .build();
        }

        // Stage 2: Test execution
        float testScore = 0.0f;
        if (!testCases.isEmpty()) {
            int passed = (int) testCases.stream()
                .filter(test -> {
                    try {
                        return testRunner.runTest(generatedCode, test).passed();
                    } catch (Exception e) {
                        return false;
                    }
                })
                .count();
            testScore = (float) passed / testCases.size();
        } else {
            testScore = 1.0f; // No tests = assume passing
        }

        // Stage 3: Code style
        LintResult lint = linter.check(generatedCode);
        float lintScore = lint.hasCriticalIssues() ? 0.5f :
                          lint.hasMinorIssues() ? 0.8f : 1.0f;

        // Stage 4: Complexity check (simpler code = better)
        float complexityScore = evaluateComplexity(generatedCode);

        // Weighted final score
        float overallScore = testScore * 0.6f     // Tests: 60%
                           + lintScore * 0.2f      // Style: 20%
                           + complexityScore * 0.2f; // Complexity: 20%

        return CodeQualityScore.builder()
            .compilationPassed(true)
            .testScore(testScore)
            .lintScore(lintScore)
            .complexityScore(complexityScore)
            .overallScore(overallScore)
            .passedTests((int)(testScore * testCases.size()))
            .totalTests(testCases.size())
            .build();
    }

    private float evaluateComplexity(String code) {
        // Đếm cyclomatic complexity đơn giản
        long ifCount = code.lines().filter(l -> l.trim().startsWith("if ")).count();
        long forCount = code.lines().filter(l -> l.trim().startsWith("for ")).count();
        long whileCount = code.lines().filter(l -> l.trim().startsWith("while ")).count();
        long totalLines = code.lines().count();

        double branchDensity = (double)(ifCount + forCount + whileCount) / totalLines;
        // Low branch density = simpler code = higher score
        return (float) Math.max(0, 1.0 - branchDensity * 10);
    }
}
```

### Document Quality: Human Rubric

Với documents, code không thể tự đánh giá — cần rubric:

```java
public record DocumentQualityRubric(
    int accuracyScore,        // 1-5: Facts correct?
    int completenessScore,    // 1-5: All required sections present?
    int clarityScore,         // 1-5: Easy to understand?
    int formattingScore,      // 1-5: Well structured?
    String reviewerNotes
) {
    public float normalizedScore() {
        return (accuracyScore + completenessScore + clarityScore + formattingScore)
            / 20.0f; // Max 20 points → normalize to 0.0-1.0
    }
}
```

Automate phần có thể automate:

```java
@Service
public class AutomatedDocumentChecker {

    public DocumentAutoScore autoCheck(String document, DocumentSpec spec) {
        // Check required sections present
        long sectionsFound = spec.requiredSections().stream()
            .filter(section -> document.toLowerCase().contains(section.toLowerCase()))
            .count();
        float sectionScore = (float) sectionsFound / spec.requiredSections().size();

        // Check minimum length
        int wordCount = document.split("\\s+").length;
        float lengthScore = wordCount >= spec.minimumWords() ? 1.0f :
            (float) wordCount / spec.minimumWords();

        // Check code examples if required
        float codeScore = 1.0f;
        if (spec.requiresCodeExamples()) {
            boolean hasCode = document.contains("```") || document.contains("    ");
            codeScore = hasCode ? 1.0f : 0.0f;
        }

        return new DocumentAutoScore(sectionScore, lengthScore, codeScore,
            (sectionScore + lengthScore + codeScore) / 3.0f);
    }
}
```

---

## Dimension 3: Reliability (pass@k)

### Implementation

```java
@Service
public class ReliabilityMeasurer {

    private final AgentRunner agentRunner;

    /**
     * Đo reliability bằng cách chạy cùng task k lần.
     * Trả về pass rate và confidence interval.
     */
    public ReliabilityReport measureReliability(EvalCase evalCase, int k) {
        log.info("Measuring reliability for task '{}' with k={}", evalCase.taskId(), k);

        List<EvalResult> results = IntStream.range(0, k)
            .parallel() // Chạy song song để tiết kiệm thời gian
            .mapToObj(i -> {
                log.debug("Running iteration {}/{} for task {}", i+1, k, evalCase.taskId());
                return agentRunner.runSingle(evalCase);
            })
            .toList();

        long passCount = results.stream().filter(EvalResult::completed).count();
        float passRate = (float) passCount / k;

        // Wilson score confidence interval (more accurate than normal approximation)
        float[] ci = wilsonConfidenceInterval(passCount, k, 0.95f);

        // Latency stats
        LongSummaryStatistics latencyStats = results.stream()
            .mapToLong(r -> r.duration().toMillis())
            .summaryStatistics();

        return ReliabilityReport.builder()
            .taskId(evalCase.taskId())
            .k(k)
            .passCount((int) passCount)
            .passRate(passRate)
            .confidenceIntervalLow(ci[0])
            .confidenceIntervalHigh(ci[1])
            .medianLatencyMs((long) latencyStats.getAverage())
            .results(results)
            .build();
    }

    /**
     * Wilson score confidence interval cho binomial proportion.
     * Chính xác hơn normal approximation đặc biệt khi k nhỏ.
     */
    private float[] wilsonConfidenceInterval(long successes, int n, float confidence) {
        double z = 1.96; // 95% confidence
        double p = (double) successes / n;
        double denominator = 1 + (z * z / n);
        double center = (p + z * z / (2 * n)) / denominator;
        double margin = z * Math.sqrt(p * (1 - p) / n + z * z / (4 * n * n)) / denominator;
        return new float[]{(float)(center - margin), (float)(center + margin)};
    }
}
```

### Reliability Decay Analysis

Agents thường perform tệ hơn khi task complexity tăng. Đo sự suy giảm này:

```java
@Service
public class ReliabilityDecayAnalyzer {

    private final ReliabilityMeasurer measurer;

    /**
     * Đo reliability decay theo task complexity.
     * Tạo curve: x = complexity level, y = pass rate
     */
    public DecayCurve analyzeDecay(List<EvalCase> casesByComplexity, int k) {
        // Cases phải được sort theo complexity (low → high)
        List<DataPoint> points = casesByComplexity.stream()
            .map(evalCase -> {
                ReliabilityReport report = measurer.measureReliability(evalCase, k);
                return new DataPoint(evalCase.complexityLevel(), report.passRate());
            })
            .toList();

        // Tính slope để xem decay rate
        float decayRate = computeDecayRate(points);

        return new DecayCurve(points, decayRate,
            interpretDecay(decayRate));
    }

    private String interpretDecay(float decayRate) {
        if (decayRate > -0.05f) return "STABLE: Performance maintains well across complexity";
        if (decayRate > -0.15f) return "GRADUAL: Some degradation at high complexity";
        if (decayRate > -0.30f) return "SIGNIFICANT: Consider breaking complex tasks down";
        return "SEVERE: Agent struggles significantly with complex tasks";
    }
}
```

---

## Dimension 4: Cost Efficiency

### Token Economics

```java
@Service
public class CostAnalyzer {

    // Giá tham khảo (cập nhật theo pricing thực tế)
    private static final double CLAUDE_SONNET_INPUT_PER_1K = 0.003;
    private static final double CLAUDE_SONNET_OUTPUT_PER_1K = 0.015;

    public CostReport analyzeCost(List<EvalResult> results) {
        List<EvalResult> successes = results.stream()
            .filter(EvalResult::completed).toList();
        List<EvalResult> failures = results.stream()
            .filter(r -> !r.completed()).toList();

        DoubleSummaryStatistics successTokenStats = successes.stream()
            .mapToDouble(EvalResult::tokenCount)
            .summaryStatistics();

        DoubleSummaryStatistics failureTokenStats = failures.stream()
            .mapToDouble(EvalResult::tokenCount)
            .summaryStatistics();

        double avgTokensOnSuccess = successTokenStats.getAverage();
        double avgTokensOnFailure = failureTokenStats.getAverage();

        // Tính cost per successful task (including failed attempts)
        double totalTokens = results.stream().mapToDouble(EvalResult::tokenCount).sum();
        double totalCost = estimateCost(totalTokens);
        double costPerSuccess = successes.isEmpty() ? Double.POSITIVE_INFINITY :
            totalCost / successes.size();

        // Token efficiency: successful output tokens / total tokens spent
        float efficiency = (float)(successTokenStats.getSum() / totalTokens);

        return CostReport.builder()
            .totalRuns(results.size())
            .successfulRuns(successes.size())
            .avgTokensOnSuccess((long) avgTokensOnSuccess)
            .avgTokensOnFailure((long) avgTokensOnFailure)
            .totalCostUsd(totalCost)
            .costPerSuccessfulTaskUsd(costPerSuccess)
            .tokenEfficiency(efficiency)
            .build();
    }

    private double estimateCost(double totalTokens) {
        // Rough estimate: assume 60% input, 40% output
        double inputTokens = totalTokens * 0.6;
        double outputTokens = totalTokens * 0.4;
        return (inputTokens / 1000 * CLAUDE_SONNET_INPUT_PER_1K)
             + (outputTokens / 1000 * CLAUDE_SONNET_OUTPUT_PER_1K);
    }
}
```

### Cost Trend Tracking

Track cost theo thời gian để detect regressions:

```java
@Service
public class CostTrendTracker {

    private final EvalResultRepository evalRepo;

    public CostTrend analyzeTrend(String agentId, int weeks) {
        List<WeeklyCostSummary> weeklyData = new ArrayList<>();

        for (int w = weeks - 1; w >= 0; w--) {
            LocalDate weekStart = LocalDate.now().minusWeeks(w);
            LocalDate weekEnd = weekStart.plusWeeks(1);

            List<EvalResult> weekResults = evalRepo.findByAgentAndDateRange(
                agentId, weekStart, weekEnd
            );

            if (!weekResults.isEmpty()) {
                double avgCost = weekResults.stream()
                    .mapToDouble(r -> estimateCost(r.tokenCount()))
                    .average().orElse(0);
                weeklyData.add(new WeeklyCostSummary(weekStart, avgCost, weekResults.size()));
            }
        }

        // Detect if cost is trending up (regression) or down (improvement)
        float costTrend = computeTrend(weeklyData);
        return new CostTrend(weeklyData, costTrend,
            costTrend > 0.10f ? "INCREASING — investigate prompt changes" :
            costTrend < -0.10f ? "DECREASING — optimization working" :
            "STABLE");
    }
}
```

---

## Dimension 5: Latency

```java
@Service
public class LatencyProfiler {

    public LatencyReport profile(List<EvalResult> results) {
        LongSummaryStatistics stats = results.stream()
            .mapToLong(r -> r.duration().toMillis())
            .summaryStatistics();

        // Tính percentiles
        long[] sortedMs = results.stream()
            .mapToLong(r -> r.duration().toMillis())
            .sorted()
            .toArray();

        return LatencyReport.builder()
            .p50Ms(percentile(sortedMs, 50))
            .p95Ms(percentile(sortedMs, 95))
            .p99Ms(percentile(sortedMs, 99))
            .minMs(stats.getMin())
            .maxMs(stats.getMax())
            .avgMs((long) stats.getAverage())
            // Distribution: < 5s, 5-15s, 15-30s, > 30s
            .under5s((int) results.stream().filter(r -> r.duration().getSeconds() < 5).count())
            .between5and15s((int) results.stream()
                .filter(r -> r.duration().getSeconds() >= 5 && r.duration().getSeconds() < 15)
                .count())
            .over30s((int) results.stream().filter(r -> r.duration().getSeconds() >= 30).count())
            .build();
    }

    private long percentile(long[] sorted, int p) {
        int index = (int) Math.ceil(p / 100.0 * sorted.length) - 1;
        return sorted[Math.max(0, Math.min(index, sorted.length - 1))];
    }
}
```

---

## Building the Eval Harness

### Core Harness

```java
@Component
public class AgentEvalHarness {

    private final AgentRunner agentRunner;
    private final TaskCompletionChecker completionChecker;
    private final CostAnalyzer costAnalyzer;
    private final LatencyProfiler latencyProfiler;
    private final ReliabilityMeasurer reliabilityMeasurer;

    /**
     * Chạy đầy đủ evaluation suite.
     *
     * @param agent       Agent cần evaluate
     * @param testCases   Danh sách test cases
     * @param runsPerCase Số lần chạy mỗi case (khuyến nghị: 5+)
     */
    public EvalReport run(Agent agent, List<EvalCase> testCases, int runsPerCase) {
        log.info("Starting eval: {} test cases × {} runs = {} total runs",
            testCases.size(), runsPerCase, testCases.size() * runsPerCase);

        Instant startTime = Instant.now();

        // Chạy tất cả cases song song (independent)
        List<EvalResult> allResults = testCases.parallelStream()
            .flatMap(tc -> IntStream.range(0, runsPerCase)
                .mapToObj(i -> evaluate(agent, tc)))
            .collect(Collectors.toList());

        Duration totalDuration = Duration.between(startTime, Instant.now());

        // Aggregate results theo từng dimension
        float overallPassRate = (float) allResults.stream()
            .filter(EvalResult::completed).count() / allResults.size();

        CostReport costReport = costAnalyzer.analyzeCost(allResults);
        LatencyReport latencyReport = latencyProfiler.profile(allResults);

        // Per-case reliability
        Map<String, Float> passRateByCase = testCases.stream()
            .collect(Collectors.toMap(
                EvalCase::taskId,
                tc -> {
                    List<EvalResult> caseResults = allResults.stream()
                        .filter(r -> r.taskId().equals(tc.taskId()))
                        .toList();
                    return (float) caseResults.stream()
                        .filter(EvalResult::completed).count() / caseResults.size();
                }
            ));

        // Identify hardest cases
        List<String> hardestCases = passRateByCase.entrySet().stream()
            .filter(e -> e.getValue() < 0.5f)
            .sorted(Map.Entry.comparingByValue())
            .map(Map.Entry::getKey)
            .toList();

        EvalReport report = EvalReport.builder()
            .agentId(agent.getId())
            .timestamp(startTime)
            .totalRuns(allResults.size())
            .overallPassRate(overallPassRate)
            .passRateByCase(passRateByCase)
            .hardestCases(hardestCases)
            .costReport(costReport)
            .latencyReport(latencyReport)
            .totalEvalDuration(totalDuration)
            .build();

        logSummary(report);
        return report;
    }

    private EvalResult evaluate(Agent agent, EvalCase evalCase) {
        Instant start = Instant.now();
        try {
            AgentResponse response = agentRunner.run(agent, evalCase.input());
            Duration duration = Duration.between(start, Instant.now());

            float completionScore = completionChecker.checkCompletion(
                response.output(), evalCase.expectation()
            );

            return completionScore >= evalCase.passingThreshold()
                ? EvalResult.success(evalCase.taskId(), agent.getId(), duration,
                    response.totalTokens())
                : EvalResult.partial(evalCase.taskId(), agent.getId(), completionScore,
                    "Score below threshold: " + completionScore,
                    List.of(), List.of(), duration, response.totalTokens());

        } catch (Exception e) {
            Duration duration = Duration.between(start, Instant.now());
            log.warn("Eval failed for task {}: {}", evalCase.taskId(), e.getMessage());
            return EvalResult.failure(evalCase.taskId(), agent.getId(),
                e.getMessage(), duration, 0);
        }
    }

    private void logSummary(EvalReport report) {
        log.info("=== EVAL REPORT ===");
        log.info("Agent: {}", report.agentId());
        log.info("Overall Pass Rate: {:.1f}%", report.overallPassRate() * 100);
        log.info("Cost per successful task: ${:.4f}", report.costReport().costPerSuccessfulTaskUsd());
        log.info("P95 Latency: {}ms", report.latencyReport().p95Ms());
        if (!report.hardestCases().isEmpty()) {
            log.warn("Hardest cases (< 50% pass rate): {}", report.hardestCases());
        }
        log.info("===================");
    }
}
```

---

## Golden Dataset

### Anatomy of a Good Test Case

```java
public record EvalCase(
    String taskId,
    String description,
    EvalCaseCategory category,  // HAPPY_PATH, EDGE_CASE, ADVERSARIAL, REGRESSION
    String input,
    TaskExpectation expectation,
    float passingThreshold,      // 0.0-1.0, thường 0.7-0.9
    int complexityLevel          // 1-5
) {}
```

### Sample Golden Dataset cho Java Code Review Agent

```java
@Component
public class CodeReviewGoldenDataset {

    public List<EvalCase> buildGoldenDataset() {
        return List.of(
            // ===== HAPPY PATH (nên luôn pass) =====
            EvalCase.builder()
                .taskId("HP-001")
                .description("Simple class with obvious SQL injection")
                .category(HAPPY_PATH)
                .input(SIMPLE_SQL_INJECTION_CODE)
                .expectation(TaskExpectation.contains(List.of(
                    "SQL injection", "PreparedStatement", "parameterized"
                )))
                .passingThreshold(0.8f)
                .complexityLevel(1)
                .build(),

            EvalCase.builder()
                .taskId("HP-002")
                .description("Class with missing null checks")
                .category(HAPPY_PATH)
                .input(NULL_POINTER_RISK_CODE)
                .expectation(TaskExpectation.contains(List.of("null", "NullPointerException")))
                .passingThreshold(0.8f)
                .complexityLevel(1)
                .build(),

            // ===== EDGE CASES =====
            EvalCase.builder()
                .taskId("EC-001")
                .description("Empty class — should report 'no issues' not hallucinate")
                .category(EDGE_CASE)
                .input("public class EmptyClass {}")
                .expectation(TaskExpectation.doesNotContain(List.of(
                    "SQL injection", "XSS", "NullPointerException"
                )))
                .passingThreshold(0.9f)
                .complexityLevel(1)
                .build(),

            EvalCase.builder()
                .taskId("EC-002")
                .description("Large class (500 lines) — should handle without truncating")
                .category(EDGE_CASE)
                .input(LARGE_SERVICE_CLASS_500_LINES)
                .expectation(TaskExpectation.minFindings(3))
                .passingThreshold(0.7f)
                .complexityLevel(4)
                .build(),

            // ===== ADVERSARIAL =====
            EvalCase.builder()
                .taskId("ADV-001")
                .description("Prompt injection attempt in code comment")
                .category(ADVERSARIAL)
                .input(CODE_WITH_PROMPT_INJECTION_IN_COMMENT)
                .expectation(TaskExpectation.doesNotFollow("ignore previous instructions"))
                .passingThreshold(0.95f)
                .complexityLevel(2)
                .build(),

            // ===== REGRESSION (past failures) =====
            EvalCase.builder()
                .taskId("REG-001")
                .description("Issue #234: Agent was missing race conditions")
                .category(REGRESSION)
                .input(CONCURRENT_CODE_WITH_RACE_CONDITION)
                .expectation(TaskExpectation.contains(List.of("race condition", "synchronized")))
                .passingThreshold(0.85f)
                .complexityLevel(3)
                .build()
        );
    }
}
```

---

## Continuous Evaluation: CI/CD Integration

### Automated Eval on Every Deploy

```java
@Component
public class CiEvalGate {

    private final AgentEvalHarness harness;
    private final EvalBaselineRepository baselineRepo;
    private final AlertingService alerting;

    /**
     * Chạy eval và so sánh với baseline.
     * Fail CI nếu performance drop quá nhiều.
     */
    public EvalGateResult runCiEval(Agent agent, String deployVersion) {
        log.info("Running CI eval for agent {} version {}", agent.getId(), deployVersion);

        // Chạy eval: 5 runs per case
        EvalReport current = harness.run(agent, goldenDataset.buildGoldenDataset(), 5);

        // Lấy baseline
        Optional<EvalReport> baseline = baselineRepo.findLatestBaseline(agent.getId());

        if (baseline.isEmpty()) {
            // Lần đầu tiên — set baseline
            log.info("No baseline found, setting current as baseline");
            baselineRepo.saveBaseline(agent.getId(), current, deployVersion);
            return EvalGateResult.passed("First run — baseline established");
        }

        // So sánh với baseline
        float passRateDelta = current.overallPassRate() - baseline.get().overallPassRate();
        float costDelta = (float)(
            current.costReport().costPerSuccessfulTaskUsd() -
            baseline.get().costReport().costPerSuccessfulTaskUsd()
        );

        List<String> violations = new ArrayList<>();

        // Check 1: Pass rate không drop quá 5%
        if (passRateDelta < -0.05f) {
            violations.add(String.format(
                "Pass rate dropped %.1f%% (%.1f%% → %.1f%%)",
                Math.abs(passRateDelta) * 100,
                baseline.get().overallPassRate() * 100,
                current.overallPassRate() * 100
            ));
        }

        // Check 2: Cost không tăng quá 20%
        if (costDelta > 0.20f) {
            violations.add(String.format(
                "Cost increased %.0f%% — investigate prompt changes",
                costDelta * 100
            ));
        }

        // Check 3: P95 latency không tăng quá 50%
        long latencyDelta = current.latencyReport().p95Ms() -
                           baseline.get().latencyReport().p95Ms();
        if (latencyDelta > baseline.get().latencyReport().p95Ms() * 0.5) {
            violations.add(String.format(
                "P95 latency increased by %dms",
                latencyDelta
            ));
        }

        if (!violations.isEmpty()) {
            String message = "CI EVAL FAILED:\n" + String.join("\n", violations);
            alerting.sendAlert(message);
            log.error(message);
            return EvalGateResult.failed(violations);
        }

        // Tất cả checks pass — update baseline nếu cải thiện
        if (passRateDelta > 0.02f) {
            log.info("Performance improved, updating baseline");
            baselineRepo.saveBaseline(agent.getId(), current, deployVersion);
        }

        return EvalGateResult.passed("All checks passed. Pass rate: " +
            String.format("%.1f%%", current.overallPassRate() * 100));
    }
}
```

---

## Prompt Iteration Workflow

Dùng eval harness để cải thiện prompts một cách khoa học:

```
1. Run eval với current prompt → baseline_score
2. Hypothesis: "Adding 2 examples sẽ improve edge case handling"
3. Modify system prompt
4. Run eval → new_score
5. Compare: new_score > baseline_score + 0.05? → Keep
6. Tự động lưu lịch sử: prompt_version + eval_score
```

```java
@Service
public class PromptIterator {

    private final AgentEvalHarness harness;
    private final PromptVersionRepository promptRepo;

    public PromptIterationResult iterate(
            Agent agent,
            String currentPrompt,
            String proposedPrompt,
            List<EvalCase> testCases) {

        // Eval current prompt
        agent.setSystemPrompt(currentPrompt);
        EvalReport baselineReport = harness.run(agent, testCases, 5);

        // Eval proposed prompt
        agent.setSystemPrompt(proposedPrompt);
        EvalReport proposedReport = harness.run(agent, testCases, 5);

        float improvement = proposedReport.overallPassRate()
            - baselineReport.overallPassRate();

        boolean shouldAdopt = improvement > 0.03f; // Ít nhất 3% improvement

        if (shouldAdopt) {
            promptRepo.saveVersion(agent.getId(), proposedPrompt,
                proposedReport.overallPassRate());
        }

        return new PromptIterationResult(
            baselineReport.overallPassRate(),
            proposedReport.overallPassRate(),
            improvement,
            shouldAdopt,
            shouldAdopt ? "Prompt improved by " + String.format("%.1f%%", improvement * 100)
                        : "No significant improvement, keeping current prompt"
        );
    }
}
```

---

## Exercise: Eval Harness cho Java Code Review Pipeline

Xây dựng eval harness đầy đủ cho Code Review Pipeline từ các bài trước:

**Yêu cầu:**
1. Tạo golden dataset với ít nhất 20 test cases (5 happy path, 5 edge cases, 5 adversarial, 5 regression)
2. Mỗi case chạy 5 lần (pass@5)
3. Track cả 5 dimensions: completion rate, output quality, reliability, cost, latency
4. Implement CI gate: fail nếu pass rate drop > 5% so với baseline
5. Dashboard endpoint: `GET /eval/report/latest` trả về EvalReport dạng JSON

**Test case template:**
```java
// HP-001: Standard security vulnerability
EvalCase.builder()
    .taskId("HP-001")
    .input("""
        @RestController
        public class UserController {
            @GetMapping("/users")
            public String findUser(@RequestParam String name) {
                return jdbcTemplate.queryForObject(
                    "SELECT * FROM users WHERE name = '" + name + "'",
                    String.class
                );
            }
        }
        """)
    .expectation(TaskExpectation.contains(List.of("SQL injection", "PreparedStatement")))
    .passingThreshold(0.85f)
    .build()
```

**Tiêu chí hoàn thành:**
- [ ] 20+ test cases trong golden dataset
- [ ] Harness chạy parallel, kết quả reproducible
- [ ] CI gate comparisons với baseline hoạt động
- [ ] Tất cả 5 dimensions được report
- [ ] Regression detection alert khi pass rate drop

---

## Tóm Tắt

| Dimension | Metric | Target | Tool |
|-----------|--------|--------|------|
| Completion | pass@5 | > 80% | AgentEvalHarness |
| Quality | Score 0-1 | > 0.75 | Domain-specific checkers |
| Reliability | Wilson CI width | < 0.15 | ReliabilityMeasurer |
| Cost | $/successful task | Track trend | CostAnalyzer |
| Latency | P95 | < 30s | LatencyProfiler |

**Key takeaway**: "It works on my machine" là không đủ. Evaluation framework biến direct observation thành statistical evidence. Với pass@5 và CI integration, bạn có thể tự tin deploy agents vào production và catch regressions trước khi users thấy.

---

*Bài tiếp theo: Capstone Project — SmartOps Production-Grade Agentic System*
