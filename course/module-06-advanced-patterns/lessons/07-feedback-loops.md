# Lesson 07: Feedback Loops & Self-Correction — Agents That Improve

> **Module 06 — Advanced Agentic Patterns**
> Prerequisite: Lessons 01-06 (ReAct, Plan-and-Execute, Memory, Ingestion, Context Management)
> Thời lượng ước tính: 90 phút

---

## Tại Sao Agents Cần Feedback Loops?

Trong thực tế, không có agent nào hoàn hảo ngay từ lần đầu. Code được generate có thể không compile. Document được viết ra có thể thiếu thông tin quan trọng. SQL query có thể trả về kết quả sai. Nếu agent chỉ generate một lần rồi trả kết quả, người dùng sẽ nhận được output chất lượng thấp mà không có cơ hội cải thiện.

**Feedback loop** là cơ chế cho phép agent nhận biết khi output của mình chưa đạt yêu cầu và tự điều chỉnh. Thay vì "generate và hy vọng", agent sẽ "generate, kiểm tra, cải thiện, lặp lại cho đến khi đạt tiêu chuẩn".

Có ba lý do kỹ thuật chính khiến feedback loops là bắt buộc trong production systems:

1. **LLMs không deterministic** — cùng một prompt có thể cho ra output khác nhau mỗi lần, và đôi khi output đó có lỗi.
2. **Task complexity vượt qua context window** — với tasks phức tạp, model cần nhiều vòng lặp để "suy nghĩ đủ sâu".
3. **External constraints không thể encode vào prompt** — compiler rules, business logic, database constraints — những thứ này chỉ có thể kiểm tra bằng cách thực thi thực sự.

---

## Tổng Quan 4 Loại Feedback Loop

```
┌─────────────────────────────────────────────────────────┐
│                   FEEDBACK LOOP TAXONOMY                  │
├─────────────────┬───────────────────────────────────────┤
│ Type 1          │ Reflection (Self-Critique)             │
│                 │ Agent đánh giá output của chính mình  │
├─────────────────┼───────────────────────────────────────┤
│ Type 2          │ External Validation                    │
│                 │ Compiler, tests, linter kiểm tra       │
├─────────────────┼───────────────────────────────────────┤
│ Type 3          │ Human-in-the-Loop (HITL)               │
│                 │ Con người review và approve            │
├─────────────────┼───────────────────────────────────────┤
│ Type 4          │ Reinforcement from Feedback            │
│                 │ Cải thiện dài hạn từ collected data   │
└─────────────────┴───────────────────────────────────────┘
```

---

## Type 1: Reflection (Self-Critique)

### Khái Niệm

Reflection là kỹ thuật đơn giản nhất nhưng hiệu quả đáng ngạc nhiên: sau khi generate output, agent được yêu cầu **đóng vai người review** và phê bình output của chính mình. Sau đó, dựa trên critique, agent generate lại phiên bản cải thiện hơn.

```
Generate → Critique own output → Identify weaknesses → Regenerate → Repeat N times
```

Cơ chế này hoạt động vì LLMs thường "biết" khi một output không tốt — chúng chỉ không tự động apply kiến thức đó trừ khi được yêu cầu. Khi bạn explicitly hỏi "output này có vấn đề gì không?", model sẽ phát hiện ra những lỗi mà nó không nhận ra trong lần generate đầu tiên.

### Reflection Prompt Template

```
You are reviewing the following output that was generated for this task:

TASK: {original_task}

PREVIOUS OUTPUT:
{previous_output}

Please review this output critically. Evaluate:
1. ACCURACY: Are all facts and logic correct? Any errors?
2. COMPLETENESS: Is anything important missing?
3. CLARITY: Is it clear and easy to understand?
4. CORRECTNESS: Does it fully address the original task?

If the output has no significant issues, respond with exactly: "ACCEPTABLE"
Otherwise, list specific issues and provide an improved version.
```

Lưu ý stopping condition quan trọng: nếu critique trả về "ACCEPTABLE", dừng lại ngay. Không nên phản ánh vô hạn — điều đó chỉ tốn tokens mà không cải thiện chất lượng.

### Java Implementation

```java
@Service
public class ReflectiveGenerationService {

    private final AnthropicClient claude;
    private static final int DEFAULT_MAX_REFLECTIONS = 3;

    public ReflectiveGenerationService(AnthropicClient claude) {
        this.claude = claude;
    }

    public String generateWithReflection(String task, int maxReflections) {
        // Bước 1: Generate output ban đầu
        String output = generateInitial(task);
        log.info("Initial generation complete, starting reflection loop");

        for (int i = 0; i < maxReflections; i++) {
            log.debug("Reflection round {}/{}", i + 1, maxReflections);

            // Bước 2: Tự critique output
            String critique = critique(output, task);

            // Bước 3: Kiểm tra stopping condition
            if (isAcceptable(critique)) {
                log.info("Output accepted after {} reflection(s)", i);
                break;
            }

            // Bước 4: Regenerate dựa trên critique
            log.debug("Issues found, regenerating. Critique: {}", critique);
            output = regenerate(task, output, critique);
        }

        return output;
    }

    private String generateInitial(String task) {
        return claude.complete("""
            Complete the following task to the best of your ability:
            
            %s
            """.formatted(task));
    }

    private String critique(String output, String originalTask) {
        return claude.complete("""
            You are reviewing the following output that was generated for this task:
            
            TASK: %s
            
            PREVIOUS OUTPUT:
            %s
            
            Review this output critically. Evaluate accuracy, completeness, clarity,
            and correctness. If no significant issues exist, respond with exactly: ACCEPTABLE
            Otherwise, list specific issues found.
            """.formatted(originalTask, output));
    }

    private boolean isAcceptable(String critique) {
        // Normalize và check stopping condition
        return critique.trim().equalsIgnoreCase("ACCEPTABLE")
            || critique.toLowerCase().contains("no significant issues");
    }

    private String regenerate(String task, String previousOutput, String critique) {
        return claude.complete("""
            The following output was generated for a task, but has issues that need fixing:
            
            ORIGINAL TASK: %s
            
            PREVIOUS OUTPUT (with issues):
            %s
            
            CRITIQUE (issues to fix):
            %s
            
            Please provide an improved version that addresses all issues mentioned.
            """.formatted(task, previousOutput, critique));
    }

    // Overload với default max reflections
    public String generateWithReflection(String task) {
        return generateWithReflection(task, DEFAULT_MAX_REFLECTIONS);
    }
}
```

### Tracking Reflection Quality

Để đo lường xem reflection có thực sự cải thiện output không, hãy track reflection depth:

```java
public record ReflectionTrace(
    String initialOutput,
    List<ReflectionRound> rounds,
    String finalOutput,
    int totalRounds,
    int totalTokensUsed
) {
    public record ReflectionRound(
        String critique,
        String improvedOutput,
        boolean wasAccepted
    ) {}
}
```

### Khi Nào Dùng Reflection

| Use Case | Reflection Useful? | Notes |
|----------|-------------------|-------|
| Code generation | Cao | Detect logic errors, missing edge cases |
| Document writing | Cao | Improve clarity, completeness |
| Complex analysis | Cao | Catch reasoning errors |
| Simple Q&A | Thấp | Overkill, tốn tokens không cần thiết |
| Factual lookup | Không | Reflection không thêm thông tin mới |

**Giới hạn thực tế**: maxReflections = 3 là con số tốt cho production. Sau 3 vòng, improvements thường giảm dần trong khi cost tăng linear.

---

## Type 2: External Validation

### Khái Niệm

Reflection là agent tự đánh giá chính mình — về cơ bản là "chủ quan". External Validation là agent nhận feedback từ **hệ thống bên ngoài khách quan**: compiler, test runner, linter, JSON schema validator, database.

Đây là feedback loop mạnh nhất cho code generation vì compiler không bao giờ nói dối: code hoặc compile được hoặc không.

```
Generate → Validate (compiler, tests, linter) → If fail: fix with error context → Retry
```

Điểm mấu chốt: khi re-attempt, **luôn đưa error message vào context**. Claude cần biết cụ thể lỗi là gì để sửa đúng chỗ.

### Java Code Generation với Compilation Check

```java
@Service
public class SelfCorrectingCodeGenerator {

    private final AnthropicClient claude;
    private final JavaCompilerService compiler;
    private static final int MAX_ATTEMPTS = 5;

    public String generateAndCompile(String spec) {
        String lastError = null;

        for (int attempt = 1; attempt <= MAX_ATTEMPTS; attempt++) {
            log.info("Code generation attempt {}/{}", attempt, MAX_ATTEMPTS);

            // Generate code, đưa error từ lần trước vào context nếu có
            String code = generateCode(spec, lastError);

            // Validate bằng compiler thực
            CompileResult result = compiler.compile(code);

            if (result.success()) {
                log.info("Code compiled successfully on attempt {}", attempt);
                return code;
            }

            // Lưu error để đưa vào next attempt
            lastError = result.errorMessage();
            log.warn("Compile failed (attempt {}): {}", attempt, lastError);
        }

        throw new GenerationFailedException(
            "Could not generate compilable code after " + MAX_ATTEMPTS + " attempts. " +
            "Last error: " + lastError
        );
    }

    private String generateCode(String spec, String previousError) {
        if (previousError == null) {
            // First attempt — clean prompt
            return claude.complete("""
                Generate Java code that implements the following specification:
                
                %s
                
                Return only the Java code, no explanations.
                """.formatted(spec));
        } else {
            // Subsequent attempts — include error context
            return claude.complete("""
                The previously generated Java code had a compilation error.
                Please fix it.
                
                SPECIFICATION:
                %s
                
                COMPILATION ERROR:
                %s
                
                Generate corrected Java code that compiles successfully.
                Return only the Java code, no explanations.
                """.formatted(spec, previousError));
        }
    }
}
```

### Multi-Stage External Validation

Với production code, compile là điều kiện cần nhưng chưa đủ. Cần thêm test execution:

```java
@Service
public class MultiStageValidator {

    private final JavaCompilerService compiler;
    private final TestRunner testRunner;
    private final CheckstyleRunner linter;

    public ValidationResult validateFully(String code, List<String> testCases) {
        // Stage 1: Compilation
        CompileResult compileResult = compiler.compile(code);
        if (!compileResult.success()) {
            return ValidationResult.failed("COMPILE_ERROR", compileResult.errorMessage());
        }

        // Stage 2: Test execution
        TestResult testResult = testRunner.runTests(code, testCases);
        if (!testResult.allPassed()) {
            String failedTests = testResult.failures().stream()
                .map(f -> f.testName() + ": " + f.errorMessage())
                .collect(Collectors.joining("\n"));
            return ValidationResult.failed("TEST_FAILURE", failedTests);
        }

        // Stage 3: Linting (optional but recommended)
        LintResult lintResult = linter.check(code);
        if (lintResult.hasCriticalIssues()) {
            return ValidationResult.failed("LINT_ERROR", lintResult.criticalIssues());
        }

        return ValidationResult.success();
    }
}
```

### Các External Validator Phổ Biến

| Validator | Use Case | Java Implementation |
|-----------|----------|---------------------|
| `javac` | Java compilation | `javax.tools.JavaCompiler` |
| JUnit runner | Unit test execution | `JUnitPlatform.run()` |
| Checkstyle | Code style | `CheckstyleRunner` |
| JSON Schema | JSON validation | `NetworkNT JsonSchemaFactory` |
| OpenAPI validator | API response format | `swagger-request-validator` |
| SQL parser | SQL syntax check | `JSQLParser` |
| `pg explain` | Query plan analysis | JDBC + EXPLAIN ANALYZE |

### SQL Query Validation Example

```java
@Service
public class SqlQueryValidator {

    private final DataSource dataSource;

    public SqlValidationResult validateQuery(String sqlQuery) {
        // Bước 1: Parse syntax
        try {
            CCJSqlParserUtil.parse(sqlQuery);
        } catch (JSQLParserException e) {
            return SqlValidationResult.syntaxError(e.getMessage());
        }

        // Bước 2: Explain plan (không execute thực sự — safe)
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(
                 "EXPLAIN " + sqlQuery)) {
            ResultSet rs = stmt.executeQuery();
            String plan = extractPlan(rs);
            return SqlValidationResult.success(plan);
        } catch (SQLException e) {
            return SqlValidationResult.executionError(e.getMessage());
        }
    }
}
```

---

## Type 3: Human-in-the-Loop (HITL)

### Khi Nào HITL Là Bắt Buộc

Không phải mọi quyết định đều nên để agent tự động hóa hoàn toàn. Có những hành động mà hậu quả của một sai lầm là không thể đảo ngược hoặc quá nghiêm trọng:

```
Generate → Present to human → Human approves/corrects → Agent continues
```

**Danh sách BẮT BUỘC HITL trong production:**

- Database migrations (đặc biệt là DROP, ALTER, DELETE không có WHERE)
- Infrastructure changes (scale down, terminate instances)
- Financial operations (transfers, refunds, charge)
- Sending emails/notifications tới users thực
- Thay đổi security configurations (firewall rules, IAM policies)
- Deploys to production environment

Nguyên tắc đơn giản: **nếu action không thể rollback trong < 5 phút, cần HITL.**

### Approval Gate với Slack Integration

```java
@Service
public class HumanApprovalGate {

    private final SlackClient slackClient;
    private final ApprovalRepository approvalRepo;
    private static final String APPROVAL_CHANNEL = "#agent-approvals";
    private static final Duration APPROVAL_TIMEOUT = Duration.ofMinutes(30);

    public ApprovalResult requestApproval(
            String agentId,
            String actionType,
            String actionDescription,
            String context) {

        // Tạo approval request với unique ID
        String approvalId = UUID.randomUUID().toString();
        Instant expiresAt = Instant.now().plus(APPROVAL_TIMEOUT);

        // Lưu vào database để track
        approvalRepo.save(new ApprovalRequest(
            approvalId, agentId, actionType, actionDescription,
            context, ApprovalStatus.PENDING, expiresAt
        ));

        // Gửi message tới Slack với approve/reject buttons
        slackClient.sendMessage(
            APPROVAL_CHANNEL,
            buildApprovalMessage(approvalId, agentId, actionType, actionDescription, context)
        );

        log.info("Approval requested [{}] for agent {} action: {}", approvalId, agentId, actionType);

        // Poll database cho đến khi approved/rejected hoặc timeout
        return waitForDecision(approvalId, expiresAt);
    }

    private SlackMessage buildApprovalMessage(
            String approvalId, String agentId,
            String actionType, String description, String context) {

        return SlackMessage.builder()
            .text(String.format(":robot_face: Agent `%s` is requesting approval", agentId))
            .block(SectionBlock.builder()
                .text("*Action Type:* " + actionType + "\n" +
                      "*Description:* " + description)
                .build())
            .block(ContextBlock.builder()
                .text("*Context:*\n```" + context + "```")
                .build())
            .block(ActionsBlock.builder()
                .element(ButtonElement.builder()
                    .text("Approve")
                    .style("primary")
                    .actionId("approve_" + approvalId)
                    .build())
                .element(ButtonElement.builder()
                    .text("Reject")
                    .style("danger")
                    .actionId("reject_" + approvalId)
                    .build())
                .build())
            .build();
    }

    private ApprovalResult waitForDecision(String approvalId, Instant expiresAt) {
        while (Instant.now().isBefore(expiresAt)) {
            ApprovalRequest request = approvalRepo.findById(approvalId).orElseThrow();

            if (request.status() == ApprovalStatus.APPROVED) {
                log.info("Approval [{}] approved by {}", approvalId, request.decidedBy());
                return ApprovalResult.approved(request.decidedBy(), request.comment());
            }

            if (request.status() == ApprovalStatus.REJECTED) {
                log.info("Approval [{}] rejected by {}", approvalId, request.decidedBy());
                return ApprovalResult.rejected(request.decidedBy(), request.comment());
            }

            // Chưa có quyết định, chờ 5 giây rồi check lại
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return ApprovalResult.error("Interrupted while waiting for approval");
            }
        }

        // Timeout: tự động reject (safer default)
        approvalRepo.updateStatus(approvalId, ApprovalStatus.TIMED_OUT);
        log.warn("Approval [{}] timed out after {}", approvalId, APPROVAL_TIMEOUT);
        return ApprovalResult.timedOut();
    }
}
```

### Non-Blocking HITL

Trong pattern blocking ở trên, agent bị "đơ" trong khi chờ. Đây là vấn đề khi timeout dài (30 phút). Non-blocking HITL cho phép agent tiếp tục làm việc khác:

```java
@Service
public class NonBlockingHITL {

    private final HumanApprovalGate approvalGate;
    private final AgentTaskQueue taskQueue;

    public CompletableFuture<ApprovalResult> requestApprovalAsync(
            String agentId,
            PendingAction action) {

        // Gửi approval request
        String approvalId = approvalGate.submitRequest(agentId, action);

        // Trả về future — agent không bị block
        return CompletableFuture.supplyAsync(() -> {
            // Background thread poll cho đến khi có kết quả
            return approvalGate.waitForDecision(approvalId, Duration.ofMinutes(30));
        }, asyncExecutor);
    }

    public void processWithHITL(String agentId, List<Action> actions) {
        List<CompletableFuture<ApprovalResult>> pendingApprovals = new ArrayList<>();

        for (Action action : actions) {
            if (action.requiresApproval()) {
                // Submit async, không chờ
                CompletableFuture<ApprovalResult> future =
                    requestApprovalAsync(agentId, action);
                pendingApprovals.add(future.thenApply(result -> {
                    if (result.approved()) {
                        action.execute(); // Thực thi sau khi được approve
                    }
                    return result;
                }));
            } else {
                // Safe actions: execute ngay
                action.execute();
            }
        }

        // Chờ tất cả approvals hoàn thành (blocking chỉ ở đây)
        CompletableFuture.allOf(pendingApprovals.toArray(new CompletableFuture[0])).join();
    }
}
```

### Audit Trail

Mọi HITL interaction phải được log đầy đủ cho compliance:

```java
@Entity
@Table(name = "hitl_audit_log")
public class HitlAuditLog {
    @Id
    private String approvalId;
    private String agentId;
    private String actionType;
    private String actionDescription;
    @Column(columnDefinition = "TEXT")
    private String fullContext;
    private String requestedAt;
    private String decidedAt;
    private String decidedBy;        // Who approved/rejected
    private String decision;         // APPROVED, REJECTED, TIMED_OUT
    private String decisionComment;
    private boolean actionExecuted;
    private String executionResult;
}
```

---

## Type 4: Reinforcement from Feedback

### Long-Term Improvement

Ba loại feedback loop trên hoạt động trong scope của một task. Type 4 hoạt động ở cấp độ **hệ thống theo thời gian**: thu thập feedback từ nhiều task, phân tích patterns, cải thiện system prompts và examples để agent ngày càng tốt hơn.

**Quan trọng**: Đây **KHÔNG** phải fine-tuning model. Đây là prompt engineering và system optimization dựa trên data.

```
Collect Feedback → Analyze Patterns → Update System Prompts/Examples → Measure Improvement
```

### FeedbackCollector Service

```java
@Service
public class AgentFeedbackCollector {

    private final FeedbackRepository feedbackRepo;

    // User explicitly rate output
    public void collectRating(String taskId, String agentId,
                               int rating, String comment) {
        feedbackRepo.save(new AgentFeedback(
            taskId, agentId, FeedbackType.EXPLICIT_RATING,
            rating, comment, Instant.now()
        ));
    }

    // User corrects agent output
    public void collectCorrection(String taskId, String agentId,
                                   String originalOutput, String correctedOutput) {
        feedbackRepo.save(new AgentFeedback(
            taskId, agentId, FeedbackType.CORRECTION,
            originalOutput, correctedOutput, Instant.now()
        ));
        log.info("Correction collected for task {}: {} chars changed",
            taskId, editDistance(originalOutput, correctedOutput));
    }

    // Implicit: task completion time, retry count
    public void collectUsageMetrics(String taskId, String agentId,
                                     int retryCount, Duration completionTime,
                                     boolean taskCompleted) {
        feedbackRepo.save(new AgentFeedback(
            taskId, agentId, FeedbackType.USAGE_METRICS,
            retryCount, completionTime, taskCompleted, Instant.now()
        ));
    }
}
```

### FeedbackAnalyzer

```java
@Service
public class FeedbackAnalyzer {

    private final FeedbackRepository feedbackRepo;

    public FeedbackInsights analyze(String agentId, LocalDate from, LocalDate to) {
        List<AgentFeedback> feedbacks = feedbackRepo.findByAgentAndDateRange(agentId, from, to);

        // Task types có nhiều corrections nhất
        Map<String, Long> correctionsByTaskType = feedbacks.stream()
            .filter(f -> f.type() == FeedbackType.CORRECTION)
            .collect(Collectors.groupingBy(
                AgentFeedback::taskType,
                Collectors.counting()
            ));

        // Average rating theo task type
        Map<String, Double> avgRatingByTaskType = feedbacks.stream()
            .filter(f -> f.type() == FeedbackType.EXPLICIT_RATING)
            .collect(Collectors.groupingBy(
                AgentFeedback::taskType,
                Collectors.averagingDouble(AgentFeedback::rating)
            ));

        // Retry patterns — task types có nhiều retries nhất
        Map<String, Double> avgRetryByTaskType = feedbacks.stream()
            .filter(f -> f.type() == FeedbackType.USAGE_METRICS)
            .collect(Collectors.groupingBy(
                AgentFeedback::taskType,
                Collectors.averagingDouble(f -> f.retryCount())
            ));

        return new FeedbackInsights(
            correctionsByTaskType,
            avgRatingByTaskType,
            avgRetryByTaskType,
            generateRecommendations(correctionsByTaskType, avgRatingByTaskType)
        );
    }

    private List<String> generateRecommendations(
            Map<String, Long> corrections,
            Map<String, Double> ratings) {

        List<String> recommendations = new ArrayList<>();

        corrections.entrySet().stream()
            .filter(e -> e.getValue() > 10) // Hơn 10 corrections
            .forEach(e -> recommendations.add(
                String.format("Task type '%s' has %d corrections — " +
                    "consider adding more examples to system prompt",
                    e.getKey(), e.getValue())
            ));

        ratings.entrySet().stream()
            .filter(e -> e.getValue() < 3.5) // Rating dưới 3.5/5
            .forEach(e -> recommendations.add(
                String.format("Task type '%s' has low rating (%.1f/5) — " +
                    "review and improve output format",
                    e.getKey(), e.getValue())
            ));

        return recommendations;
    }
}
```

---

## Kết Hợp Feedback Loops: Practical Recommendations

Trong thực tế, bạn hiếm khi dùng chỉ một loại feedback loop. Đây là recommendations:

```
┌─────────────────────────────────────────────────────────────┐
│              COMBINING FEEDBACK LOOPS                        │
├──────────────────────┬──────────────────────────────────────┤
│ Code Generation      │ External Validation (compile + test) │
│                      │ + Reflection nếu tests fail          │
├──────────────────────┼──────────────────────────────────────┤
│ Document Writing     │ Reflection (2-3 rounds)              │
│                      │ + HITL cho important docs            │
├──────────────────────┼──────────────────────────────────────┤
│ Infrastructure Ops   │ HITL LUÔN LUÔN, không exception      │
├──────────────────────┼──────────────────────────────────────┤
│ Data Analysis        │ Reflection (sanity checks)           │
│                      │ + Human review nếu anomalies         │
├──────────────────────┼──────────────────────────────────────┤
│ All task types       │ Type 4 (collect feedback long-term)  │
└──────────────────────┴──────────────────────────────────────┘
```

### Integrated Feedback Pipeline

```java
@Service
public class FeedbackAwarePipeline {

    private final ReflectiveGenerationService reflection;
    private final SelfCorrectingCodeGenerator externalValidation;
    private final HumanApprovalGate hitl;
    private final AgentFeedbackCollector feedbackCollector;

    public PipelineResult execute(AgentTask task) {
        String agentId = task.agentId();
        String taskId = task.taskId();
        Instant startTime = Instant.now();

        try {
            String output;

            switch (task.type()) {
                case CODE_GENERATION -> {
                    // External validation first, reflection nếu cần
                    output = externalValidation.generateAndCompile(task.spec());
                }
                case DOCUMENT_WRITING -> {
                    // Reflection, sau đó HITL nếu document quan trọng
                    output = reflection.generateWithReflection(task.spec(), 3);
                    if (task.requiresApproval()) {
                        ApprovalResult approval = hitl.requestApproval(
                            agentId, "PUBLISH_DOCUMENT", output, task.context()
                        );
                        if (!approval.approved()) {
                            return PipelineResult.rejected(approval.reason());
                        }
                    }
                }
                case INFRASTRUCTURE_CHANGE -> {
                    // HITL LUÔN, không reflection (human must see it)
                    output = generateInfrastructurePlan(task.spec());
                    ApprovalResult approval = hitl.requestApproval(
                        agentId, "INFRASTRUCTURE_CHANGE", output, task.context()
                    );
                    if (!approval.approved()) {
                        return PipelineResult.rejected(approval.reason());
                    }
                }
                default -> output = reflection.generateWithReflection(task.spec(), 2);
            }

            // Collect metrics for Type 4 improvement
            feedbackCollector.collectUsageMetrics(
                taskId, agentId, 1, Duration.between(startTime, Instant.now()), true
            );

            return PipelineResult.success(output);

        } catch (Exception e) {
            feedbackCollector.collectUsageMetrics(
                taskId, agentId, 1, Duration.between(startTime, Instant.now()), false
            );
            return PipelineResult.failed(e.getMessage());
        }
    }
}
```

---

## Exercise: Self-Correcting Java Code RAG Agent

Trong bài trước, bạn đã xây dựng Java Codebase RAG agent. Bây giờ hãy thêm self-correction:

**Yêu cầu:**
1. Khi Claude generate code từ RAG context, chạy compilation check
2. Nếu compile fail, feed error message back cho Claude cùng với original context
3. Thử tối đa 3 lần
4. Log mỗi attempt với: attempt number, error message, time taken
5. Nếu 3 lần đều fail, trả về partial result với error explanation

**Hint:**
```java
public class RagCodeGenerator {
    // Combine RAG retrieval + external validation
    public String generateFromCodebase(String question) {
        // Step 1: Retrieve relevant code chunks
        List<CodeChunk> context = vectorStore.search(question, 5);
        
        // Step 2: Generate with self-correction loop
        String lastError = null;
        for (int attempt = 0; attempt < 3; attempt++) {
            String code = generateWithContext(question, context, lastError);
            CompileResult result = compiler.compile(code);
            if (result.success()) return code;
            lastError = result.errorMessage();
        }
        
        return generateFallbackExplanation(question, context, lastError);
    }
}
```

**Tiêu chí hoàn thành:**
- [ ] Compilation check hoạt động
- [ ] Error message được đưa vào context của lần retry
- [ ] Tối đa 3 attempts được enforce
- [ ] Logs đầy đủ cho debugging
- [ ] Fallback khi tất cả attempts đều fail

---

## Tóm Tắt

| Feedback Type | Cơ chế | Tốt nhất cho | Chi phí |
|--------------|--------|--------------|---------|
| Reflection | LLM tự critique | Documents, analysis | Trung bình (3x tokens) |
| External Validation | Compiler/tests | Code generation | Thấp (execution only) |
| HITL | Human review | High-stakes actions | Cao (human time) |
| Reinforcement | Long-term collection | System improvement | Cao (upfront investment) |

**Key takeaway**: Feedback loops là sự khác biệt giữa "agent demo" và "production agent". Không có feedback loop, agent của bạn sẽ thỉnh thoảng fail theo những cách bạn không thể dự đoán. Với feedback loops, failures trở thành inputs cho improvement.

---

*Bài tiếp theo: Lesson 08 — Deduplication Patterns in Multi-Agent Pipelines*
