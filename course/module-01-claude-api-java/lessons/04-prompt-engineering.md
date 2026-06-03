# Lesson 04: Prompt Engineering cho Engineers

> **Thời lượng**: 3 buổi (~7 giờ)  
> **Mục tiêu**: Nắm vững các kỹ thuật prompt engineering, viết prompts hiệu quả cho Java backend use cases, xây dựng hệ thống test prompt có hệ thống

---

## 1. Tại sao Prompt Engineering quan trọng với Engineers

### 1.1 Không có compiler, không có type system

Với Java code, khi bạn viết sai:
```java
String name = 42; // Compiler error: incompatible types
```
Compiler bắt lỗi ngay. Với LLM prompts:
```
"Analyze the code and tell me things about it"
```
Không có lỗi nào — Claude sẽ trả lời, nhưng output có thể không phải những gì bạn cần.

**Prompt engineering là kỹ năng viết "specifications" mà LLM hiểu và thực thi đúng.**

### 1.2 So sánh với software engineering

| Software Engineering | Prompt Engineering |
|---------------------|-------------------|
| Function signature | Tool description |
| Type annotations | Output format spec |
| Unit tests | Eval test cases |
| Code review | Prompt review |
| Refactoring | Prompt iteration |
| Edge case handling | Few-shot examples cho edge cases |

### 1.3 Chi phí của prompt tệ

```
Prompt tệ → Output kém → User unhappy → Tăng maxTokens để "fix" → Tốn tiền hơn
Prompt tốt → Output tốt → User happy → Cost optimal
```

Ví dụ thực tế: prompt vague vs specific cho code review:

```
// VAGUE — 800 tokens output, 60% relevant
"Review my code"

// SPECIFIC — 400 tokens output, 95% relevant
"Review this Java method for: (1) null pointer risks, (2) performance issues.
Output format: list of issues with severity HIGH/MEDIUM/LOW and line number."
```

---

## 2. Core Techniques

### 2.1 System Prompt Design

System prompt là nền tảng. Cấu trúc tốt nhất cho Java backend services:

```java
public class SystemPromptTemplates {

    /**
     * Template đầy đủ cho một production system prompt
     */
    public static String buildSystemPrompt(SystemPromptConfig config) {
        return String.format("""
            ## Role
            %s

            ## Context
            %s

            ## Capabilities
            You have access to the following tools:
            %s

            ## Constraints
            %s

            ## Output Format
            %s

            ## Examples
            %s
            """,
            config.role(),
            config.context(),
            config.toolDescriptions(),
            config.constraints(),
            config.outputFormat(),
            config.examples()
        );
    }

    /**
     * Ví dụ: Code Review Assistant
     */
    public static String codeReviewerPrompt() {
        return """
            ## Role
            You are a Senior Java Engineer with 10+ years experience in Spring Boot,
            microservices, and enterprise systems. You perform thorough, constructive code reviews.

            ## Review Criteria
            Review code for the following, in priority order:
            1. CRITICAL: Security vulnerabilities (SQL injection, XSS, auth bypass)
            2. CRITICAL: Data loss risks (missing transactions, race conditions)
            3. HIGH: Logic errors that cause incorrect behavior
            4. HIGH: Performance issues (N+1 queries, missing indexes, memory leaks)
            5. MEDIUM: Code quality (readability, naming, complexity)
            6. LOW: Style and formatting

            ## Output Format
            For each issue found:
            ```
            [SEVERITY] Line X: <issue description>
            Problem: <what is wrong and why>
            Fix: <specific suggestion>
            ```

            End with a summary: "Overall: APPROVE / REQUEST_CHANGES / NEEDS_DISCUSSION"

            ## Constraints
            - Only comment on the code provided, not hypothetical issues
            - If code is good, say so explicitly — don't fabricate issues
            - Keep each issue description under 3 sentences
            """;
    }

    /**
     * Ví dụ: SQL Generator
     */
    public static String sqlGeneratorPrompt(String schema) {
        return """
            ## Role
            You are a PostgreSQL expert. Generate safe, optimized SQL queries.

            ## Database Schema
            """ + schema + """

            ## Rules
            - Only generate SELECT queries (never INSERT/UPDATE/DELETE)
            - Always include appropriate WHERE clauses to avoid full table scans
            - Add LIMIT clause (max 1000 rows) unless user explicitly needs all rows
            - Use CTEs for complex queries instead of nested subqueries
            - Include comments explaining non-obvious query logic

            ## Output Format
            ```sql
            -- Comment explaining the query
            SELECT ...
            ```
            Then: brief explanation of what the query does and any assumptions made.

            ## Constraints
            - If the request is ambiguous, ask for clarification before generating
            - If the request requires data not in the schema, explain what's missing
            """;
    }
}
```

### 2.2 Few-Shot Prompting — Examples thay vì Rules

Few-shot là kỹ thuật **mạnh nhất** và thường bị bỏ qua. Thay vì giải thích rules dài dòng, cho Claude thấy examples.

```java
public class FewShotExamples {

    /**
     * BAD: Giải thích rules dài dòng
     */
    public static String rulesOnlyPrompt() {
        return """
            Classify customer support tickets as BILLING, TECHNICAL, SHIPPING, or OTHER.
            BILLING tickets are about payments, invoices, charges, refunds, subscriptions.
            TECHNICAL tickets are about bugs, errors, features not working, crashes.
            SHIPPING tickets are about delivery, tracking, lost packages, delays.
            OTHER is for everything else.
            Output only the category name.
            """;
    }

    /**
     * GOOD: Few-shot examples — Claude học pattern từ examples
     */
    public static String fewShotPrompt() {
        return """
            Classify customer support tickets. Output only the category name.
            Categories: BILLING, TECHNICAL, SHIPPING, OTHER

            Examples:
            Ticket: "I was charged twice for my subscription last month"
            Category: BILLING

            Ticket: "The app crashes when I try to upload a file larger than 5MB"
            Category: TECHNICAL

            Ticket: "My order was supposed to arrive 3 days ago but still no sign of it"
            Category: SHIPPING

            Ticket: "How do I change my account username?"
            Category: OTHER

            Ticket: "I need a receipt for my last purchase for tax purposes"
            Category: BILLING

            Ticket: "The dark mode toggle doesn't save my preference after logging out"
            Category: TECHNICAL

            Now classify this ticket:
            Ticket: "{ticket_text}"
            Category:
            """;
    }

    /**
     * Sử dụng trong Java
     */
    public static String classifyTicket(AnthropicClient client, String ticketText) {
        String prompt = fewShotPrompt().replace("{ticket_text}", ticketText);

        Message response = client.messages().create(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_SONNET_4_6)
                .maxTokens(20)  // Chỉ cần 1 word — tiết kiệm cost
                .addUserMessage(prompt)
                .build()
        );

        return extractText(response).trim();
    }
}
```

**Số examples tối ưu**: 3-5 examples thường đủ. Tăng thêm khi:
- Task phức tạp có nhiều edge cases
- Output format phức tạp (JSON với nhiều fields)
- Domain-specific terminology

### 2.3 Chain-of-Thought (CoT) — Bắt Claude "suy nghĩ"

CoT cải thiện độ chính xác cho tasks cần reasoning. Đặc biệt hiệu quả với:
- Toán học / logic
- Code debugging
- Multi-step analysis
- Decisions với nhiều factors

```java
public class ChainOfThoughtExamples {

    /**
     * WITHOUT CoT — kết quả kém hơn cho complex tasks
     */
    public static String withoutCoT(String bugReport) {
        return String.format("""
            Diagnose this Java exception and provide the fix:
            %s
            """, bugReport);
    }

    /**
     * WITH CoT — kết quả tốt hơn đáng kể
     */
    public static String withCoT(String bugReport) {
        return String.format("""
            Diagnose this Java exception and provide the fix.

            Think through this step by step:
            1. What type of exception is this? What does it typically mean?
            2. What is the stack trace telling us? Which line caused the issue?
            3. What was the code trying to do at that point?
            4. What are the possible root causes?
            5. Which root cause is most likely given the context?
            6. What is the specific fix?

            Exception and stack trace:
            %s
            """, bugReport);
    }

    /**
     * Structured CoT với XML tags cho complex analysis
     */
    public static String structuredCoT(String codeToAnalyze) {
        return String.format("""
            Analyze this Java code for security vulnerabilities.

            <thinking>
            Work through each section systematically:
            1. Input validation — are all user inputs validated?
            2. Authentication/authorization — are protected resources checked?
            3. SQL/injection risks — are queries parameterized?
            4. Sensitive data — is any PII/password logged or exposed?
            5. Dependencies — any known vulnerable patterns?
            </thinking>

            After your analysis, provide:
            <findings>
            List each vulnerability with severity and location
            </findings>

            <recommendations>
            Specific fixes for each finding
            </recommendations>

            Code to analyze:
            ```java
            %s
            ```
            """, codeToAnalyze);
    }
}
```

### 2.4 XML Tags — Structure rõ ràng cho Complex Prompts

XML tags giúp Claude phân biệt rõ các phần của prompt, đặc biệt khi có nhiều context:

```java
public class XmlTagExamples {

    public static String documentAnalysisPrompt(
            String userQuestion,
            String documentContent,
            String existingAnalysis) {

        return String.format("""
            <task>
            Answer the user's question about the provided document.
            Build on the existing analysis if relevant.
            </task>

            <document>
            %s
            </document>

            <existing_analysis>
            %s
            </existing_analysis>

            <question>
            %s
            </question>

            <instructions>
            - Answer based on the document content only
            - If the answer is not in the document, say "Not found in document"
            - Reference specific sections when possible
            - Be concise — aim for 2-3 sentences unless detail is requested
            </instructions>
            """,
            documentContent,
            existingAnalysis,
            userQuestion
        );
    }

    /**
     * Tại sao XML tags tốt hơn plain text:
     *
     * 1. Claude được train để hiểu XML-like structure
     * 2. Rõ ràng ranh giới giữa các phần
     * 3. Tránh nhầm lẫn khi content có special characters
     * 4. Dễ đọc và maintain hơn
     */
}
```

### 2.5 Structured Output — Force JSON/XML

Đây là kỹ thuật cực quan trọng cho backend engineers. Cần parse Claude output thành Java objects.

```java
public class StructuredOutputExamples {

    private static final ObjectMapper mapper = new ObjectMapper();

    /**
     * Technique 1: JSON schema trong system prompt
     */
    public static String forceJsonSystemPrompt() {
        return """
            You always respond with valid JSON only. No markdown, no explanation outside JSON.

            Response schema:
            {
              "analysis": {
                "summary": "string — one sentence summary",
                "issues": [
                  {
                    "type": "string — one of: bug, performance, security, style",
                    "severity": "string — one of: critical, high, medium, low",
                    "line": "integer or null",
                    "description": "string",
                    "suggestion": "string"
                  }
                ],
                "overall_verdict": "string — one of: approve, request_changes, major_rework",
                "confidence": "string — one of: high, medium, low"
              }
            }
            """;
    }

    /**
     * Technique 2: Example JSON trong user message
     */
    public static String jsonExampleInPrompt(String javaCode) {
        return String.format("""
            Analyze this Java code and return a JSON response.

            Example response format:
            {"analysis": {"summary": "...", "issues": [{"type": "bug", "severity": "high", "line": 42, "description": "...", "suggestion": "..."}], "overall_verdict": "request_changes"}}

            Code to analyze:
            ```java
            %s
            ```

            Return JSON only, no other text:
            """, javaCode);
    }

    /**
     * Parse và validate JSON response
     */
    public static CodeAnalysis parseAnalysisResponse(String jsonResponse) {
        // Clean response (đôi khi Claude thêm markdown code block)
        String cleaned = jsonResponse.trim()
            .replaceAll("^```json\\s*", "")
            .replaceAll("^```\\s*", "")
            .replaceAll("\\s*```$", "")
            .trim();

        try {
            JsonNode root = mapper.readTree(cleaned);
            JsonNode analysis = root.get("analysis");

            if (analysis == null) {
                throw new IllegalArgumentException("Missing 'analysis' field in response");
            }

            // Map to Java record
            return new CodeAnalysis(
                analysis.get("summary").asText(),
                parseIssues(analysis.get("issues")),
                analysis.get("overall_verdict").asText(),
                analysis.get("confidence").asText()
            );
        } catch (Exception e) {
            throw new RuntimeException("Failed to parse Claude response as JSON: " + e.getMessage()
                + "\nRaw response: " + jsonResponse, e);
        }
    }

    private static List<Issue> parseIssues(JsonNode issuesNode) {
        List<Issue> issues = new ArrayList<>();
        if (issuesNode != null && issuesNode.isArray()) {
            for (JsonNode issue : issuesNode) {
                issues.add(new Issue(
                    issue.get("type").asText(),
                    issue.get("severity").asText(),
                    issue.has("line") && !issue.get("line").isNull()
                        ? issue.get("line").asInt() : null,
                    issue.get("description").asText(),
                    issue.get("suggestion").asText()
                ));
            }
        }
        return issues;
    }

    record CodeAnalysis(String summary, List<Issue> issues,
                        String overallVerdict, String confidence) {}
    record Issue(String type, String severity, Integer line,
                 String description, String suggestion) {}
}
```

**Kỹ thuật nâng cao: Validate JSON trước khi parse**

```java
public class SafeJsonExtractor {

    /**
     * Retry với corrective prompt nếu Claude không trả JSON hợp lệ
     */
    public static <T> T extractWithRetry(
            AnthropicClient client,
            String prompt,
            Class<T> targetClass,
            int maxRetries) {

        String response = null;
        for (int attempt = 1; attempt <= maxRetries; attempt++) {
            try {
                String actualPrompt = attempt == 1 ? prompt
                    : prompt + "\n\nIMPORTANT: Your previous response was not valid JSON. "
                    + "Return ONLY valid JSON matching the schema. No other text.";

                Message message = client.messages().create(
                    MessageCreateParams.builder()
                        .model(Model.CLAUDE_SONNET_4_6)
                        .maxTokens(2048)
                        .addUserMessage(actualPrompt)
                        .build()
                );

                response = extractText(message);
                String cleaned = cleanJson(response);
                return mapper.readValue(cleaned, targetClass);

            } catch (Exception e) {
                if (attempt == maxRetries) {
                    throw new RuntimeException(
                        "Failed to extract valid JSON after " + maxRetries + " attempts. "
                        + "Last response: " + response, e);
                }
                log.warn("Attempt {}/{} failed: {}", attempt, maxRetries, e.getMessage());
            }
        }
        throw new IllegalStateException("Should not reach here");
    }

    private static String cleanJson(String raw) {
        return raw.trim()
            .replaceAll("(?s)^```(?:json)?\\s*", "")
            .replaceAll("\\s*```$", "")
            .trim();
    }
}
```

---

## 3. Common Mistakes của Engineers Mới

### 3.1 Over-explaining thay vì giving examples

```java
// ❌ BAD: Giải thích dài dòng
String badPrompt = """
    When I ask you to generate a Java method, I want you to create a method
    that has proper Javadoc with @param and @return tags. The method should
    follow Java naming conventions where method names start with a verb and
    use camelCase. Parameters should also be in camelCase. The method body
    should handle null inputs and throw appropriate exceptions. Error messages
    should be descriptive. You should also add inline comments for complex logic.
    Generate a method to calculate compound interest.
    """;

// ✅ GOOD: Examples là tốt hơn
String goodPrompt = """
    Generate a Java method following this pattern:

    /**
     * Calculates the final amount after simple interest.
     *
     * @param principal the initial amount (must be > 0)
     * @param rate annual interest rate as decimal (e.g., 0.05 for 5%)
     * @param years number of years (must be > 0)
     * @return final amount after interest
     * @throws IllegalArgumentException if any parameter is invalid
     */
    public static double calculateSimpleInterest(double principal, double rate, int years) {
        if (principal <= 0) throw new IllegalArgumentException("Principal must be positive");
        if (rate < 0) throw new IllegalArgumentException("Rate cannot be negative");
        if (years <= 0) throw new IllegalArgumentException("Years must be positive");
        return principal * (1 + rate * years);
    }

    Now generate a similar method for COMPOUND interest.
    """;
```

### 3.2 Vague instructions

```java
// ❌ BAD: Vague
"Improve this code"
"Make it better"
"Fix the issues"

// ✅ GOOD: Specific
"Refactor this method to: (1) extract the validation logic into a separate method, "
+ "(2) replace the for loop with Stream API, (3) add null checks for all parameters. "
+ "Keep the same method signature."
```

### 3.3 Missing output format specification

```java
// ❌ BAD: Không specify format → nhận random format
"List the design patterns used in this code"
// Output có thể là: essay, bullet points, numbered list, table...

// ✅ GOOD: Specify format rõ ràng
"""
Identify design patterns in this code.
Format your response as:
- Pattern Name (e.g., "Singleton"): One sentence explanation of where/how it's used
List only patterns that are clearly present, not hypothetical ones.
"""
```

### 3.4 Không handle edge cases

```java
// ❌ BAD: Không nói Claude làm gì khi không có data
"Summarize the customer's recent orders"

// ✅ GOOD: Handle edge cases
"""
Summarize the customer's recent orders.
- If there are orders: provide count, total value, and most recent order date
- If there are no orders: say "No orders found for this customer"
- If the order data is incomplete: summarize what's available and note what's missing
"""
```

### 3.5 Prompt quá dài không cần thiết

```java
// ❌ BAD: 500 từ instructions cho task đơn giản
// → Tốn tokens, đôi khi Claude "mất" các instructions ở giữa

// ✅ GOOD: Ngắn gọn, đủ ý
// Rule: Nếu task đơn giản, prompt đơn giản. Chỉ thêm complexity khi cần.
"Classify this Java exception as: NullPointer / IllegalArgument / IO / Other\nException: {exception}"
// → 2 dòng, rõ ràng, hiệu quả
```

---

## 4. Prompt Templates trong Java

### 4.1 Simple Parameterized Templates

```java
public class PromptTemplate {

    private final String template;

    public PromptTemplate(String template) {
        this.template = template;
    }

    /**
     * Fill template với named parameters
     * Template syntax: {paramName}
     */
    public String fill(Map<String, String> params) {
        String result = template;
        for (Map.Entry<String, String> entry : params.entrySet()) {
            result = result.replace("{" + entry.getKey() + "}", entry.getValue());
        }
        // Validate không còn unfilled placeholders
        if (result.contains("{") && result.contains("}")) {
            // Find unfilled params
            java.util.regex.Matcher matcher =
                java.util.regex.Pattern.compile("\\{(\\w+)\\}").matcher(result);
            List<String> unfilled = new ArrayList<>();
            while (matcher.find()) unfilled.add(matcher.group(1));
            if (!unfilled.isEmpty()) {
                throw new IllegalArgumentException("Unfilled template params: " + unfilled);
            }
        }
        return result;
    }

    // Convenience method cho single param
    public String fill(String paramName, String value) {
        return fill(Map.of(paramName, value));
    }
}

// Usage
public class JavaBackendPrompts {

    public static final PromptTemplate CODE_REVIEW = new PromptTemplate("""
        Review this {language} code for bugs, performance issues, and security vulnerabilities.

        Context: {context}

        Code:
        ```{language}
        {code}
        ```

        Focus on: {focus_areas}
        Output format: bullet points grouped by severity (CRITICAL/HIGH/MEDIUM/LOW)
        """);

    public static final PromptTemplate SQL_GENERATOR = new PromptTemplate("""
        Generate a PostgreSQL SELECT query for the following request.

        Database schema:
        {schema}

        Request: {request}

        Constraints:
        - Include LIMIT {max_rows}
        - Only use tables and columns that exist in the schema above
        """);

    public static final PromptTemplate ERROR_DIAGNOSIS = new PromptTemplate("""
        Diagnose this Java exception and provide a fix.

        Application context: {app_context}

        Exception:
        {exception}

        Stack trace:
        {stack_trace}

        Steps to diagnose:
        1. Identify the exception type and its meaning
        2. Find the root cause line in the stack trace
        3. Explain why this happens
        4. Provide the specific fix with code example
        """);
}

// Usage example
String reviewPrompt = JavaBackendPrompts.CODE_REVIEW.fill(Map.of(
    "language", "Java",
    "context", "Spring Boot REST controller for user management",
    "code", userControllerCode,
    "focus_areas", "null pointer risks, SQL injection, authentication"
));
```

### 4.2 Builder Pattern cho Complex Prompts

```java
public class PromptBuilder {

    private String role;
    private String context;
    private final List<String> instructions = new ArrayList<>();
    private final List<String[]> examples = new ArrayList<>(); // [input, output]
    private String outputFormat;
    private String task;

    public PromptBuilder role(String role) {
        this.role = role;
        return this;
    }

    public PromptBuilder context(String context) {
        this.context = context;
        return this;
    }

    public PromptBuilder instruction(String instruction) {
        this.instructions.add(instruction);
        return this;
    }

    public PromptBuilder example(String input, String output) {
        this.examples.add(new String[]{input, output});
        return this;
    }

    public PromptBuilder outputFormat(String format) {
        this.outputFormat = format;
        return this;
    }

    public PromptBuilder task(String task) {
        this.task = task;
        return this;
    }

    public String build() {
        StringBuilder sb = new StringBuilder();

        if (role != null) {
            sb.append("You are: ").append(role).append("\n\n");
        }

        if (context != null) {
            sb.append("Context:\n").append(context).append("\n\n");
        }

        if (!instructions.isEmpty()) {
            sb.append("Instructions:\n");
            for (int i = 0; i < instructions.size(); i++) {
                sb.append(i + 1).append(". ").append(instructions.get(i)).append("\n");
            }
            sb.append("\n");
        }

        if (!examples.isEmpty()) {
            sb.append("Examples:\n");
            for (String[] example : examples) {
                sb.append("Input: ").append(example[0]).append("\n");
                sb.append("Output: ").append(example[1]).append("\n\n");
            }
        }

        if (outputFormat != null) {
            sb.append("Output format:\n").append(outputFormat).append("\n\n");
        }

        if (task != null) {
            sb.append("Task:\n").append(task);
        }

        return sb.toString();
    }
}

// Usage
String prompt = new PromptBuilder()
    .role("Senior Java Backend Engineer specializing in Spring Boot")
    .context("We are migrating from Java 8 to Java 17")
    .instruction("Identify Java 8 patterns that have better Java 17 equivalents")
    .instruction("Show the before/after code transformation")
    .instruction("Explain why the Java 17 approach is better")
    .example(
        "for(String s : list) { System.out.println(s); }",
        "list.forEach(System.out::println); // Method reference, more concise"
    )
    .example(
        "if(obj != null) { return obj.getValue(); } else { return defaultValue; }",
        "return Optional.ofNullable(obj).map(Obj::getValue).orElse(defaultValue);"
    )
    .outputFormat("For each pattern: [Java 8 code] → [Java 17 code] + explanation")
    .task(legacyJavaCode)
    .build();
```

---

## 5. Testing Prompts — Systematic Approach

### 5.1 Tại sao cần test prompts

Prompt thay đổi → output thay đổi. Không có test → không biết prompt có regress không.

```java
/**
 * Eval framework đơn giản cho prompt testing
 */
public class PromptEvaluator {

    private final AnthropicClient client;

    record EvalCase(String name, String input, String expectedPattern,
                    EvalType evalType) {}

    enum EvalType {
        EXACT_MATCH,     // Output phải match exactly
        CONTAINS,        // Output phải chứa string này
        REGEX,           // Output phải match regex
        JSON_VALID,      // Output phải là valid JSON
        JSON_SCHEMA,     // Output JSON phải có fields này
        MANUAL_REVIEW    // Cần human review
    }

    record EvalResult(String caseName, boolean passed, String actualOutput,
                      String failReason) {}

    public List<EvalResult> evaluate(
            String systemPrompt,
            List<EvalCase> testCases) {

        List<EvalResult> results = new ArrayList<>();

        for (EvalCase testCase : testCases) {
            System.out.printf("Running: %s... ", testCase.name());

            Message response = client.messages().create(
                MessageCreateParams.builder()
                    .model(Model.CLAUDE_SONNET_4_6)
                    .maxTokens(1024)
                    .system(systemPrompt)
                    .addUserMessage(testCase.input())
                    .build()
            );

            String output = extractText(response);
            EvalResult result = checkOutput(testCase, output);
            results.add(result);

            System.out.println(result.passed() ? "PASS" : "FAIL: " + result.failReason());
        }

        // Summary
        long passed = results.stream().filter(EvalResult::passed).count();
        System.out.printf("%nResults: %d/%d passed (%.0f%%)%n",
            passed, results.size(), (double) passed / results.size() * 100);

        return results;
    }

    private EvalResult checkOutput(EvalCase testCase, String output) {
        return switch (testCase.evalType()) {
            case EXACT_MATCH -> {
                boolean pass = output.trim().equals(testCase.expectedPattern());
                yield new EvalResult(testCase.name(), pass, output,
                    pass ? null : "Expected: " + testCase.expectedPattern());
            }
            case CONTAINS -> {
                boolean pass = output.contains(testCase.expectedPattern());
                yield new EvalResult(testCase.name(), pass, output,
                    pass ? null : "Missing: " + testCase.expectedPattern());
            }
            case JSON_VALID -> {
                try {
                    new ObjectMapper().readTree(output);
                    yield new EvalResult(testCase.name(), true, output, null);
                } catch (Exception e) {
                    yield new EvalResult(testCase.name(), false, output,
                        "Invalid JSON: " + e.getMessage());
                }
            }
            case REGEX -> {
                boolean pass = output.matches("(?s).*" + testCase.expectedPattern() + ".*");
                yield new EvalResult(testCase.name(), pass, output,
                    pass ? null : "Pattern not found: " + testCase.expectedPattern());
            }
            default -> new EvalResult(testCase.name(), true, output, "Manual review needed");
        };
    }
}
```

### 5.2 Test Cases cho Java Backend Prompts

```java
public class PromptTestSuite {

    public static void main(String[] args) {
        AnthropicClient client = AnthropicClient.builder()
            .apiKey(System.getenv("ANTHROPIC_API_KEY"))
            .build();

        PromptEvaluator evaluator = new PromptEvaluator(client);

        // Test: Ticket Classifier
        List<PromptEvaluator.EvalCase> classifierTests = List.of(
            new PromptEvaluator.EvalCase(
                "billing_charge_double",
                "I was charged twice for my subscription",
                "BILLING",
                PromptEvaluator.EvalType.CONTAINS
            ),
            new PromptEvaluator.EvalCase(
                "technical_crash",
                "App crashes when uploading files",
                "TECHNICAL",
                PromptEvaluator.EvalType.CONTAINS
            ),
            new PromptEvaluator.EvalCase(
                "shipping_late",
                "Package hasn't arrived after 2 weeks",
                "SHIPPING",
                PromptEvaluator.EvalType.CONTAINS
            ),
            new PromptEvaluator.EvalCase(
                "edge_case_empty",
                "",  // Empty input — should not crash
                "OTHER",
                PromptEvaluator.EvalType.CONTAINS
            ),
            new PromptEvaluator.EvalCase(
                "edge_case_ambiguous",
                "My money is gone",  // Could be billing or shipping
                "(BILLING|SHIPPING)",  // Either is acceptable
                PromptEvaluator.EvalType.REGEX
            )
        );

        System.out.println("=== Ticket Classifier Eval ===");
        evaluator.evaluate(FewShotExamples.fewShotPrompt(), classifierTests);

        // Test: JSON Output
        List<PromptEvaluator.EvalCase> jsonTests = List.of(
            new PromptEvaluator.EvalCase(
                "returns_valid_json",
                "Analyze: public void foo() {}",
                null,
                PromptEvaluator.EvalType.JSON_VALID
            ),
            new PromptEvaluator.EvalCase(
                "contains_severity_field",
                "Analyze: String x = null; x.length();",
                "\"severity\"",
                PromptEvaluator.EvalType.CONTAINS
            )
        );

        System.out.println("\n=== JSON Output Eval ===");
        evaluator.evaluate(StructuredOutputExamples.forceJsonSystemPrompt(), jsonTests);
    }
}
```

---

## 6. Exercise: 3 Prompts cho Java Backend Use Cases

### Exercise 6.1: Code Review Prompt

Viết prompt cho automated code review. Requirements:
- Input: Java method code
- Output: JSON với issues array
- Test với ít nhất 3 test cases: code tốt, code có bug, code có security issue

```java
// Skeleton
public class CodeReviewPromptExercise {

    // TODO: Viết systemPrompt cho code reviewer
    static String CODE_REVIEW_SYSTEM_PROMPT = """
        // TODO: Your prompt here
        // Phải include:
        // 1. Role definition
        // 2. Review criteria (CRITICAL/HIGH/MEDIUM/LOW)
        // 3. JSON output schema với fields: issues[], overall_verdict
        // 4. Ít nhất 1 example input/output
        """;

    // TODO: Viết test cases
    static List<String> TEST_CASES = List.of(
        // TODO: Case 1: Code tốt — expected: no CRITICAL/HIGH issues
        """
        public String getUserName(Long id) {
            User user = userRepository.findById(id).orElseThrow();
            return user.getName();
        }
        """,

        // TODO: Case 2: Null pointer bug
        // Hint: method không check null trước khi gọi method trên object

        // TODO: Case 3: SQL injection vulnerability
        // Hint: string concatenation trong SQL query
        ""
    );

    public static void main(String[] args) {
        // TODO: Run eval với CODE_REVIEW_SYSTEM_PROMPT và TEST_CASES
        // Expected pass rate: >= 80%
    }
}
```

### Exercise 6.2: SQL Generation Prompt

```java
public class SqlGenerationPromptExercise {

    static final String SCHEMA = """
        CREATE TABLE orders (
            id BIGSERIAL PRIMARY KEY,
            customer_id BIGINT NOT NULL,
            status VARCHAR(20), -- PENDING, PROCESSING, SHIPPED, DELIVERED, CANCELLED
            total_amount DECIMAL(10,2),
            created_at TIMESTAMP DEFAULT NOW()
        );
        CREATE TABLE order_items (
            id BIGSERIAL PRIMARY KEY,
            order_id BIGINT REFERENCES orders(id),
            product_id BIGINT,
            quantity INTEGER,
            unit_price DECIMAL(10,2)
        );
        """;

    // TODO: Viết SQL Generator prompt với schema trên
    static String SQL_GEN_PROMPT = ""; // Your prompt here

    // TODO: Test cases — input: natural language, expected: valid SQL
    static List<String[]> TEST_QUERIES = List.of(
        new String[]{"How many orders were placed today?",
                     "SELECT.*COUNT.*orders.*TODAY\\|CURRENT_DATE"},
        new String[]{"Top 5 most expensive orders",
                     "SELECT.*orders.*ORDER BY.*DESC.*LIMIT 5"},
        new String[]{"Orders that are still pending",
                     "SELECT.*orders.*WHERE.*PENDING"}
        // TODO: Add 2 more test cases
    );
}
```

### Exercise 6.3: Error Diagnosis Prompt

```java
public class ErrorDiagnosisPromptExercise {

    // TODO: Viết Error Diagnosis prompt
    // Input: exception class + stack trace + brief app context
    // Output: (1) root cause, (2) specific fix với code example, (3) prevention tip

    static final List<String> TEST_EXCEPTIONS = List.of(
        // NullPointerException
        """
        java.lang.NullPointerException: Cannot invoke "String.length()" because "str" is null
            at com.example.UserService.processName(UserService.java:42)
            at com.example.UserController.updateUser(UserController.java:87)
        """,

        // ConcurrentModificationException
        """
        java.util.ConcurrentModificationException
            at java.util.ArrayList$Itr.checkForComodification(ArrayList.java:911)
            at com.example.OrderService.removeExpiredOrders(OrderService.java:156)
        """,

        // LazyInitializationException (Hibernate)
        """
        org.hibernate.LazyInitializationException: failed to lazily initialize a collection
        of role: com.example.Order.items, could not initialize proxy - no Session
            at com.example.ReportService.generateReport(ReportService.java:78)
        """
    );
}
```

---

## Tóm tắt Lesson 04

| Kỹ thuật | Khi dùng | Key Point |
|---------|---------|----------|
| System prompt design | Mọi production feature | Role + Constraints + Format + Examples |
| Few-shot | Task có pattern rõ ràng | 3-5 examples > 500 words rules |
| Chain-of-Thought | Complex reasoning, debugging | "Think step by step before answering" |
| XML tags | Prompt phức tạp với nhiều sections | Ranh giới rõ ràng giữa instructions và data |
| Structured output | Backend cần parse response | JSON schema trong system prompt + validate |
| Prompt templates | Tái sử dụng prompts | Parameterized với `{placeholder}` |
| Eval framework | Production prompts | Test trước khi deploy, track regressions |

### Checklist trước khi deploy prompt vào production

- [ ] Có system prompt rõ ràng với role, constraints, output format
- [ ] Có ít nhất 3 few-shot examples cho tasks phức tạp
- [ ] Output format được specify và validate trong Java code
- [ ] Edge cases được handle (empty input, invalid input, no data found)
- [ ] Eval test suite có ít nhất 5 test cases
- [ ] Pass rate >= 80% trên eval suite
- [ ] maxTokens được set phù hợp với expected output size
- [ ] Model được chọn phù hợp (Haiku cho simple, Sonnet cho complex)

**Tiếp theo**: [Project — AI Database Assistant](../exercises/project-ai-database-assistant.md) — Tổng hợp tất cả kiến thức từ 4 lessons để xây dựng production-ready Spring Boot app.
