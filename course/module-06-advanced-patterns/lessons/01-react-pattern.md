# Lesson 01: ReAct Pattern — Reasoning + Acting

> **Module 06 — Advanced Agentic Patterns**
> **Thời gian**: ~ 4 giờ | **Difficulty**: Intermediate-Advanced

---

## Mục tiêu bài học

Sau bài này bạn sẽ:
- Hiểu ReAct là gì và tại sao nó là nền tảng của mọi agentic system
- Phân biệt được ReAct với Chain-of-Thought và simple tool use
- Implement ReAct loop hoàn chỉnh trong Java với Anthropic SDK
- Viết ReAct system prompt chuẩn
- Tránh được các lỗi phổ biến khi triển khai ReAct

---

## 1. ReAct là gì?

**ReAct** (viết tắt của _Reasoning + Acting_) là một pattern được giới thiệu trong bài báo [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629) của Yao et al., 2022. Ý tưởng cốt lõi rất đơn giản nhưng cực kỳ mạnh mẽ:

> **Thay vì để LLM chỉ _suy nghĩ_ hoặc chỉ _hành động_, ReAct interleave (xen kẽ) hai quá trình đó trong một vòng lặp liên tục.**

Mỗi iteration của vòng lặp bao gồm:

```
Thought:     [Claude lý luận về tình huống hiện tại]
Action:      [Claude gọi một tool — search, read file, execute code...]
Observation: [Kết quả trả về từ tool]
Thought:     [Claude lý luận về kết quả vừa nhận]
Action:      [Tool call tiếp theo, hoặc trả về câu trả lời cuối]
...
```

Vòng lặp này tiếp tục cho đến khi agent tìm ra câu trả lời hoặc đạt giới hạn bước.

---

## 2. Tại sao ReAct quan trọng?

Hãy tưởng tượng bạn là một senior Java engineer được giao task: _"Tìm tất cả security vulnerabilities trong codebase này."_

**Nếu không có ReAct (blind acting):**
```
Agent gọi read_file("src/") → đọc tất cả files
Agent gọi analyze_security() → chạy analysis
Agent trả về kết quả
```
Vấn đề: agent không _suy nghĩ_ giữa các bước. Nó không biết file nào quan trọng hơn, không biết kết quả bước trước ảnh hưởng thế nào đến bước sau.

**Nếu chỉ có CoT (reasoning without acting):**
```
Thought: "Tôi nên kiểm tra auth module trước vì đó là attack surface lớn nhất"
Thought: "Sau đó check SQL queries để tìm injection vulnerabilities"
Thought: "Cuối cùng review input validation"
Answer: "Đây là những chỗ có thể có vulnerability: ..."
```
Vấn đề: agent _suy nghĩ_ rất hay nhưng không thực sự đọc code. Output là hallucination.

**Với ReAct:**
```
Thought: "Tôi nên bắt đầu bằng cách xem cấu trúc project để hiểu các module chính"
Action: list_files("src/main/java")
Observation: [AuthController.java, UserService.java, PaymentService.java, ...]

Thought: "AuthController có vẻ là priority cao nhất cho security review"
Action: read_file("src/main/java/AuthController.java")
Observation: [file content với potential JWT vulnerability...]

Thought: "Tôi thấy JWT token không được validate expiry. Cần check service layer"
Action: read_file("src/main/java/AuthService.java")
Observation: [confirms vulnerability, cũng thấy SQL concatenation...]

Thought: "Tìm thấy 2 issues: JWT và SQL injection. Để final answer"
Answer: "Security vulnerabilities found: 1) JWT không validate expiry tại line 47..."
```

ReAct cho phép agent **update reasoning dựa trên real observations** — đây là điều làm cho nó mạnh mẽ hơn hẳn các approach khác.

---

## 3. ReAct vs Chain-of-Thought vs Simple Tool Use

| | Chain-of-Thought | Simple Tool Use | ReAct |
|---|---|---|---|
| **Reasoning** | Có (internal) | Không | Có (explicit) |
| **Acting** | Không | Có (one-shot) | Có (multi-step) |
| **Adaptive** | Không | Không | Có |
| **Grounded** | Không (hallucination risk) | Có | Có |
| **Multi-step** | Không | Không (thường) | Có |
| **Dùng khi nào** | Reasoning tasks | Đơn giản, biết trước tool | Phức tạp, cần adapt |

**Chain-of-Thought (CoT)**: Claude suy nghĩ step-by-step trong internal monologue nhưng không gọi bất kỳ tool nào. Output dựa 100% vào training data — có thể sai hoặc outdated.

**Simple Tool Use**: Claude biết chính xác tool nào cần gọi ngay từ đầu. Phù hợp khi task đơn giản, ví dụ: "Dịch text này sang tiếng Anh" (gọi `translate` một lần, xong).

**ReAct**: Claude vừa suy nghĩ, vừa hành động, vừa điều chỉnh dựa trên kết quả. Phù hợp cho tasks phức tạp, không biết trước cần làm gì tiếp theo.

---

## 4. The Core ReAct Loop

```
┌─────────────────────────────────────────┐
│              ReAct Loop                  │
│                                          │
│  Input Task                              │
│      │                                   │
│      ▼                                   │
│  ┌─────────┐                             │
│  │ THOUGHT │ Claude reasons about        │
│  │         │ current state               │
│  └────┬────┘                             │
│       │                                  │
│       ▼                                  │
│  ┌─────────┐    Tool Result              │
│  │ ACTION  │◄──────────────┐             │
│  │(tool    │               │             │
│  │ call)   │               │             │
│  └────┬────┘               │             │
│       │                    │             │
│       ▼                    │             │
│  ┌─────────────┐           │             │
│  │ OBSERVATION │───────────┘             │
│  │(tool result)│                         │
│  └────┬────────┘                         │
│       │                                  │
│       ▼                                  │
│  Final Answer? ─── Yes ──► Return Result │
│       │                                  │
│      No                                  │
│       │                                  │
│       └──► Back to THOUGHT               │
│                                          │
│  (max steps guard prevents infinite loop)│
└─────────────────────────────────────────┘
```

---

## 5. Implementing ReAct với Claude — Java

### 5.1 ReAct System Prompt

System prompt là thành phần quan trọng nhất để trigger ReAct behavior. Nó phải:
1. Hướng dẫn Claude _format_ output đúng (Thought/Action/Observation)
2. Liệt kê tools có sẵn
3. Định nghĩa stopping condition (khi nào thì dùng "Final Answer")

```java
public class ReActPrompts {

    public static final String REACT_SYSTEM_PROMPT = """
        You are an expert Java engineer assistant. You solve tasks by interleaving \
        reasoning (Thought) with tool calls (Action) and observing results (Observation).
        
        Available tools:
        - list_files(path): List files in a directory
        - read_file(path): Read content of a file
        - search_code(query, path): Search for code patterns using regex
        - run_checkstyle(file): Run Checkstyle on a Java file, returns violations
        - run_tests(test_class): Run a specific test class, returns pass/fail results
        - execute_sql(query): Execute a read-only SQL query on the schema
        
        Follow this EXACT format for every step:
        
        Thought: [Your reasoning about what to do next. Be specific about WHY you're \
        choosing this action.]
        Action: tool_name(argument)
        Observation: [This will be filled in with the tool result]
        
        Continue the Thought/Action/Observation cycle until you have enough information \
        to answer the original question.
        
        When you have a final answer, write:
        Final Answer: [Your complete, detailed answer]
        
        Important rules:
        - Always reason BEFORE acting
        - Base your reasoning on actual observations, not assumptions
        - If a tool call fails, reason about why and try a different approach
        - Be concise in thoughts but thorough in the final answer
        """;
}
```

### 5.2 Java ReAct Agent — Complete Implementation

```java
package com.course.module06.lesson01;

import com.anthropic.client.AnthropicClient;
import com.anthropic.client.okhttp.AnthropicOkHttpClient;
import com.anthropic.models.messages.*;

import java.util.*;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class ReActAgent {

    private final AnthropicClient client;
    private final Map<String, Tool> toolRegistry;
    private static final int MAX_STEPS = 15;

    // Pattern để detect "Final Answer:" trong response text
    private static final Pattern FINAL_ANSWER_PATTERN =
        Pattern.compile("Final Answer:\\s*(.+)", Pattern.DOTALL);

    public ReActAgent(AnthropicClient client) {
        this.client = client;
        this.toolRegistry = buildToolRegistry();
    }

    /**
     * Entry point: chạy ReAct loop cho một task.
     *
     * @param task Mô tả task bằng ngôn ngữ tự nhiên
     * @return Final answer từ agent
     */
    public String run(String task) {
        List<MessageParam> history = new ArrayList<>();
        history.add(buildUserMessage(task));

        System.out.println("=== ReAct Agent Starting ===");
        System.out.println("Task: " + task);
        System.out.println();

        for (int step = 1; step <= MAX_STEPS; step++) {
            System.out.printf("--- Step %d/%d ---%n", step, MAX_STEPS);

            // Gọi Claude với toàn bộ conversation history
            Message response = client.messages().create(
                MessageCreateParams.builder()
                    .model(Model.CLAUDE_SONNET_4_6)
                    .system(ReActPrompts.REACT_SYSTEM_PROMPT)
                    .messages(history)
                    .tools(new ArrayList<>(toolRegistry.values()))
                    .maxTokens(2048)
                    .build()
            );

            // Thêm assistant response vào history
            history.add(buildAssistantMessage(response));

            // Case 1: Claude gọi tools (Action step)
            if (response.stopReason() == StopReason.TOOL_USE) {
                List<MessageParam> toolResults = processToolCalls(response);
                history.addAll(toolResults);
                continue;
            }

            // Case 2: Claude kết thúc với END_TURN — check final answer
            if (response.stopReason() == StopReason.END_TURN) {
                String responseText = extractText(response);
                System.out.println("Claude: " + responseText);

                // Check xem có "Final Answer:" không
                Matcher matcher = FINAL_ANSWER_PATTERN.matcher(responseText);
                if (matcher.find()) {
                    String finalAnswer = matcher.group(1).trim();
                    System.out.println("\n=== Final Answer ===");
                    System.out.println(finalAnswer);
                    return finalAnswer;
                }

                // END_TURN nhưng không có Final Answer — treat as complete
                // (Claude có thể format khác nhau)
                return responseText;
            }

            // Case 3: MAX_TOKENS reached — agent cần tiếp tục
            if (response.stopReason() == StopReason.MAX_TOKENS) {
                System.out.println("Warning: Max tokens reached at step " + step);
                // Có thể continue hoặc summarize tùy use case
                break;
            }
        }

        throw new MaxStepsExceededException(
            "ReAct loop vượt quá " + MAX_STEPS + " bước mà không có final answer. " +
            "Task có thể quá phức tạp hoặc agent bị loop."
        );
    }

    /**
     * Xử lý tất cả tool calls trong một response.
     * Claude có thể gọi nhiều tools trong một turn (parallel tool use).
     */
    private List<MessageParam> processToolCalls(Message response) {
        List<ToolResultBlockParam> toolResults = new ArrayList<>();

        for (ContentBlock block : response.content()) {
            if (!(block instanceof ToolUseBlock toolUse)) continue;

            System.out.printf("Action: %s(%s)%n",
                toolUse.name(),
                toolUse.input());

            String result = executeToolCall(toolUse);
            System.out.println("Observation: " + truncate(result, 500));
            System.out.println();

            toolResults.add(ToolResultBlockParam.builder()
                .toolUseId(toolUse.id())
                .content(result)
                .build());
        }

        // Tool results được wrap vào một user message
        return List.of(MessageParam.builder()
            .role(MessageParam.Role.USER)
            .content(ContentBlockParam.ofToolResult(
                // Nếu có nhiều tool results, wrap tất cả
                toolResults.size() == 1
                    ? toolResults.get(0)
                    : mergeToolResults(toolResults)
            ))
            .build());
    }

    /**
     * Execute một tool call cụ thể.
     * Trong thực tế, đây là nơi bạn dispatch đến actual implementations.
     */
    private String executeToolCall(ToolUseBlock toolUse) {
        Tool tool = toolRegistry.get(toolUse.name());
        if (tool == null) {
            return "Error: Unknown tool '" + toolUse.name() + "'";
        }

        try {
            // Trong production, mỗi tool có implementation riêng
            return tool.execute(toolUse.input());
        } catch (Exception e) {
            return "Error executing tool: " + e.getMessage();
        }
    }

    /**
     * Build tool registry với các tools thực tế.
     * Trong production, inject các services này qua constructor.
     */
    private Map<String, Tool> buildToolRegistry() {
        Map<String, Tool> registry = new HashMap<>();

        registry.put("list_files", input -> {
            String path = (String) input.get("path");
            return FileSystemTool.listFiles(path);
        });

        registry.put("read_file", input -> {
            String path = (String) input.get("path");
            return FileSystemTool.readFile(path);
        });

        registry.put("search_code", input -> {
            String query = (String) input.get("query");
            String path = (String) input.get("path");
            return CodeSearchTool.search(query, path);
        });

        registry.put("run_checkstyle", input -> {
            String file = (String) input.get("file");
            return CheckstyleTool.run(file);
        });

        registry.put("run_tests", input -> {
            String testClass = (String) input.get("test_class");
            return TestRunnerTool.run(testClass);
        });

        return registry;
    }

    // ── Helper methods ───────────────────────────────────────────────────────

    private MessageParam buildUserMessage(String text) {
        return MessageParam.builder()
            .role(MessageParam.Role.USER)
            .content(text)
            .build();
    }

    private MessageParam buildAssistantMessage(Message response) {
        return MessageParam.builder()
            .role(MessageParam.Role.ASSISTANT)
            .content(response.content().stream()
                .map(block -> (ContentBlockParam) block)
                .toList())
            .build();
    }

    private String extractText(Message response) {
        return response.content().stream()
            .filter(b -> b instanceof TextBlock)
            .map(b -> ((TextBlock) b).text())
            .reduce("", (a, b) -> a + b);
    }

    private String truncate(String text, int maxLen) {
        return text.length() > maxLen
            ? text.substring(0, maxLen) + "... [truncated]"
            : text;
    }

    private ToolResultBlockParam mergeToolResults(List<ToolResultBlockParam> results) {
        // Implementation để handle multiple tool results
        // Thực tế cần build proper ContentBlockParam list
        return results.get(0); // Simplified for example
    }

    // ── Functional interface cho Tool ────────────────────────────────────────

    @FunctionalInterface
    interface Tool {
        String execute(Map<String, Object> input) throws Exception;
    }

    // ── Custom exception ──────────────────────────────────────────────────────

    public static class MaxStepsExceededException extends RuntimeException {
        public MaxStepsExceededException(String message) {
            super(message);
        }
    }
}
```

### 5.3 Định nghĩa Tools cho Claude

Tools phải được định nghĩa theo schema mà Anthropic SDK hiểu:

```java
package com.course.module06.lesson01;

import com.anthropic.models.messages.Tool;
import com.anthropic.models.shared.JsonSchema;

import java.util.List;
import java.util.Map;

public class ReActTools {

    public static List<Tool> getTools() {
        return List.of(
            buildListFilesTool(),
            buildReadFileTool(),
            buildSearchCodeTool(),
            buildRunChecksyleTool()
        );
    }

    private static Tool buildListFilesTool() {
        return Tool.builder()
            .name("list_files")
            .description("""
                List all files in a directory. Returns a tree structure of the directory.
                Use this to understand project structure before reading specific files.
                """)
            .inputSchema(JsonSchema.builder()
                .type("object")
                .properties(Map.of(
                    "path", Map.of(
                        "type", "string",
                        "description", "Directory path relative to project root, e.g. 'src/main/java'"
                    )
                ))
                .required(List.of("path"))
                .build())
            .build();
    }

    private static Tool buildReadFileTool() {
        return Tool.builder()
            .name("read_file")
            .description("""
                Read the full content of a Java source file.
                Use this to inspect specific classes, methods, or configurations.
                """)
            .inputSchema(JsonSchema.builder()
                .type("object")
                .properties(Map.of(
                    "path", Map.of(
                        "type", "string",
                        "description", "Full file path, e.g. 'src/main/java/com/example/AuthService.java'"
                    )
                ))
                .required(List.of("path"))
                .build())
            .build();
    }

    private static Tool buildSearchCodeTool() {
        return Tool.builder()
            .name("search_code")
            .description("""
                Search for code patterns using regex across the codebase.
                Useful for finding all usages of a method, class, or pattern.
                """)
            .inputSchema(JsonSchema.builder()
                .type("object")
                .properties(Map.of(
                    "query", Map.of(
                        "type", "string",
                        "description", "Regex pattern to search for"
                    ),
                    "path", Map.of(
                        "type", "string",
                        "description", "Directory to search in (optional, defaults to src/)"
                    )
                ))
                .required(List.of("query"))
                .build())
            .build();
    }

    private static Tool buildRunChecksyleTool() {
        return Tool.builder()
            .name("run_checkstyle")
            .description("""
                Run Checkstyle analysis on a Java file.
                Returns a list of style violations with line numbers and descriptions.
                """)
            .inputSchema(JsonSchema.builder()
                .type("object")
                .properties(Map.of(
                    "file", Map.of(
                        "type", "string",
                        "description", "Path to the Java file to analyze"
                    )
                ))
                .required(List.of("file"))
                .build())
            .build();
    }
}
```

### 5.4 Chạy Agent

```java
package com.course.module06.lesson01;

import com.anthropic.client.AnthropicClient;
import com.anthropic.client.okhttp.AnthropicOkHttpClient;

public class ReActDemo {

    public static void main(String[] args) {
        AnthropicClient client = AnthropicOkHttpClient.builder()
            .apiKey(System.getenv("ANTHROPIC_API_KEY"))
            .build();

        ReActAgent agent = new ReActAgent(client);

        String task = """
            Analyze the Spring Boot application in src/main/java and identify:
            1. Any potential security vulnerabilities (SQL injection, hardcoded secrets, etc.)
            2. Classes that violate Checkstyle rules
            3. Missing unit tests (look for Service classes without corresponding Test classes)
            
            Provide a prioritized list of findings.
            """;

        String result = agent.run(task);
        System.out.println("\nFinal Result:\n" + result);
    }
}
```

---

## 6. Ví dụ trace thực tế

Dưới đây là ví dụ trace của ReAct agent khi phân tích một Spring Boot app:

```
=== ReAct Agent Starting ===
Task: Analyze the Spring Boot application...

--- Step 1/15 ---
Thought: Trước tiên tôi cần hiểu cấu trúc project để biết các module chính.
Action: list_files(src/main/java)
Observation: src/main/java/
├── com/example/
│   ├── controller/
│   │   ├── AuthController.java
│   │   ├── UserController.java
│   │   └── PaymentController.java
│   ├── service/
│   │   ├── AuthService.java
│   │   ├── UserService.java
│   │   └── PaymentService.java
│   └── repository/
│       ├── UserRepository.java
│       └── PaymentRepository.java

--- Step 2/15 ---
Thought: PaymentController và AuthController là priority cao nhất về security.
         Bắt đầu với AuthController vì auth thường là attack surface lớn nhất.
Action: read_file(src/main/java/com/example/controller/AuthController.java)
Observation: @RestController
             public class AuthController {
                 @PostMapping("/login")
                 public ResponseEntity<String> login(@RequestBody LoginRequest req) {
                     String query = "SELECT * FROM users WHERE email='" 
                         + req.getEmail() + "'";  // <-- SQL concatenation!
                     ...
                 }
             }

--- Step 3/15 ---
Thought: Phát hiện SQL string concatenation trong AuthController — đây là SQL injection vulnerability.
         Cần check thêm UserRepository để xem pattern này có lặp lại không.
Action: search_code(SELECT.*\+.*req\., src/main/java)
Observation: Found 3 matches:
             - AuthController.java:23: ... + req.getEmail() + ...
             - UserController.java:45: ... + req.getUsername() + ...
             - PaymentRepository.java:67: ... + amount + ...

--- Step 4/15 ---
Thought: 3 SQL injection vulnerabilities. Tiếp theo check hardcoded secrets.
Action: search_code((password|secret|apiKey|token)\s*=\s*"[^"]+", src/main/java)
Observation: Found 1 match:
             - AuthService.java:12: private String jwtSecret = "my-super-secret-123";

... [vài steps nữa] ...

Final Answer: 
## Security Analysis Report

### Critical Issues (Fix Immediately)
1. **SQL Injection** (3 files): AuthController.java:23, UserController.java:45, 
   PaymentRepository.java:67 — đang dùng string concatenation thay vì PreparedStatement
   
2. **Hardcoded JWT Secret** (AuthService.java:12) — secret nằm trong source code,
   cần chuyển sang environment variable

### Missing Tests
- PaymentService — không có PaymentServiceTest
- AuthService — không có AuthServiceTest
```

---

## 7. Stopping Conditions — Quan trọng

Một trong những lỗi phổ biến nhất khi implement ReAct là **quên stopping conditions**. Agent có thể bị loop vô tận hoặc chạy quá nhiều steps tốn tiền.

### 4 stopping conditions cần implement:

```java
public enum StopReason {
    FINAL_ANSWER,       // Claude tự nói "Final Answer:"
    END_TURN_NO_TOOLS,  // Claude không gọi tool, đã done
    MAX_STEPS,          // Vượt quá giới hạn steps
    ERROR               // Unrecoverable error
}

// Trong run loop:
private StopReason checkStopCondition(Message response, String text, int step) {
    // 1. Final Answer pattern trong text
    if (FINAL_ANSWER_PATTERN.matcher(text).find()) {
        return StopReason.FINAL_ANSWER;
    }

    // 2. END_TURN không có tool calls = Claude đã done
    if (response.stopReason() == StopReason.END_TURN
            && response.content().stream().noneMatch(b -> b instanceof ToolUseBlock)) {
        return StopReason.END_TURN_NO_TOOLS;
    }

    // 3. Max steps guard
    if (step >= MAX_STEPS) {
        return StopReason.MAX_STEPS;
    }

    return null; // Continue
}
```

### MAX_STEPS phù hợp là bao nhiêu?

| Task Type | Recommended MAX_STEPS |
|-----------|----------------------|
| Simple lookup | 3-5 |
| Code analysis | 10-15 |
| Complex refactoring | 15-20 |
| Research task | 20-30 |

Đặt MAX_STEPS quá thấp → agent không hoàn thành task.
Đặt quá cao → waste tokens, tốn tiền, slow.

---

## 8. Common Mistakes và Cách Sửa

### Mistake 1: Không thêm Observations vào history

```java
// WRONG — agent không nhớ kết quả tool calls
history.add(buildUserMessage(task));
Message response = callClaude(history);
// Chỉ thêm assistant response, quên tool results!
history.add(buildAssistantMessage(response));

// CORRECT
history.add(buildUserMessage(task));
Message response = callClaude(history);
history.add(buildAssistantMessage(response));
history.addAll(processToolCalls(response)); // <- Phải có dòng này
```

### Mistake 2: System prompt không rõ ràng về format

```
// WRONG — Claude không biết khi nào dùng "Final Answer:"
"You are a helpful assistant that uses tools to answer questions."

// CORRECT — explicit về format và stopping condition
"... When you have enough information, write:
Final Answer: [your complete answer]
Do NOT use 'Final Answer' until you are certain you have all needed information."
```

### Mistake 3: Không handle tool errors

```java
// WRONG — exception sẽ crash toàn bộ agent
private String executeToolCall(ToolUseBlock toolUse) {
    return toolRegistry.get(toolUse.name()).execute(toolUse.input()); // NPE nếu tool không tồn tại
}

// CORRECT — always return a string, never throw
private String executeToolCall(ToolUseBlock toolUse) {
    Tool tool = toolRegistry.get(toolUse.name());
    if (tool == null) {
        return "Error: Tool '" + toolUse.name() + "' not found. Available tools: "
            + String.join(", ", toolRegistry.keySet());
    }
    try {
        return tool.execute(toolUse.input());
    } catch (Exception e) {
        return "Tool execution failed: " + e.getMessage()
            + ". Please try a different approach.";
    }
}
```

### Mistake 4: Quá nhiều tools

Đưa cho Claude quá nhiều tools (> 20) làm giảm chất lượng quyết định. Nguyên tắc:
- <= 10 tools cho một ReAct agent
- Nhóm các tools liên quan thành một tool với sub-actions nếu cần

---

## 9. Khi nào KHÔNG dùng ReAct

ReAct không phải silver bullet. Đừng dùng khi:

1. **Task đơn giản, không cần nhiều bước**: "Translate this text" → gọi thẳng API
2. **Sequence of tools đã biết trước**: Dùng hardcoded pipeline thay vì ReAct
3. **Latency là priority tối cao**: Mỗi ReAct step = 1 API call → latency tích lũy
4. **Budget rất thấp**: ReAct tốn nhiều tokens hơn direct call
5. **Deterministic output cần**: ReAct có thể take different paths mỗi lần

---

## 10. Exercise

### Bài tập: Spring Boot Diagnostic Agent

Implement một ReAct agent để diagnose vấn đề trong Spring Boot application. Agent cần:

**Tools cần implement:**
- `check_health(endpoint)` — gọi Spring Actuator health endpoint
- `read_logs(service, lines)` — đọc N dòng cuối của log file
- `check_db_connection(datasource)` — test database connectivity
- `list_beans(filter)` — list Spring beans matching filter

**Task để test:**
```
"Our Spring Boot app is returning 503 errors intermittently.
 The database is PostgreSQL. Diagnose the root cause."
```

**Expected behavior của agent:**
1. Check /actuator/health → thấy DB connection pool exhausted
2. Read recent logs → thấy timeout errors
3. Check DB connection pool settings → thấy max pool size quá nhỏ
4. Final Answer: đề xuất tăng `spring.datasource.hikari.maximum-pool-size`

**Checklist:**
- [ ] Agent có Thought/Action/Observation format đúng
- [ ] Tối đa 10 steps
- [ ] Tool errors không crash agent
- [ ] Final Answer rõ ràng và actionable
- [ ] Viết unit test cho ReActAgent với mock tools

---

## Tóm tắt

| Khái niệm | Tóm tắt |
|-----------|---------|
| **ReAct** | Interleave reasoning và acting trong một loop |
| **Core loop** | Thought → Action → Observation → repeat |
| **Vs CoT** | CoT chỉ reason, ReAct reason + act |
| **Vs simple tool use** | Simple = one-shot, ReAct = adaptive multi-step |
| **Stopping conditions** | Final Answer, END_TURN, MAX_STEPS, error |
| **Common mistakes** | Quên observations, vague system prompt, no error handling |
| **Khi không dùng** | Simple tasks, known sequences, latency-critical |

---

> **Tiếp theo**: [Lesson 02 — Planning Patterns](02-planning-patterns.md)
