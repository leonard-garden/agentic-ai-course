# Lesson 02: Planning Patterns — Plan Before You Act

> **Module 06 — Advanced Agentic Patterns**
> **Thời gian**: ~ 4 giờ | **Difficulty**: Intermediate-Advanced

---

## Mục tiêu bài học

Sau bài này bạn sẽ:
- Hiểu tại sao planning là yếu tố quyết định thành bại của complex agentic tasks
- Nắm vững 4 planning patterns: CoT, Plan-and-Execute, ReWOO, Tree of Thoughts
- Biết cách chọn pattern phù hợp cho từng loại task
- Implement `PlannerAgent` trong Java với strategy pattern để switch linh hoạt giữa các modes

---

## 1. Tại sao Planning matters?

Hãy thử giao cho một intern Java developer task này mà không giải thích gì thêm:

> "Migrate our monolith Spring Boot app to microservices."

Nếu họ bắt đầu ngay lập tức không suy nghĩ — tạo random services, move code tùy tiện — kết quả sẽ là thảm họa. Nhưng nếu họ dành 30 phút để:
1. Hiểu current architecture
2. Identify domain boundaries
3. Plan migration order (leaf services first)
4. Define rollback strategy

...thì khả năng thành công cao hơn nhiều.

**LLM agents cũng vậy.** Không có planning, agent sẽ:
- Act on first impulse, không consider alternatives
- Bị mắc kẹt ở dead ends mà không có cách thoát
- Làm các việc theo thứ tự sai (dependency errors)
- Waste tokens làm lại việc đã làm

**Với planning, agent có thể:**
- Decompose task phức tạp thành subtasks manageable
- Identify dependencies và thứ tự thực hiện
- Allocate resources phù hợp (tools, steps)
- Detect khi nào cần replan

---

## 2. Tổng quan 4 Planning Patterns

```
Complexity / Adaptability
        ▲
        │                              ┌─────────────────┐
        │                              │ Tree of Thoughts │
        │                              │ (Multiple paths, │
        │                              │  branch & prune) │
        │               ┌─────────────┴─────────────┐
        │               │        ReWOO               │
        │               │  (Upfront plan, no mid-    │
        │               │   step observations)       │
        │  ┌────────────┴────────────┐
        │  │    Plan-and-Execute     │
        │  │ (Sequential plan +      │
        │  │  adaptive execution)   │
        │  └────────────┬────────────┘
        │  ┌────────────┴────────────┐
        │  │   Chain-of-Thought      │
        │  │  (Internal reasoning,   │
        │  │   no tool use)          │
        │  └─────────────────────────┘
        └──────────────────────────────────────► Token Efficiency
```

---

## 3. Pattern 1: Chain-of-Thought (CoT)

### Khái niệm

CoT là pattern đơn giản nhất: hướng dẫn Claude **suy nghĩ từng bước trước khi trả lời**, không cần tool nào. Câu thần chú nổi tiếng nhất: _"Let's think step by step."_

CoT không phải là agentic pattern theo nghĩa nghiêm túc — không có tool use, không có external actions. Nhưng nó là **nền tảng reasoning** cho mọi pattern phức tạp hơn.

### Zero-shot CoT vs Few-shot CoT

**Zero-shot CoT**: Chỉ thêm "think step by step" vào prompt.
```
"Debug this NullPointerException. Think step by step."
```

**Few-shot CoT**: Cung cấp ví dụ của reasoning chains.
```
"Debug this NullPointerException.

Example of how to debug:
Q: Why does this code throw NPE at line 23?
A: Let me trace the execution:
   1. method() is called at line 10
   2. It returns null when user is not found (line 15)
   3. The caller at line 23 calls .getName() without null check
   → Root cause: missing null check before line 23

Now debug this NPE: [your code]"
```

Few-shot CoT thường cho kết quả tốt hơn nhưng tốn nhiều tokens hơn.

### Java use case: Debugging Complex Spring Boot Issues

```java
package com.course.module06.lesson02;

import com.anthropic.client.AnthropicClient;
import com.anthropic.models.messages.*;

public class ChainOfThoughtDebugger {

    private static final String COT_SYSTEM_PROMPT = """
        You are an expert Spring Boot debugger. When given an error or issue,
        reason through it systematically before giving a diagnosis.
        
        Always structure your reasoning as:
        1. What the error message tells us
        2. Common root causes for this type of error
        3. How to narrow down to the specific cause
        4. The most likely root cause given the context
        5. Recommended fix
        
        Be precise and reference specific line numbers, class names, and
        Spring configuration properties when relevant.
        """;

    private final AnthropicClient client;

    public ChainOfThoughtDebugger(AnthropicClient client) {
        this.client = client;
    }

    public String debug(String errorContext) {
        Message response = client.messages().create(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_SONNET_4_6)
                .system(COT_SYSTEM_PROMPT)
                .addUserMessage(errorContext)
                .maxTokens(2048)
                .build()
        );

        return response.content().stream()
            .filter(b -> b instanceof TextBlock)
            .map(b -> ((TextBlock) b).text())
            .reduce("", String::concat);
    }

    public static void main(String[] args) {
        AnthropicClient client = AnthropicOkHttpClient.builder()
            .apiKey(System.getenv("ANTHROPIC_API_KEY"))
            .build();

        ChainOfThoughtDebugger debugger = new ChainOfThoughtDebugger(client);

        String errorContext = """
            Stack trace:
            org.springframework.beans.factory.UnsatisfiedDependencyException:
            Error creating bean with name 'userController': Unsatisfied dependency
            expressed through field 'userService'; nested exception is
            org.springframework.beans.factory.NoSuchBeanDefinitionException:
            No qualifying bean of type 'com.example.UserService' available
            
            UserController.java:
            @RestController
            public class UserController {
                @Autowired
                private UserService userService;  // line 8
            }
            
            UserService.java:
            public class UserService {  // line 1 — NO @Service annotation!
                // ...
            }
            """;

        System.out.println(debugger.debug(errorContext));
    }
}
```

### Khi nào dùng CoT:
- Tasks không cần external data (reasoning over provided context)
- Debugging với đầy đủ stack trace và code
- Architectural decisions, code review
- Giải thích complex concepts

---

## 4. Pattern 2: Plan-and-Execute

### Khái niệm

Plan-and-Execute tách biệt rõ ràng hai phases:

```
Phase 1 — PLANNING (Planner LLM):
  Input:  Task description
  Output: Ordered list of concrete, executable steps

Phase 2 — EXECUTION (Executor LLM hoặc cùng LLM):
  For each step in plan:
    Execute step
    Observe result
    If unexpected → Replan
```

Điểm khác biệt quan trọng so với ReAct: **plan được tạo trước, không phải on-the-fly**. Điều này cho phép:
- Kiểm tra plan trước khi execute (human review)
- Parallelize các steps độc lập
- Detect circular dependencies sớm

### Planner Prompt

```java
public static final String PLANNER_PROMPT = """
    You are an expert task planner. Given a task, break it down into a concrete,
    ordered list of executable steps.
    
    Rules for good plans:
    - Each step must be atomic (one clear action)
    - Steps must be in dependency order (prerequisites first)
    - Each step must specify which tool to use and with what arguments
    - Include verification steps after critical actions
    - Maximum 10 steps for most tasks
    
    Output format (JSON):
    {
      "task_summary": "Brief description of what we're accomplishing",
      "steps": [
        {
          "step_number": 1,
          "description": "What this step does",
          "tool": "tool_name",
          "arguments": {"key": "value"},
          "depends_on": [],
          "can_parallelize_with": []
        }
      ],
      "estimated_total_steps": N,
      "success_criteria": "How to know the task is complete"
    }
    """;
```

### Java Implementation

```java
package com.course.module06.lesson02;

import com.anthropic.client.AnthropicClient;
import com.anthropic.models.messages.*;
import com.fasterxml.jackson.databind.ObjectMapper;

import java.util.*;

public class PlanAndExecuteAgent {

    private final AnthropicClient client;
    private final Map<String, ToolExecutor> tools;
    private final ObjectMapper mapper = new ObjectMapper();

    public record ExecutionPlan(
        String taskSummary,
        List<PlanStep> steps,
        int estimatedTotalSteps,
        String successCriteria
    ) {}

    public record PlanStep(
        int stepNumber,
        String description,
        String tool,
        Map<String, Object> arguments,
        List<Integer> dependsOn,
        List<Integer> canParallelizeWith
    ) {}

    public record StepResult(
        int stepNumber,
        boolean success,
        String output,
        boolean needsReplan
    ) {}

    public String execute(String task) throws Exception {
        // Phase 1: Generate plan
        System.out.println("=== Phase 1: Planning ===");
        ExecutionPlan plan = generatePlan(task);
        System.out.println("Plan: " + plan.taskSummary());
        plan.steps().forEach(s ->
            System.out.printf("  Step %d: %s (tool: %s)%n",
                s.stepNumber(), s.description(), s.tool())
        );

        // Phase 2: Execute plan
        System.out.println("\n=== Phase 2: Execution ===");
        Map<Integer, StepResult> results = new LinkedHashMap<>();
        int currentStep = 0;

        while (currentStep < plan.steps().size()) {
            PlanStep step = plan.steps().get(currentStep);

            // Check dependencies
            if (!dependenciesMet(step, results)) {
                System.out.println("Waiting for dependencies for step " + step.stepNumber());
                currentStep++;
                continue;
            }

            System.out.printf("Executing step %d: %s%n",
                step.stepNumber(), step.description());

            StepResult result = executeStep(step, results);
            results.put(step.stepNumber(), result);

            // Replan nếu step thất bại hoặc kết quả unexpected
            if (result.needsReplan()) {
                System.out.println("Unexpected result, replanning from step " + step.stepNumber());
                plan = replan(task, plan, results, step.stepNumber());
                currentStep = findReplanStartStep(plan, results);
                continue;
            }

            currentStep++;
        }

        // Synthesize final answer từ tất cả results
        return synthesizeResults(task, plan, results);
    }

    private ExecutionPlan generatePlan(String task) throws Exception {
        Message response = client.messages().create(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_SONNET_4_6)
                .system(PlanningPrompts.PLANNER_PROMPT)
                .addUserMessage("Create a plan for this task:\n\n" + task)
                .maxTokens(2048)
                .build()
        );

        String jsonText = extractJsonFromResponse(response);
        return mapper.readValue(jsonText, ExecutionPlan.class);
    }

    private StepResult executeStep(PlanStep step, Map<Integer, StepResult> previousResults) {
        ToolExecutor executor = tools.get(step.tool());
        if (executor == null) {
            return new StepResult(step.stepNumber(), false,
                "Unknown tool: " + step.tool(), false);
        }

        try {
            // Inject results from dependent steps into arguments if needed
            Map<String, Object> args = injectDependencyOutputs(
                step.arguments(), step.dependsOn(), previousResults);

            String output = executor.execute(args);
            boolean needsReplan = detectUnexpectedResult(output, step);

            return new StepResult(step.stepNumber(), true, output, needsReplan);
        } catch (Exception e) {
            return new StepResult(step.stepNumber(), false,
                "Error: " + e.getMessage(), true);
        }
    }

    private ExecutionPlan replan(
            String originalTask,
            ExecutionPlan currentPlan,
            Map<Integer, StepResult> completedResults,
            int failedStep) throws Exception {

        String context = buildReplanContext(originalTask, currentPlan, completedResults, failedStep);

        Message response = client.messages().create(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_SONNET_4_6)
                .system(PlanningPrompts.PLANNER_PROMPT)
                .addUserMessage(context)
                .maxTokens(2048)
                .build()
        );

        return mapper.readValue(extractJsonFromResponse(response), ExecutionPlan.class);
    }

    private boolean dependenciesMet(PlanStep step, Map<Integer, StepResult> results) {
        return step.dependsOn().stream()
            .allMatch(dep -> results.containsKey(dep) && results.get(dep).success());
    }

    private boolean detectUnexpectedResult(String output, PlanStep step) {
        // Heuristic: nếu output chứa error keywords → cần replan
        String lower = output.toLowerCase();
        return lower.contains("error") || lower.contains("not found")
            || lower.contains("failed") || lower.contains("exception");
    }

    private String synthesizeResults(
            String task, ExecutionPlan plan, Map<Integer, StepResult> results) throws Exception {

        StringBuilder context = new StringBuilder();
        context.append("Original task: ").append(task).append("\n\n");
        context.append("Execution results:\n");
        results.forEach((step, result) ->
            context.append(String.format("Step %d: %s%n  Output: %s%n%n",
                step, result.success() ? "SUCCESS" : "FAILED", result.output()))
        );

        Message response = client.messages().create(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_SONNET_4_6)
                .addSystemMessage("Synthesize the execution results into a clear final answer.")
                .addUserMessage(context.toString())
                .maxTokens(1024)
                .build()
        );

        return extractText(response);
    }

    // ... helper methods (extractJsonFromResponse, injectDependencyOutputs, etc.)

    @FunctionalInterface
    interface ToolExecutor {
        String execute(Map<String, Object> args) throws Exception;
    }
}
```

### Java Refactoring Planner — Ví dụ thực tế

```java
package com.course.module06.lesson02;

public class JavaRefactoringPlanner {

    // Task: "Extract UserService business logic to separate domain layer"
    // Plan sẽ generate:
    // Step 1: list_files(src/) → Hiểu cấu trúc hiện tại
    // Step 2: read_file(UserService.java) → Phân tích methods
    // Step 3: identify_business_methods(UserService) → Tìm methods cần extract
    // Step 4: create_domain_class(UserDomainService) → Tạo class mới
    // Step 5: move_methods(UserService → UserDomainService) → Move logic
    // Step 6: update_dependencies(UserController) → Update callers
    // Step 7: run_tests() → Verify không break gì
    // Step 8: run_checkstyle() → Ensure code quality

    public static void main(String[] args) throws Exception {
        AnthropicClient client = AnthropicOkHttpClient.builder()
            .apiKey(System.getenv("ANTHROPIC_API_KEY"))
            .build();

        PlanAndExecuteAgent agent = new PlanAndExecuteAgent(client,
            buildRefactoringTools());

        String task = """
            Refactor the UserService class to separate concerns:
            1. Extract all database operations to UserRepository (if not already there)
            2. Keep only business logic in UserService
            3. Ensure all existing unit tests still pass
            4. Codebase: src/main/java/com/example/
            """;

        String result = agent.execute(task);
        System.out.println("Refactoring complete:\n" + result);
    }
}
```

---

## 5. Pattern 3: ReWOO (Reasoning WithOut Observation)

### Khái niệm

ReWOO (Xu et al., 2023) là một optimization của ReAct: thay vì interleave reasoning và acting, **toàn bộ plan được tạo upfront trước khi execute bất kỳ step nào**.

```
ReAct:
  Thought₁ → Action₁ → Observe₁ → Thought₂ → Action₂ → Observe₂ → ...

ReWOO:
  [Planning Phase]
  Thought₁ → Action₁ (planned)
  Thought₂ → Action₂ (planned, references #E1)
  Thought₃ → Action₃ (planned, references #E1, #E2)
  
  [Execution Phase]
  Execute Action₁ → #E1 = result
  Execute Action₂ → #E2 = result
  Execute Action₃ → #E3 = result
  
  [Solver Phase]
  Synthesize #E1, #E2, #E3 → Final Answer
```

### So sánh ReWOO vs ReAct

| | ReAct | ReWOO |
|---|---|---|
| **Token usage** | Cao hơn (repetitive context) | Thấp hơn (30-40% savings) |
| **Adaptability** | Cao (reacts to each observation) | Thấp (plan fixed upfront) |
| **Parallelism** | Khó (sequential by nature) | Dễ (steps có thể run parallel) |
| **Error recovery** | Tốt (can replan mid-stream) | Kém (cần restart nếu plan sai) |
| **Best for** | Uncertain, exploratory tasks | Predictable, structured tasks |

### Java Implementation

```java
package com.course.module06.lesson02;

import com.anthropic.client.AnthropicClient;
import com.anthropic.models.messages.*;
import java.util.*;
import java.util.concurrent.*;
import java.util.regex.*;

public class ReWOOAgent {

    private final AnthropicClient client;
    private final Map<String, ToolExecutor> tools;
    private final ExecutorService executor = Executors.newFixedThreadPool(4);

    // Pattern để parse planned steps: #E1 = tool[args]
    private static final Pattern STEP_PATTERN =
        Pattern.compile("#E(\\d+)\\s*=\\s*(\\w+)\\[(.+?)\\]", Pattern.DOTALL);

    // Pattern để reference previous evidence: #E1, #E2...
    private static final Pattern EVIDENCE_REF = Pattern.compile("#E(\\d+)");

    private static final String REWOO_PLANNER_PROMPT = """
        You are a task planner. Create a complete plan BEFORE any execution.
        
        Format each step as:
        Plan: [reasoning for this step]
        #E{N} = tool_name[argument that may reference #E{N-1}, #E{N-2}, ...]
        
        Example:
        Plan: First understand the project structure
        #E1 = list_files[src/main/java]
        
        Plan: Based on #E1, read the main service file
        #E2 = read_file[{file from #E1 that looks like main service}]
        
        Plan: Search for all usages of the class found in #E2
        #E3 = search_code[{class name from #E2}]
        
        Rules:
        - Reference previous evidence with #E{N} notation
        - Each step must produce useful evidence for later steps or the final answer
        - Plan ALL steps upfront, do not react to intermediate results
        """;

    public String run(String task) throws Exception {
        // Phase 1: Generate full plan
        System.out.println("=== ReWOO Phase 1: Planning ===");
        String rawPlan = generatePlan(task);
        List<PlannedStep> steps = parsePlan(rawPlan);

        System.out.println("Generated " + steps.size() + " planned steps:");
        steps.forEach(s -> System.out.printf("  #E%d = %s[...]%n",
            s.evidenceId(), s.tool()));

        // Phase 2: Execute steps (potentially parallel where safe)
        System.out.println("\n=== ReWOO Phase 2: Execution ===");
        Map<Integer, String> evidence = executeSteps(steps);

        // Phase 3: Solve using collected evidence
        System.out.println("\n=== ReWOO Phase 3: Solving ===");
        return solve(task, rawPlan, evidence);
    }

    private String generatePlan(String task) {
        Message response = client.messages().create(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_SONNET_4_6)
                .system(REWOO_PLANNER_PROMPT)
                .addUserMessage("Create a complete plan for:\n\n" + task)
                .maxTokens(2048)
                .build()
        );
        return extractText(response);
    }

    private List<PlannedStep> parsePlan(String rawPlan) {
        List<PlannedStep> steps = new ArrayList<>();
        Matcher matcher = STEP_PATTERN.matcher(rawPlan);

        while (matcher.find()) {
            int id = Integer.parseInt(matcher.group(1));
            String tool = matcher.group(2);
            String argTemplate = matcher.group(3);
            steps.add(new PlannedStep(id, tool, argTemplate));
        }

        return steps;
    }

    private Map<Integer, String> executeSteps(List<PlannedStep> steps) {
        Map<Integer, String> evidence = new ConcurrentHashMap<>();

        // Topological execution: execute steps in order, resolving #E references
        for (PlannedStep step : steps) {
            // Resolve evidence references in arguments
            String resolvedArg = resolveEvidenceRefs(step.argTemplate(), evidence);

            System.out.printf("Executing #E%d = %s[%s]%n",
                step.evidenceId(), step.tool(), truncate(resolvedArg, 100));

            ToolExecutor toolExec = tools.get(step.tool());
            String result;
            if (toolExec != null) {
                try {
                    result = toolExec.execute(Map.of("argument", resolvedArg));
                } catch (Exception e) {
                    result = "Error: " + e.getMessage();
                }
            } else {
                result = "Error: Unknown tool " + step.tool();
            }

            evidence.put(step.evidenceId(), result);
            System.out.println("  → " + truncate(result, 200));
        }

        return evidence;
    }

    private String resolveEvidenceRefs(String template, Map<Integer, String> evidence) {
        StringBuffer sb = new StringBuffer();
        Matcher matcher = EVIDENCE_REF.matcher(template);

        while (matcher.find()) {
            int refId = Integer.parseInt(matcher.group(1));
            String evidenceValue = evidence.getOrDefault(refId, "[evidence not yet available]");
            // Truncate large evidence to avoid prompt explosion
            matcher.appendReplacement(sb, truncate(evidenceValue, 500));
        }
        matcher.appendTail(sb);
        return sb.toString();
    }

    private String solve(String task, String plan, Map<Integer, String> evidence) {
        StringBuilder solverPrompt = new StringBuilder();
        solverPrompt.append("Original task: ").append(task).append("\n\n");
        solverPrompt.append("Execution plan:\n").append(plan).append("\n\n");
        solverPrompt.append("Evidence collected:\n");

        evidence.forEach((id, result) ->
            solverPrompt.append(String.format("#E%d: %s%n%n", id, result))
        );

        solverPrompt.append("\nUsing all the evidence above, provide the final answer.");

        Message response = client.messages().create(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_SONNET_4_6)
                .addSystemMessage("Synthesize the evidence to answer the original task completely.")
                .addUserMessage(solverPrompt.toString())
                .maxTokens(2048)
                .build()
        );

        return extractText(response);
    }

    record PlannedStep(int evidenceId, String tool, String argTemplate) {}

    private String truncate(String s, int max) {
        return s.length() > max ? s.substring(0, max) + "..." : s;
    }

    private String extractText(Message m) {
        return m.content().stream()
            .filter(b -> b instanceof TextBlock)
            .map(b -> ((TextBlock) b).text())
            .reduce("", String::concat);
    }

    @FunctionalInterface
    interface ToolExecutor {
        String execute(Map<String, Object> args) throws Exception;
    }
}
```

---

## 6. Pattern 4: Tree of Thoughts (ToT)

### Khái niệm

Tree of Thoughts (Yao et al., 2023) là pattern mạnh nhất và tốn kém nhất. Thay vì một single reasoning path, ToT explore **nhiều reasoning branches song song**, đánh giá từng branch, và chọn path tốt nhất.

```
                    Task
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
      Branch A   Branch B   Branch C
      (Approach  (Approach  (Approach
       1: REST)   2: gRPC)   3: MQ)
          │          │          │
       Score: 6   Score: 8   Score: 5
          │          │
          ×     ┌────┼────┐
               ▼         ▼
          Sub-branch B1  Sub-branch B2
          (Push model)   (Pull model)
               │              │
           Score: 7       Score: 9
               │
               ×        ← WINNER
                         ▼
                     Sub-branch B2
                     (Final path)
```

### Khi nào dùng ToT:

- **Creative problems**: "Design 3 approaches for rate limiting our API"
- **Optimization**: "Find the best database schema for this use case"
- **Multiple valid solutions**: "Suggest refactoring strategies, evaluate trade-offs"
- **Complex architectural decisions**: "Microservices vs monolith for our use case"

### Java Implementation

```java
package com.course.module06.lesson02;

import com.anthropic.client.AnthropicClient;
import com.anthropic.models.messages.*;
import java.util.*;
import java.util.concurrent.*;
import java.util.stream.*;

public class TreeOfThoughtsAgent {

    private final AnthropicClient client;
    private final int numBranches;        // Số branches per level
    private final int maxDepth;           // Độ sâu tối đa của tree
    private final int beamWidth;          // Giữ lại N best branches ở mỗi level

    public TreeOfThoughtsAgent(AnthropicClient client) {
        this(client, 3, 3, 2); // defaults: 3 branches, depth 3, beam width 2
    }

    public TreeOfThoughtsAgent(AnthropicClient client, int numBranches,
                                int maxDepth, int beamWidth) {
        this.client = client;
        this.numBranches = numBranches;
        this.maxDepth = maxDepth;
        this.beamWidth = beamWidth;
    }

    public String run(String task) throws Exception {
        System.out.println("=== Tree of Thoughts ===");
        System.out.printf("Config: %d branches, depth %d, beam width %d%n",
            numBranches, maxDepth, beamWidth);

        // Root node
        ThoughtNode root = new ThoughtNode(0, null, "ROOT", task, 0.0);
        List<ThoughtNode> currentLevel = List.of(root);

        ThoughtNode bestNode = root;
        double bestScore = -1;

        for (int depth = 1; depth <= maxDepth; depth++) {
            System.out.printf("%n--- Depth %d/%d ---%n", depth, maxDepth);

            // Generate children for all current level nodes
            List<ThoughtNode> nextLevel = expandNodes(currentLevel, task, depth);

            System.out.printf("Generated %d thoughts%n", nextLevel.size());

            // Evaluate all nodes at this level
            evaluateNodes(nextLevel, task);

            // Beam search: keep only top beamWidth nodes
            nextLevel.sort(Comparator.comparingDouble(ThoughtNode::score).reversed());
            currentLevel = nextLevel.stream().limit(beamWidth).collect(Collectors.toList());

            // Track best overall node
            ThoughtNode levelBest = nextLevel.get(0);
            if (levelBest.score() > bestScore) {
                bestScore = levelBest.score();
                bestNode = levelBest;
            }

            System.out.printf("Best at depth %d: score=%.2f, thought=%s%n",
                depth, levelBest.score(), truncate(levelBest.thought(), 100));

            // Early stopping: perfect score
            if (bestScore >= 9.5) {
                System.out.println("Excellent solution found, stopping early");
                break;
            }
        }

        // Generate final answer from the best path
        return generateFinalAnswer(task, buildPath(bestNode));
    }

    private List<ThoughtNode> expandNodes(
            List<ThoughtNode> nodes, String task, int depth) throws Exception {

        List<ThoughtNode> allChildren = new ArrayList<>();

        for (ThoughtNode node : nodes) {
            List<String> thoughts = generateThoughts(task, node, numBranches);
            for (int i = 0; i < thoughts.size(); i++) {
                allChildren.add(new ThoughtNode(
                    node.id() * 10 + i, node,
                    thoughts.get(i), task, 0.0
                ));
            }
        }

        return allChildren;
    }

    private List<String> generateThoughts(
            String task, ThoughtNode parent, int n) {

        String context = buildContext(task, parent);
        String prompt = String.format("""
            Task: %s
            
            Current thinking path:
            %s
            
            Generate %d DIFFERENT next thoughts or approaches to consider.
            Each thought should explore a distinct direction.
            
            Format:
            Thought 1: [thought]
            Thought 2: [thought]
            ...
            """, task, context, n);

        Message response = client.messages().create(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_HAIKU_4_5) // Dùng Haiku để tiết kiệm chi phí cho expansion
                .addUserMessage(prompt)
                .maxTokens(1024)
                .build()
        );

        return parseThoughts(extractText(response), n);
    }

    private void evaluateNodes(List<ThoughtNode> nodes, String task) {
        // Evaluate all nodes in parallel để tiết kiệm thời gian
        nodes.parallelStream().forEach(node -> {
            double score = evaluateThought(task, node);
            node.setScore(score);
            System.out.printf("  Evaluated: score=%.1f | %s%n",
                score, truncate(node.thought(), 80));
        });
    }

    private double evaluateThought(String task, ThoughtNode node) {
        String prompt = String.format("""
            Task: %s
            
            Proposed approach/thought: %s
            
            Rate this thought on a scale of 1-10 based on:
            - Feasibility (is it doable?)
            - Relevance (does it address the task?)
            - Originality (is it insightful?)
            - Completeness (does it make progress?)
            
            Respond with ONLY a number between 1 and 10.
            """, task, node.thought());

        Message response = client.messages().create(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_HAIKU_4_5)
                .addUserMessage(prompt)
                .maxTokens(10) // Chỉ cần số
                .build()
        );

        String scoreText = extractText(response).trim();
        try {
            return Double.parseDouble(scoreText.replaceAll("[^0-9.]", ""));
        } catch (NumberFormatException e) {
            return 5.0; // Default score nếu parse fail
        }
    }

    private String generateFinalAnswer(String task, List<ThoughtNode> path) {
        String pathDescription = path.stream()
            .skip(1) // Skip root
            .map(n -> "- " + n.thought())
            .collect(Collectors.joining("\n"));

        Message response = client.messages().create(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_SONNET_4_6) // Sonnet cho final synthesis
                .addSystemMessage("You are an expert synthesizer. Create a comprehensive final answer.")
                .addUserMessage(String.format("""
                    Task: %s
                    
                    Best reasoning path discovered:
                    %s
                    
                    Based on this reasoning path, provide a comprehensive final answer.
                    """, task, pathDescription))
                .maxTokens(2048)
                .build()
        );

        return extractText(response);
    }

    private List<ThoughtNode> buildPath(ThoughtNode node) {
        List<ThoughtNode> path = new ArrayList<>();
        ThoughtNode current = node;
        while (current != null) {
            path.add(0, current);
            current = current.parent();
        }
        return path;
    }

    private String buildContext(String task, ThoughtNode node) {
        List<ThoughtNode> path = buildPath(node);
        return path.stream()
            .skip(1) // Skip root
            .map(n -> "→ " + n.thought())
            .collect(Collectors.joining("\n"));
    }

    private List<String> parseThoughts(String response, int n) {
        List<String> thoughts = new ArrayList<>();
        Pattern p = Pattern.compile("Thought \\d+:\\s*(.+?)(?=Thought \\d+:|$)", Pattern.DOTALL);
        Matcher m = p.matcher(response);
        while (m.find() && thoughts.size() < n) {
            thoughts.add(m.group(1).trim());
        }
        // Pad nếu không parse đủ
        while (thoughts.size() < n) {
            thoughts.add("Explore alternative approach " + (thoughts.size() + 1));
        }
        return thoughts;
    }

    // Mutable node để set score sau khi evaluate
    static class ThoughtNode {
        private final int id;
        private final ThoughtNode parent;
        private final String thought;
        private final String task;
        private double score;

        ThoughtNode(int id, ThoughtNode parent, String thought, String task, double score) {
            this.id = id;
            this.parent = parent;
            this.thought = thought;
            this.task = task;
            this.score = score;
        }

        int id() { return id; }
        ThoughtNode parent() { return parent; }
        String thought() { return thought; }
        double score() { return score; }
        void setScore(double score) { this.score = score; }
    }

    private String extractText(Message m) {
        return m.content().stream()
            .filter(b -> b instanceof TextBlock)
            .map(b -> ((TextBlock) b).text())
            .reduce("", String::concat);
    }

    private String truncate(String s, int max) {
        return s.length() > max ? s.substring(0, max) + "..." : s;
    }
}
```

---

## 7. Decision Matrix — Chọn Pattern nào?

```
┌─────────────────────────────────────────────────────────────┐
│                 Planning Pattern Selection                    │
│                                                              │
│  Task type?                                                   │
│     │                                                         │
│     ├─► Simple, one-shot ────────────────► Direct API call   │
│     │   (translate, summarize)             (no planning)      │
│     │                                                         │
│     ├─► Debugging / Analysis ──────────────► Chain-of-Thought │
│     │   (with full context provided)        (internal reason)  │
│     │                                                         │
│     ├─► Multi-step, steps predictable? ─► YES ─► ReWOO       │
│     │                   │                         (efficient)  │
│     │                  NO                                      │
│     │                   │                                      │
│     │                   ▼                                      │
│     │   Need to adapt based on results? ─► YES ─► ReAct       │
│     │                   │                                      │
│     │                  NO                                      │
│     │                   │                                      │
│     │                   ▼                                      │
│     ├─► Linear multi-step ─────────────────► Plan-and-Execute │
│     │                                                          │
│     └─► Multiple valid solutions? ─────────► Tree of Thoughts │
│         (creative, optimization)                               │
└─────────────────────────────────────────────────────────────┘
```

### Quick reference bằng ví dụ thực tế:

| Use case | Pattern | Lý do |
|----------|---------|-------|
| "Explain this stack trace" | CoT | Chỉ cần reason, đủ context |
| "Refactor UserService, order matters" | Plan-and-Execute | Steps rõ ràng, có dependencies |
| "Find security issues in codebase" | ReAct | Không biết trước cần read file nào |
| "Generate API documentation" | ReWOO | Steps predictable: list → read → generate |
| "Design our microservices architecture" | ToT | Multiple valid designs cần evaluate |

---

## 8. PlannerAgent với Strategy Pattern

Đây là implementation tổng hợp cho phép switch giữa các planning modes:

```java
package com.course.module06.lesson02;

import com.anthropic.client.AnthropicClient;

public class PlannerAgent {

    public enum PlanningMode {
        CHAIN_OF_THOUGHT,
        PLAN_AND_EXECUTE,
        REWOO,
        TREE_OF_THOUGHTS
    }

    private final AnthropicClient client;
    private final PlanningMode mode;

    public PlannerAgent(AnthropicClient client, PlanningMode mode) {
        this.client = client;
        this.mode = mode;
    }

    public String run(String task) throws Exception {
        return switch (mode) {
            case CHAIN_OF_THOUGHT -> new ChainOfThoughtDebugger(client).debug(task);
            case PLAN_AND_EXECUTE  -> new PlanAndExecuteAgent(client, buildTools()).execute(task);
            case REWOO             -> new ReWOOAgent(client, buildTools()).run(task);
            case TREE_OF_THOUGHTS  -> new TreeOfThoughtsAgent(client).run(task);
        };
    }

    /**
     * Auto-select planning mode based on task characteristics.
     * Useful khi muốn agent tự quyết định.
     */
    public static PlanningMode selectMode(String task) {
        String lower = task.toLowerCase();

        // Simple analysis tasks → CoT
        if (lower.contains("explain") || lower.contains("debug")
                || lower.contains("analyze") && task.length() < 500) {
            return PlanningMode.CHAIN_OF_THOUGHT;
        }

        // Creative / multiple options → ToT
        if (lower.contains("design") || lower.contains("best approach")
                || lower.contains("alternatives") || lower.contains("compare")) {
            return PlanningMode.TREE_OF_THOUGHTS;
        }

        // Exploratory → ReAct
        if (lower.contains("find") || lower.contains("investigate")
                || lower.contains("search") || lower.contains("discover")) {
            return PlanningMode.REWOO; // ReWOO for structured search
        }

        // Default: Plan-and-Execute for complex multi-step
        return PlanningMode.PLAN_AND_EXECUTE;
    }

    private Map<String, PlanAndExecuteAgent.ToolExecutor> buildTools() {
        // Build và return tool registry
        return ToolRegistry.buildDefaultTools();
    }
}
```

---

## 9. Exercise

### Bài tập: Java Refactoring Planner

Implement một complete Plan-and-Execute agent để:

1. **Input**: Một Java class file có code smell (God class, long method, v.v.)
2. **Planning phase**: Agent tạo plan để refactor theo Clean Code principles
3. **Execution phase**: Agent thực hiện từng bước refactoring
4. **Verification**: Agent chạy tests để confirm không break gì

**Tools cần implement:**
- `read_file(path)` — đọc Java source file
- `write_file(path, content)` — ghi Java source file sau refactoring
- `run_tests(class)` — chạy unit tests
- `analyze_code_smells(path)` — detect code smells (God class, long method > 50 lines, deep nesting > 4 levels)

**Test input**: Tạo một `UserController.java` dài 300 dòng với business logic trong controller (anti-pattern).

**Expected plan từ agent:**
```json
{
  "steps": [
    {"step": 1, "tool": "read_file", "description": "Read UserController.java"},
    {"step": 2, "tool": "analyze_code_smells", "description": "Identify smells"},
    {"step": 3, "tool": "write_file", "description": "Create UserService.java with extracted logic"},
    {"step": 4, "tool": "write_file", "description": "Update UserController to delegate to UserService"},
    {"step": 5, "tool": "run_tests", "description": "Verify no regressions"}
  ]
}
```

**Checklist:**
- [ ] Plan được generate trước execution
- [ ] Agent replan khi step thất bại
- [ ] Tests pass sau refactoring
- [ ] Viết integration test cho PlanAndExecuteAgent với mock tools

---

## Tóm tắt

| Pattern | Khi dùng | Token cost | Adaptability |
|---------|----------|------------|--------------|
| **CoT** | Reasoning over given context | Thấp | N/A |
| **Plan-and-Execute** | Sequential tasks, known steps | Trung bình | Trung bình (replan) |
| **ReWOO** | Predictable multi-step | Thấp nhất | Thấp |
| **Tree of Thoughts** | Multiple valid solutions | Cao nhất | Cao |

---

> **Trước đó**: [Lesson 01 — ReAct Pattern](01-react-pattern.md)
> **Tiếp theo**: [Lesson 03 — State Management](03-state-management.md)
