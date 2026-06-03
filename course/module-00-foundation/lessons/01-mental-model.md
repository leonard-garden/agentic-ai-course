# Lesson 01 — Mental Model: Agentic AI là gì?

> **Module**: 00 — Foundation & Mental Model
> **Thời gian đọc**: ~45 phút | **Thời gian thực hành**: ~45 phút
> **Difficulty**: Beginner

---

## Mục tiêu bài học

Sau bài này bạn có thể:
- Giải thích Agentic AI khác Traditional Software ở điểm nào
- Hiểu LLM là gì và tại sao nó là "reasoning engine, not a database"
- Phân biệt 2 Anthropic primitives: Messages API vs Managed Agents
- Nhận ra và đặt tên 5 workflow patterns từ Anthropic
- Map từng pattern sang Java/microservices concepts quen thuộc

---

## 1. Agentic AI là gì? Khác gì Traditional Software?

### Traditional Software: Deterministic và Rule-Based

Khi bạn viết một Spring Boot service xử lý đơn hàng, logic của nó hoàn toàn **deterministic**:

```java
// Traditional: Rule-based, predictable, deterministic
public OrderResult processOrder(Order order) {
    if (order.getTotal() > 1000) {
        return requireManagerApproval(order);
    }
    if (inventory.isAvailable(order.getItems())) {
        return fulfillOrder(order);
    }
    return backorderItems(order);
}
```

Bạn biết chính xác:
- Input nào dẫn đến output nào
- Execution path nào được chạy
- Kết quả có thể test và reproduce 100%

### Agentic AI: Probabilistic và Goal-Directed

Một AI agent được cho mục tiêu, không phải instructions:

```
Goal: "Xử lý các support tickets tồn đọng trong hệ thống.
       Phân loại, gán priority, draft responses, và escalate
       những gì cần human review."
```

Agent sẽ **tự quyết định**:
1. Dùng tool nào (đọc database? gọi API? tìm kiếm docs?)
2. Thực hiện bước nào trước
3. Khi nào cần dừng lại và hỏi human
4. Cách handle edge cases chưa được lập trình sẵn

Đây là sự khác biệt cốt lõi:

| Dimension | Traditional Software | Agentic AI |
|-----------|---------------------|------------|
| **Behavior** | Rule-based, deterministic | Goal-directed, probabilistic |
| **Error handling** | Explicit exceptions | Reasoning about failures |
| **Adaptation** | Requires code changes | Adapts from context |
| **Task scope** | Single, well-defined tasks | Open-ended, multi-step tasks |
| **State** | Explicit state machines | Emergent state from reasoning |
| **Testing** | Unit tests, deterministic | Evals, probabilistic assertions |
| **Debugging** | Stack traces | Reasoning traces, token inspection |

### Analogy cho Java Engineer

Hãy nghĩ về sự khác biệt giữa:

- **Traditional Software** = Một stored procedure PostgreSQL: bạn viết chính xác từng bước, input/output type-safe, có thể test mọi edge case
- **Agentic AI** = Một senior developer được giao một ticket: họ đọc requirements, hỏi câu hỏi làm rõ, tự quyết định approach, và deliver kết quả — đôi khi theo cách bạn không ngờ tới, nhưng thường đúng

---

## 2. LLM as a Reasoning Engine, Not a Database

Đây là một trong những mental model quan trọng nhất. Rất nhiều người nghĩ LLM như một "knowledge base" hoặc "smart search engine". Điều này dẫn đến cách dùng sai.

### LLM KHÔNG phải là:

- ❌ Một database chứa facts (có thể outdated, có thể "hallucinate")
- ❌ Một search engine (không index real-time data)
- ❌ Một calculator (không tính toán số học chính xác natively)
- ❌ Một deterministic function (cùng input, có thể khác output)

### LLM LÀ:

- ✅ Một **reasoning engine** — có thể suy luận, lên kế hoạch, phân tích
- ✅ Một **language understanding engine** — hiểu context, intent, nuance
- ✅ Một **code generation engine** — biết hàng nghìn patterns, APIs, frameworks
- ✅ Một **orchestration brain** — có thể quyết định khi nào gọi tool nào

### Implication quan trọng cho Java Engineers

```java
// WRONG mental model: Dùng LLM như database
String answer = claude.ask("Số điện thoại của khách hàng #12345 là gì?");
// → Claude không có dữ liệu này. Nó sẽ hallucinate hoặc nói "tôi không biết"

// CORRECT mental model: Cấp tools để LLM có thể query
Tool customerLookupTool = Tool.builder()
    .name("get_customer_phone")
    .description("Lấy số điện thoại khách hàng từ CRM")
    .inputSchema(phoneSchema)
    .build();

// Bây giờ Claude có thể REASON về việc phải dùng tool này
// Claude tự quyết định: "Tôi cần số điện thoại → tôi sẽ gọi get_customer_phone"
```

### The Reasoning Loop

LLM hoạt động theo vòng lặp reasoning:

```
Observe context →
Reason about what's needed →
Decide action (respond? use tool? ask for clarification?) →
Execute action →
Observe result →
Repeat until goal achieved
```

Đây là lý do tại sao Agentic AI mạnh hơn simple chatbots — nó có thể **loop** qua nhiều steps, không chỉ respond once.

---

## 3. Hai Anthropic Primitives: Messages API vs Managed Agents

Anthropic cung cấp 2 cách chính để xây dựng với Claude:

### Primitive 1: Messages API

Messages API là building block cơ bản nhất. Bạn gửi một message (hoặc conversation history), Claude trả về response.

```java
// Messages API — bạn control toàn bộ conversation loop
var response = client.messages().create(
    MessageCreateParams.builder()
        .model(Model.CLAUDE_SONNET_4_5)
        .maxTokens(1024)
        .addUserMessage("Phân tích đoạn code Java này và tìm bugs")
        .build()
);
```

**Bạn chịu trách nhiệm về**:
- Conversation history management
- Tool execution loop
- State persistence
- Retry logic
- Multi-turn orchestration

### Primitive 2: Managed Agents (Agent SDK)

Managed Agents là higher-level abstraction — Anthropic quản lý nhiều phức tạp cho bạn.

```python
# Agent SDK (Python — Java SDK đang phát triển)
import anthropic

client = anthropic.Anthropic()
agent = client.beta.agents.create(
    model="claude-sonnet-4-5",
    tools=[code_analysis_tool, database_tool],
    instructions="Bạn là một code review agent..."
)

# Agent tự manage conversation loop, tool calls, state
run = client.beta.agents.runs.create(
    agent_id=agent.id,
    thread_id=thread.id,
    additional_instructions="Focus vào security vulnerabilities"
)
```

**Anthropic quản lý**:
- Conversation threading
- Tool call / response loop
- Basic state management
- Run lifecycle

### So sánh chi tiết

| Dimension | Messages API | Managed Agents |
|-----------|-------------|----------------|
| **Control** | Full control | Framework manages loop |
| **Flexibility** | Maximum | Opinionated structure |
| **Complexity** | You build everything | Abstracted away |
| **State** | You manage | Framework manages threads |
| **Tool execution** | You implement loop | Framework handles |
| **Best for** | Custom workflows, fine-grained control | Standard agent patterns, rapid prototyping |
| **Java support** | Full SDK support | Limited (Python-first) |
| **Production readiness** | Proven | Newer, evolving |

### Khi nào dùng gì?

**Dùng Messages API khi:**
- Cần full control over conversation flow
- Đang build trong Java (Java SDK hỗ trợ đầy đủ)
- Logic của bạn có nhiều custom branching
- Cần integrate sâu với existing Java infrastructure
- Đang làm production system cần predictability

**Dùng Managed Agents khi:**
- Prototyping nhanh một agent
- Standard patterns phù hợp với use case
- Team đang dùng Python
- Muốn Anthropic handle infrastructure complexity

> **Practical note cho khóa học này**: Chúng ta sẽ học Messages API làm nền tảng (Module 01–04), sau đó explore Managed Agents trong Module 04. Hiểu Messages API sâu giúp bạn dùng bất kỳ abstraction nào ở trên nó.

---

## 4. Workflows vs Agents Taxonomy

Đây là framework quan trọng nhất từ bài viết "Building Effective Agents" của Anthropic. Hiểu taxonomy này giúp bạn chọn đúng approach cho từng bài toán.

### Workflows: Predefined Paths

**Workflows** là các pattern mà LLM steps được orchestrated theo một predefined flow. Bạn biết trước luồng xử lý — LLM là một component trong flow đó.

### Agents: Dynamic Orchestration

**Agents** là các system mà LLM tự quyết định flow, tool nào dùng, và khi nào dừng. LLM là orchestrator, không chỉ là component.

> **Key insight**: Không phải mọi use case đều cần "agent". Nhiều bài toán giải quyết tốt hơn với simple workflows. Complexity có cost: harder to debug, less predictable, more expensive.

---

## 5. Năm Workflow Patterns

### Pattern 1: Prompt Chaining

**Khái niệm**: Chia một complex task thành nhiều bước tuần tự. Output của bước trước là input của bước sau.

```
Input → [LLM Step 1] → intermediate output → [LLM Step 2] → final output
```

**Ví dụ thực tế**: Generate API documentation từ Java code
```
Java source code
    ↓
[LLM Step 1: Phân tích code, extract method signatures và logic]
    ↓
Structured analysis (JSON)
    ↓
[LLM Step 2: Viết Javadoc từ analysis]
    ↓
[LLM Step 3: Generate OpenAPI spec từ Javadoc]
    ↓
Final documentation
```

**Java/Microservices Analogy**: Giống **pipeline pattern** trong stream processing (Java Streams, Kafka Streams). Mỗi stage transform data, output của stage này là input của stage tiếp theo.

```java
// Java Streams — cùng mental model
List<ApiDoc> docs = javaSources.stream()
    .map(source -> llm.extractMethods(source))      // Step 1
    .map(analysis -> llm.generateJavadoc(analysis)) // Step 2
    .map(javadoc -> llm.buildOpenApiSpec(javadoc))  // Step 3
    .collect(toList());
```

**Khi nào dùng**: Task có thể chia thành sequential steps rõ ràng, mỗi step có well-defined input/output, intermediate results cần validation.

---

### Pattern 2: Routing

**Khái niệm**: Phân loại input và route đến specialized handler phù hợp nhất.

```
Input → [Classifier LLM] → route decision → [Specialized Handler A | B | C]
```

**Ví dụ thực tế**: Customer support ticket routing
```
Support ticket text
    ↓
[LLM Classifier: Đây là billing issue, technical bug, hay feature request?]
    ↓ routing decision
    ├─ "billing" → [Billing Agent với access to payment data]
    ├─ "bug" → [Technical Agent với access to logs, code]
    └─ "feature" → [Product Agent với access to roadmap]
```

**Java/Microservices Analogy**: Giống **API Gateway routing** hoặc **Strategy Pattern**. Bạn có một router/dispatcher ở trước, nó quyết định handler nào xử lý request.

```java
// Strategy Pattern — cùng mental model
public interface TicketHandler {
    TicketResponse handle(Ticket ticket);
}

// Routing dựa trên LLM classification
public class TicketRouter {
    public TicketResponse route(Ticket ticket) {
        TicketType type = classifier.classify(ticket); // LLM call
        TicketHandler handler = handlerRegistry.get(type);
        return handler.handle(ticket);
    }
}
```

**Khi nào dùng**: Input có nhiều loại rõ ràng, mỗi loại cần xử lý khác nhau, classification quan trọng hơn việc xử lý một approach "trung bình".

---

### Pattern 3: Parallelization

**Khái niệm**: Chạy nhiều LLM tasks đồng thời, sau đó aggregate kết quả. Có hai sub-pattern:

**3a. Sectioning** — Chia task lớn thành independent sub-tasks chạy parallel:

```
Large input
    ↓ split
    ├─ [LLM Worker 1: Phần A]  ─┐
    ├─ [LLM Worker 2: Phần B]  ─┼─ aggregate → Final result
    └─ [LLM Worker 3: Phần C]  ─┘
```

**3b. Voting / Multi-perspective** — Chạy cùng task nhiều lần, vote/combine kết quả:

```
Same input → [LLM Run 1] ─┐
             [LLM Run 2] ─┼─ vote/combine → More reliable result
             [LLM Run 3] ─┘
```

**Ví dụ thực tế**: Code review cho một large PR
```
PR với 50 files thay đổi
    ↓ split by domain
    ├─ [Security Review Agent: auth, validation files]
    ├─ [Performance Review Agent: database, caching files]
    └─ [Style Review Agent: all files]
    ↓ aggregate
Final review report
```

**Java/Microservices Analogy**: Giống **CompletableFuture.allOf()** hoặc **parallel streams** trong Java. Scatter-gather pattern phổ biến trong microservices.

```java
// CompletableFuture — cùng mental model
CompletableFuture<SecurityReview> securityFuture =
    CompletableFuture.supplyAsync(() -> securityAgent.review(pr));
CompletableFuture<PerformanceReview> perfFuture =
    CompletableFuture.supplyAsync(() -> perfAgent.review(pr));
CompletableFuture<StyleReview> styleFuture =
    CompletableFuture.supplyAsync(() -> styleAgent.review(pr));

CompletableFuture.allOf(securityFuture, perfFuture, styleFuture)
    .thenApply(v -> aggregateReviews(
        securityFuture.join(),
        perfFuture.join(),
        styleFuture.join()
    ));
```

**Khi nào dùng**: Task có thể chia thành independent subtasks, latency quan trọng (parallel faster than sequential), cần multiple perspectives để tăng reliability.

---

### Pattern 4: Orchestrator-Workers

**Khái niệm**: Một orchestrator LLM lên kế hoạch và delegate tasks cho worker LLMs (hoặc tools). Orchestrator tổng hợp kết quả.

```
Goal/Task
    ↓
[Orchestrator LLM: Plan và delegate]
    ├─ "Worker A: làm task X" → [Worker A LLM] → result X
    ├─ "Worker B: làm task Y" → [Worker B LLM] → result Y
    └─ "Worker C: làm task Z" → [Worker C LLM] → result Z
    ↓
[Orchestrator: tổng hợp X + Y + Z]
    ↓
Final output
```

**Ví dụ thực tế**: Autonomous code migration agent
```
Goal: "Migrate Spring Boot 2.x app lên Spring Boot 3.x"
    ↓
[Orchestrator: Phân tích dependencies, lên migration plan]
    ↓ tasks
    ├─ [Worker: Update pom.xml dependencies]
    ├─ [Worker: Migrate javax.* imports sang jakarta.*]
    ├─ [Worker: Update security configuration]
    └─ [Worker: Fix deprecated APIs]
    ↓
[Orchestrator: Verify tổng thể, tạo migration report]
```

**Java/Microservices Analogy**: Giống **Saga Pattern** trong distributed systems hoặc **Command Pattern với Orchestrator**. Một service điều phối nhiều services khác để hoàn thành một business transaction.

```java
// Saga Orchestrator — cùng mental model
public class MigrationOrchestrator {
    public MigrationResult orchestrate(Project project) {
        MigrationPlan plan = planningLLM.createPlan(project);

        List<CompletableFuture<TaskResult>> tasks = plan.getTasks()
            .stream()
            .map(task -> executeWithWorker(task))
            .collect(toList());

        List<TaskResult> results = waitForAll(tasks);
        return planningLLM.synthesize(results); // Orchestrator aggregates
    }
}
```

**Khi nào dùng**: Tasks phức tạp, chưa biết trước các subtasks, cần dynamic planning, mỗi subtask cần specialized expertise.

---

### Pattern 5: Evaluator-Optimizer

**Khái niệm**: Một LLM generate output, một LLM khác evaluate và provide feedback. Loop lặp lại cho đến khi đạt quality threshold.

```
Task input
    ↓
[Generator LLM: Tạo output]
    ↓ output
[Evaluator LLM: Đánh giá chất lượng]
    ↓ feedback (pass/fail + suggestions)
    ├─ "pass" → ✅ Done
    └─ "needs improvement" → [Generator LLM: Revise based on feedback]
                                ↓ (loop back to Evaluator)
```

**Ví dụ thực tế**: Generate unit tests với high coverage
```
Java method code
    ↓
[Generator: Viết unit tests]
    ↓ tests
[Evaluator: Kiểm tra coverage, edge cases, quality]
    ↓ feedback
    ├─ Coverage < 80% → "Thiếu test cho null input, exception path"
    ↓ (Generator revise)
    ├─ Coverage >= 80% nhưng test quality thấp → "Tests không test behavior, chỉ test implementation"
    ↓ (Generator revise)
    └─ Pass all criteria → ✅ Tests accepted
```

**Java/Microservices Analogy**: Giống **retry with backoff** nhưng thông minh hơn — thay vì retry blindly, nó retry với improvements. Cũng giống **CI/CD pipeline** với feedback loop: code → build → test → feedback → fix → repeat.

```java
// Evaluator-Optimizer loop
public TestSuite generateQualityTests(JavaMethod method) {
    TestSuite tests = generator.generate(method);
    EvaluationResult eval;

    int maxIterations = 5;
    int iteration = 0;

    do {
        eval = evaluator.evaluate(tests, method);
        if (!eval.isAcceptable()) {
            tests = generator.revise(tests, eval.getFeedback());
        }
        iteration++;
    } while (!eval.isAcceptable() && iteration < maxIterations);

    return tests;
}
```

**Khi nào dùng**: Output quality có thể measure objectively, iterative improvement feasible, first-pass output chưa đủ tốt consistently.

---

## 6. Tổng hợp: Choosing the Right Pattern

Đây là decision tree khi bạn đối mặt với một AI engineering problem:

```
Bài toán của tôi là gì?
│
├─ Có thể chia thành sequential steps rõ ràng?
│   └─ YES → Prompt Chaining
│
├─ Input có nhiều loại, mỗi loại xử lý khác nhau?
│   └─ YES → Routing
│
├─ Có nhiều independent sub-tasks cần chạy?
│   └─ YES → Parallelization
│       ├─ Sub-tasks khác nhau → Sectioning
│       └─ Cần multiple perspectives → Voting
│
├─ Task phức tạp, cần lên plan động?
│   └─ YES → Orchestrator-Workers
│
├─ Output cần đạt quality threshold qua iteration?
│   └─ YES → Evaluator-Optimizer
│
└─ Đơn giản, single-turn task?
    └─ Simple LLM call (không cần pattern phức tạp)
```

> **Nguyên tắc quan trọng từ Anthropic**: "Use the simplest solution that works." Đừng build agent phức tạp khi một simple prompt chain là đủ. Complexity có cost: harder to debug, less predictable, more expensive, more latency.

---

## 7. Patterns có thể kết hợp

Các patterns không loại trừ nhau. Ví dụ một production system thực tế:

```
User request: "Review toàn bộ codebase và tạo security report"
    ↓
[ROUTING: Classify request type → "security_audit"]
    ↓
[ORCHESTRATOR: Lên kế hoạch audit]
    ↓
    ├─ [PARALLELIZATION: Scan các modules song song]
    │   ├─ [Auth module scan]
    │   ├─ [API layer scan]
    │   └─ [Data layer scan]
    ↓
[PROMPT CHAINING:
    Step 1: Raw findings → Structured vulnerabilities
    Step 2: Structured vulns → Prioritized recommendations
    Step 3: Recommendations → Executive report]
    ↓
[EVALUATOR-OPTIMIZER: Kiểm tra report quality, revise nếu cần]
    ↓
Final security report
```

---

## Tổng kết bài học

| Concept | Key Takeaway |
|---------|-------------|
| Agentic AI vs Traditional | Goal-directed, probabilistic vs rule-based, deterministic |
| LLM as Reasoning Engine | Dùng LLM để reason + orchestrate, không phải để store data |
| Messages API | Building block cơ bản, full control, Java-native |
| Managed Agents | Higher abstraction, Anthropic manages loop |
| Prompt Chaining | Sequential steps, known flow |
| Routing | Classify then specialize |
| Parallelization | Independent tasks concurrently |
| Orchestrator-Workers | Dynamic planning + delegation |
| Evaluator-Optimizer | Generate + evaluate + improve loop |

---

## Quiz: Kiểm tra hiểu bài

**Câu 1**: Một Java developer nói: "Tôi sẽ dùng Claude để lookup customer data từ database." Điều gì sai trong statement này?

<details>
<summary>Xem đáp án</summary>

**Sai**: Claude không thể trực tiếp lookup database của bạn. LLM là reasoning engine, không phải database. Cách đúng: cung cấp cho Claude một `database_query` tool, để Claude reason về khi nào cần query và gọi tool đó.

</details>

---

**Câu 2**: Bạn cần build một system: nhận CV (PDF), extract thông tin, so sánh với job requirements, và generate hiring recommendation. Pattern nào phù hợp nhất?

A) Routing
B) Prompt Chaining
C) Parallelization
D) Managed Agents

<details>
<summary>Xem đáp án</summary>

**B) Prompt Chaining** — Đây là sequential steps rõ ràng:
1. Extract thông tin từ CV
2. So sánh với job requirements
3. Generate recommendation

Mỗi step có well-defined input/output, và step sau phụ thuộc vào output của step trước.

</details>

---

**Câu 3**: Công ty bạn có 3 support channels: billing, technical support, và sales. Mỗi channel có different context và different tools. Khi customer gửi message, bạn cần route đến đúng channel. Pattern nào dùng?

<details>
<summary>Xem đáp án</summary>

**Routing** — Đây là classic routing use case: classify input (loại message nào?) rồi route đến specialized handler. Classifier có thể là một LLM call nhỏ (dùng Haiku để tiết kiệm chi phí), sau đó route đến specialized agent phù hợp.

</details>

---

**Câu 4**: Bạn cần review một PR với 100 files thay đổi. Nếu xử lý sequential, mỗi file mất 2 giây → 200 giây. Bạn muốn giảm xuống ~10 giây. Pattern nào?

<details>
<summary>Xem đáp án</summary>

**Parallelization (Sectioning)** — Chia 100 files thành groups, mỗi group xử lý bởi một worker song song. Giống `CompletableFuture.allOf()` trong Java. Có thể group theo domain (security files, business logic files, config files) để mỗi worker có context phù hợp.

</details>

---

**Câu 5**: Bạn đang build một feature: generate SQL query từ natural language. Requirement: query phải chạy được và trả về đúng kết quả. Sau khi generate, bạn cần validate và nếu có lỗi, tự sửa. Pattern nào?

<details>
<summary>Xem đáp án</summary>

**Evaluator-Optimizer** — 
- Generator: LLM tạo SQL query
- Evaluator: Thực sự chạy query trên DB test, check syntax errors, check kết quả có match với intent không
- Nếu fail: feedback → Generator sửa
- Loop cho đến khi query valid và đúng

Đây là pattern mạnh cho code generation tasks vì có objective evaluation criteria (code runs or doesn't).

</details>

---

## Bài tập thực hành

### Exercise 1: Pattern Identification (15 phút)

Cho mỗi use case sau, xác định pattern phù hợp và giải thích tại sao:

1. Một system nhận invoice (PDF), extract line items, categorize expenses, và generate accounting journal entry
2. Một chatbot có thể handle cả technical questions, billing questions, và general questions — mỗi loại cần access vào different data sources
3. Một system cần generate test cases cho một Java class — yêu cầu: coverage >= 90%, tests phải pass, test names phải descriptive

### Exercise 2: Java Mental Model Mapping (20 phút)

Mở một Spring Boot project bạn đang làm. Identify một business flow và:
1. Viết ra flow hiện tại (deterministic, rule-based)
2. Redesign flow nếu dùng Agentic AI — pattern nào bạn sẽ dùng?
3. Identify điểm nào LLM có thể add value nhất

---

## Tài liệu đọc thêm

- **Bắt buộc**: [Building Effective Agents — Anthropic](https://www.anthropic.com/research/building-effective-agents) — bài viết gốc mô tả 5 patterns
- **Khuyến nghị**: [Anthropic Agent Patterns](https://docs.anthropic.com/en/docs/build-with-claude/agents) — docs chính thức
- **Mở rộng**: [Agentic AI: A Conceptual Framework](https://anthropic.com/academy) — Anthropic Academy free course

---

*Tiếp theo: [Lesson 02 — Anthropic Ecosystem](./02-anthropic-ecosystem.md)*
