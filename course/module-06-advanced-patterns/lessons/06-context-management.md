# Lesson 06: Context Management — Working Within Token Limits

> **Module 06 — Advanced Patterns**
> Prerequisite: Lesson 01 (ReAct agent), Lesson 04 (Memory Systems), Lesson 05 (Ingestion Pipeline)
> Thời gian ước tính: 75 phút đọc + 60 phút lab

---

## Mở đầu: Tại sao Context Management là vấn đề thực tế

Claude 3.5 Sonnet có context window 200K tokens. Nghe có vẻ lớn — nhưng hãy làm phép tính:

```
System prompt chi tiết:           ~2,000 tokens
Tool definitions (10 tools):      ~3,000 tokens
Codebase context (5 files):      ~10,000 tokens
Conversation history (50 turns):  ~25,000 tokens
Tool results (logs, API responses): ~40,000 tokens
─────────────────────────────────────────────────
Total sau 2 giờ làm việc:         ~80,000 tokens

Mỗi API call = 80K input tokens
Cost: ~$2.40 per call (Claude 3.5 Sonnet)
Agent chạy 100 calls/ngày = $240/ngày = $87,600/năm
```

Và đó là **kịch bản vừa phải**. Agent có thể dễ dàng đạt 150K tokens trong một session debugging phức tạp.

**Context management không chỉ là vấn đề tiết kiệm tiền — nó là vấn đề correctness.** LLM performance giảm khi context window quá đầy (lost-in-the-middle problem): những thông tin ở giữa context bị "quên" một phần, trong khi thông tin ở đầu và cuối được chú ý nhiều hơn.

---

## 4 Chiến lược Context Management

### Chiến lược 1: Sliding Window

Giữ **N messages gần nhất**, bỏ đi messages cũ hơn.

```
Full history:  [M1][M2][M3][M4][M5][M6][M7][M8][M9][M10]
Window (N=6):              [M5][M6][M7][M8][M9][M10]
```

```java
@Component
public class SlidingWindowStrategy implements ContextStrategy {

    /**
     * Giữ N messages cuối cùng.
     * Luôn bảo toàn system message (index 0) nếu có.
     */
    public List<MessageParam> apply(List<MessageParam> history, int maxMessages) {
        if (history.size() <= maxMessages) return history;

        // Tìm system message nếu có
        var systemMessages = history.stream()
                .filter(m -> m.role() == Role.SYSTEM)
                .toList();
        var conversationMessages = history.stream()
                .filter(m -> m.role() != Role.SYSTEM)
                .toList();

        // Lấy N messages cuối trong conversation
        int start = Math.max(0, conversationMessages.size() - maxMessages);
        var recentMessages = new ArrayList<>(
            conversationMessages.subList(start, conversationMessages.size())
        );

        // Prepend system messages (luôn giữ)
        var result = new ArrayList<>(systemMessages);
        result.addAll(recentMessages);
        return result;
    }

    /**
     * Variant: giữ N tokens thay vì N messages.
     * Chính xác hơn nhưng cần token counting.
     */
    public List<MessageParam> applyByTokenBudget(List<MessageParam> history,
                                                   int maxTokens,
                                                   TokenCounter counter) {
        var result = new LinkedList<MessageParam>();
        int usedTokens = 0;

        // Đi từ cuối lên đầu — ưu tiên messages gần nhất
        var reversed = new ArrayList<>(history);
        Collections.reverse(reversed);

        for (var message : reversed) {
            int msgTokens = counter.count(message);
            if (usedTokens + msgTokens > maxTokens) break;
            result.addFirst(message);
            usedTokens += msgTokens;
        }

        return new ArrayList<>(result);
    }
}
```

**Khi nào dùng:** Conversational agents, chatbots, khi early context ít quan trọng hơn recent context.

**Hạn chế:** Mất early context — agent có thể "quên" instructions hoặc decisions đã đưa ra sớm trong session.

---

### Chiến lược 2: Hierarchical Summarization

Thay vì bỏ messages cũ, **tóm tắt chúng thành compact summary**, rồi thay thế messages gốc bằng summary.

```
Phase 1 (planning):    [M1][M2][M3][M4][M5]  →  [SUMMARY-1]
Phase 2 (execution):   [M6][M7][M8][M9][M10] →  [SUMMARY-2]
Current window:        [SUMMARY-1][SUMMARY-2][M11][M12][M13]
```

```java
@Component
@Slf4j
public class HierarchicalSummarizationStrategy implements ContextStrategy {

    private final ClaudeClient claude;
    private static final int MESSAGES_PER_SUMMARY = 10;
    private static final int KEEP_RECENT = 5;

    /**
     * Khi history vượt ngưỡng, tóm tắt batch messages cũ.
     * Giữ lại recent messages để maintain momentum.
     */
    public List<MessageParam> apply(List<MessageParam> history,
                                     int targetMessageCount) {
        if (history.size() <= targetMessageCount) return history;

        // Tách: messages cần summarize vs messages giữ nguyên
        int keepFrom = history.size() - KEEP_RECENT;
        var toSummarize = history.subList(0, keepFrom);
        var toKeep = history.subList(keepFrom, history.size());

        // Summarize in batches
        var summaries = new ArrayList<MessageParam>();
        var batches = partition(toSummarize, MESSAGES_PER_SUMMARY);

        for (var batch : batches) {
            var summary = summarizeBatch(batch);
            summaries.add(MessageParam.builder()
                    .role(Role.USER)
                    .content("[CONVERSATION SUMMARY]\n" + summary)
                    .build());
        }

        // Result: [summaries...] + [recent messages]
        var result = new ArrayList<>(summaries);
        result.addAll(toKeep);
        return result;
    }

    private String summarizeBatch(List<MessageParam> messages) {
        var formatted = messages.stream()
                .map(m -> m.role().name() + ": " + extractText(m))
                .collect(Collectors.joining("\n\n"));

        log.debug("Summarizing {} messages", messages.size());

        return claude.complete("""
            Summarize the following agent conversation concisely.
            Preserve ALL key information:
            - Decisions made and their rationale
            - Actions taken and their outcomes (success/failure)
            - Data discovered (errors found, metrics observed, etc.)
            - Current state of the task
            
            Be concise but COMPLETE — nothing important should be lost.
            Write in past tense, third person.
            
            Conversation:
            %s
            """.formatted(formatted));
    }

    private String extractText(MessageParam message) {
        // Extract plain text từ message content
        if (message.content() instanceof String s) return s;
        // Handle complex content types (tool results, etc.)
        return message.content().toString();
    }

    private <T> List<List<T>> partition(List<T> list, int size) {
        var result = new ArrayList<List<T>>();
        for (int i = 0; i < list.size(); i += size) {
            result.add(list.subList(i, Math.min(i + size, list.size())));
        }
        return result;
    }
}
```

**Khi nào dùng:** Multi-phase agent tasks (planning → execution → review), long debugging sessions với rõ ràng các phases.

**Hạn chế:** Summarization tốn thêm API calls. Có thể mất chi tiết quan trọng nếu summarization prompt không đủ tốt.

---

### Chiến lược 3: Selective Retention

Phân loại từng message theo **mức độ quan trọng**, giữ critical, nén useful, bỏ redundant.

```java
public enum MessageImportance {
    CRITICAL,    // Luôn giữ: initial task, final decisions, error states
    USEFUL,      // Nén khi cần: intermediate results, analysis
    REDUNDANT    // Bỏ: repetitive tool calls, verbose logs
}

@Component
public class SelectiveRetentionStrategy implements ContextStrategy {

    private final ClaudeClient claude;

    public List<MessageParam> apply(List<MessageParam> history, int tokenBudget) {
        // 1. Classify each message
        var classified = classify(history);

        // 2. Always keep CRITICAL
        var result = classified.entrySet().stream()
                .filter(e -> e.getValue() == MessageImportance.CRITICAL)
                .map(Map.Entry::getKey)
                .collect(Collectors.toCollection(ArrayList::new));

        // 3. Add USEFUL messages if budget allows
        classified.entrySet().stream()
                .filter(e -> e.getValue() == MessageImportance.USEFUL)
                .map(Map.Entry::getKey)
                .forEach(result::add);

        // 4. Sort back to original order
        var originalOrder = new HashMap<MessageParam, Integer>();
        for (int i = 0; i < history.size(); i++) originalOrder.put(history.get(i), i);
        result.sort(Comparator.comparingInt(originalOrder::get));

        return result;
    }

    private Map<MessageParam, MessageImportance> classify(
            List<MessageParam> messages) {
        // Rule-based classification (fast, no LLM call)
        return messages.stream().collect(Collectors.toMap(
            m -> m,
            this::classifyByRules
        ));
    }

    private MessageImportance classifyByRules(MessageParam message) {
        var content = extractText(message).toLowerCase();

        // CRITICAL patterns
        if (content.contains("task:") || content.contains("objective:")) {
            return MessageImportance.CRITICAL;   // Initial task definition
        }
        if (content.contains("error") && content.contains("critical")) {
            return MessageImportance.CRITICAL;   // Critical errors
        }
        if (content.contains("decision:") || content.contains("conclusion:")) {
            return MessageImportance.CRITICAL;   // Key decisions
        }

        // REDUNDANT patterns
        if (content.contains("tool_result") && content.length() > 5000) {
            return MessageImportance.REDUNDANT;  // Very long tool results (logs, etc.)
        }
        if (content.startsWith("searching") || content.startsWith("looking up")) {
            return MessageImportance.REDUNDANT;  // Routine lookups
        }

        return MessageImportance.USEFUL;  // Default
    }

    private String extractText(MessageParam m) {
        return m.content() instanceof String s ? s : m.content().toString();
    }
}
```

**Khi nào dùng:** Agents với nhiều tool calls có tính chất khác nhau (một số tool calls quan trọng, số khác là routine lookups).

---

### Chiến lược 4: External Memory (Pointer-based)

Thay vì đưa **nội dung** vào context, chỉ đưa **reference/summary** — và dùng tool để retrieve chi tiết khi cần.

```java
// ANTI-PATTERN: Inject toàn bộ file lớn vào context
// Tốn ~10,000 tokens mỗi call, ngay cả khi chỉ cần 1 method
String antiPattern = """
    Here is the full UserService.java (2,500 lines):
    """ + readEntireFile("UserService.java");   // BAD

// GOOD PATTERN: Chỉ inject summary + pointer
String goodPattern = """
    Available codebase files:
    - UserService.java: User CRUD, authentication logic
      Key methods: save(User), findById(Long), authenticate(String, String), delete(Long)
    - PaymentService.java: Payment processing
      Key methods: charge(PaymentRequest), refund(String), getStatus(String)
    
    Use the read_method tool to retrieve specific method implementations when needed.
    """;
// Khi agent cần chi tiết: agent gọi read_method("UserService", "authenticate")
// → retrieve chỉ method đó (~50 lines) thay vì toàn file
```

```java
@Component
public class ExternalMemoryStrategy {

    private final KnowledgeBaseRetriever retriever;

    /**
     * Build context với pointers thay vì full content.
     * Agent dùng tools để retrieve chi tiết khi cần.
     */
    public String buildPointerContext(List<String> relevantSources) {
        var sb = new StringBuilder();
        sb.append("=== Available Knowledge (use tools to retrieve details) ===\n\n");

        for (var source : relevantSources) {
            var summary = retriever.getSummary(source);
            sb.append("Source: ").append(source).append("\n");
            sb.append("Summary: ").append(summary).append("\n");
            sb.append("Retrieve with: retrieve_content(\"").append(source).append("\")\n\n");
        }

        return sb.toString();
    }

    /**
     * Tool definition: cho phép agent retrieve content on-demand.
     * Thay vì inject 10,000 tokens upfront, agent pull khi thực sự cần.
     */
    public Tool buildRetrieveContentTool() {
        return Tool.builder()
                .name("retrieve_content")
                .description("""
                    Retrieve the full content of a specific source.
                    Use this when you need to read the actual implementation,
                    not just the summary.
                    """)
                .inputSchema(Schema.builder()
                        .property("source", "string",
                                  "The source identifier to retrieve")
                        .required("source")
                        .build())
                .build();
    }
}
```

**Khi nào dùng:** Khi agent cần access tới large documents/files nhưng không phải lúc nào cũng cần full content. Đặc biệt hiệu quả cho codebase Q&A agents.

---

## Token Counting trong Java

Trước khi manage context, cần **đo** context size.

```java
@Component
public class TokenCounter {

    /**
     * Estimate token count từ Claude API response.
     * Đây là cách chính xác nhất — dùng actual usage từ response.
     */
    public int countFromResponse(Message response) {
        var usage = response.usage();
        return usage.inputTokens() + usage.outputTokens();
    }

    /**
     * Estimate tokens từ text (trước khi gọi API).
     * Rule of thumb: 1 token ≈ 4 characters (English), 2-3 characters (code)
     * Đây là approximation — actual tokenization phụ thuộc vào tokenizer.
     */
    public int estimateTokens(String text) {
        if (text == null || text.isBlank()) return 0;

        // Heuristic: code có token density cao hơn prose
        boolean looksLikeCode = text.contains("{") || text.contains("->") ||
                                text.contains("import ") || text.contains("def ");

        double charsPerToken = looksLikeCode ? 3.0 : 4.0;
        return (int) Math.ceil(text.length() / charsPerToken);
    }

    /**
     * Estimate tokens cho toàn bộ message list.
     */
    public int estimateTotal(List<MessageParam> messages) {
        return messages.stream()
                .mapToInt(m -> estimateTokens(extractText(m)) + 4)  // +4 cho message overhead
                .sum();
    }

    private String extractText(MessageParam m) {
        return m.content() instanceof String s ? s : m.content().toString();
    }
}
```

---

## Context Budget Manager

Quản lý token budget **proactively** — không chờ đến khi bị lỗi "context length exceeded".

```java
@Component
@Slf4j
public class ContextBudgetManager {

    private final int maxTokens;
    private final int warningThreshold;   // % của maxTokens để warn
    private final TokenCounter counter;

    private int spentTokens = 0;
    private int callCount = 0;

    public ContextBudgetManager(
            @Value("${agent.max-tokens:150000}") int maxTokens,
            @Value("${agent.warning-threshold:50}") int warningThresholdPct,
            TokenCounter counter) {
        this.maxTokens = maxTokens;
        this.warningThreshold = (int) (maxTokens * warningThresholdPct / 100.0);
        this.counter = counter;
    }

    /**
     * Kiểm tra có còn đủ budget không trước khi gọi API.
     */
    public BudgetStatus checkBudget(List<MessageParam> messages) {
        int estimatedTokens = counter.estimateTotal(messages);
        int projected = spentTokens + estimatedTokens;

        if (projected > maxTokens) {
            return BudgetStatus.EXCEEDED;
        }
        if (projected > warningThreshold) {
            log.warn("Context budget warning: {}% used ({}/{} tokens)",
                     projected * 100 / maxTokens, projected, maxTokens);
            return BudgetStatus.WARNING;
        }
        return BudgetStatus.OK;
    }

    /**
     * Record actual usage sau mỗi API call.
     */
    public void recordUsage(Usage usage) {
        spentTokens += usage.inputTokens() + usage.outputTokens();
        callCount++;
        log.debug("Token usage: +{} (total: {}/{}, calls: {})",
                  usage.inputTokens() + usage.outputTokens(),
                  spentTokens, maxTokens, callCount);
    }

    public int getRemainingBudget() {
        return Math.max(0, maxTokens - spentTokens);
    }

    public double getUsagePercent() {
        return (double) spentTokens / maxTokens * 100;
    }

    public boolean isApproachingLimit() {
        return getUsagePercent() > 70;
    }

    public AgentBudgetReport generateReport() {
        return new AgentBudgetReport(
            spentTokens, maxTokens, callCount,
            getRemainingBudget(), getUsagePercent()
        );
    }
}

public enum BudgetStatus { OK, WARNING, EXCEEDED }

public record AgentBudgetReport(
    int spentTokens, int maxTokens, int apiCalls,
    int remainingTokens, double usagePct
) {}
```

---

## Prompt Caching: Giảm 90% Chi phí System Prompt

Prompt caching là tính năng của Anthropic API cho phép **cache** phần context không thay đổi giữa các calls. Với agents có system prompt lớn (tools definitions, instructions, few-shot examples), đây là optimization quan trọng nhất.

### Chi phí so sánh

```
Không cache:
  Input tokens: $3.00 / 1M tokens
  System prompt 5,000 tokens × 100 calls = 500K tokens = $1.50

Với prompt cache:
  Cache write: $3.75 / 1M tokens (một lần)
  Cache read:  $0.30 / 1M tokens (các lần sau)
  
  Write (lần 1): 5,000 tokens = $0.019
  Read (99 lần sau): 5,000 × 99 = 495K tokens = $0.149
  Total: $0.168 thay vì $1.50 → tiết kiệm 89%
```

### Triển khai Prompt Caching trong Java SDK

```java
@Configuration
public class AgentConfiguration {

    /**
     * Build system prompt với cache_control annotation.
     * Anthropic SDK sẽ cache phần này nếu cache hits đủ nhiều.
     */
    public List<BetaTextBlockParam> buildCachedSystemPrompt() {
        // Phần 1: Static instructions (cache aggressively)
        var staticInstructions = BetaTextBlockParam.builder()
                .type("text")
                .text(LARGE_STATIC_SYSTEM_PROMPT)  // ~3,000 tokens
                .cacheControl(BetaCacheControlEphemeral.builder()
                        .type(BetaCacheControlEphemeralType.EPHEMERAL)
                        .build())
                .build();

        // Phần 2: Tool definitions (thay đổi ít — cũng cache)
        var toolDefinitions = BetaTextBlockParam.builder()
                .type("text")
                .text(serializeToolDefinitions())  // ~2,000 tokens
                .cacheControl(BetaCacheControlEphemeral.builder()
                        .type(BetaCacheControlEphemeralType.EPHEMERAL)
                        .build())
                .build();

        // Phần 3: Dynamic context (KHÔNG cache — thay đổi mỗi call)
        var dynamicContext = BetaTextBlockParam.builder()
                .type("text")
                .text(buildDynamicContext())  // current task, timestamp, etc.
                .build();

        return List.of(staticInstructions, toolDefinitions, dynamicContext);
    }
}
```

```java
@Service
public class CacheAwareAgentRunner {

    private final AnthropicBetaClient client;
    private final AgentConfiguration config;

    public Message runWithCaching(List<MessageParam> messages, String task) {
        // System prompt với cache hints
        var systemPrompt = config.buildCachedSystemPrompt();

        var response = client.beta().messages().create(
            BetaMessageCreateParams.builder()
                .model("claude-sonnet-4-5")
                .maxTokens(8096)
                .system(systemPrompt)
                .messages(messages)
                .betas(List.of("prompt-caching-2024-07-31"))
                .build()
        );

        // Log cache performance
        var usage = response.usage();
        log.info("Token usage - input: {}, output: {}, cache_read: {}, cache_write: {}",
                 usage.inputTokens(),
                 usage.outputTokens(),
                 usage.cacheReadInputTokens().orElse(0L),
                 usage.cacheCreationInputTokens().orElse(0L));

        return response;
    }
}
```

**Quy tắc cache hiệu quả:**
- Cache content **không thay đổi hoặc ít thay đổi**: system instructions, tool definitions, few-shot examples
- **Đặt cached content trước dynamic content** trong prompt — cache invalidates khi content thay đổi
- Cache chỉ tốt khi TTL đủ dài (5 phút minimum) và đủ nhiều calls share cùng cached content

---

## Adaptive Context Management: Kết hợp các Chiến lược

Agent production cần **kết hợp** các chiến lược theo ngữ cảnh, không dùng một chiến lược cố định.

```java
@Service
@Slf4j
public class AdaptiveContextManager {

    private final ContextBudgetManager budgetManager;
    private final SlidingWindowStrategy slidingWindow;
    private final HierarchicalSummarizationStrategy summarization;
    private final TokenCounter counter;

    /**
     * Áp dụng context strategy phù hợp dựa trên budget status.
     * Tự động escalate khi token usage tăng.
     */
    public List<MessageParam> optimize(List<MessageParam> history,
                                        String currentTask) {
        var status = budgetManager.checkBudget(history);

        return switch (status) {
            case OK -> {
                // Budget dư dả: giữ nguyên, không compress
                log.debug("Context OK ({:.1f}% used), no compression needed",
                          budgetManager.getUsagePercent());
                yield history;
            }
            case WARNING -> {
                // 50-70% budget: slide window, giữ recent messages
                log.info("Context WARNING ({:.1f}% used), applying sliding window",
                         budgetManager.getUsagePercent());
                yield slidingWindow.apply(history, 20);
            }
            case EXCEEDED -> {
                // > 70% budget: aggressive summarization
                log.warn("Context EXCEEDED ({:.1f}% used), applying summarization",
                         budgetManager.getUsagePercent());
                var summarized = summarization.apply(history, 10);
                // Double-check sau summarization
                if (budgetManager.checkBudget(summarized) == BudgetStatus.EXCEEDED) {
                    // Emergency: chỉ giữ tóm tắt task + 5 messages gần nhất
                    return emergencyTrim(summarized, currentTask);
                }
                yield summarized;
            }
        };
    }

    /**
     * Emergency trim: chỉ giữ task description + N messages gần nhất.
     * Dùng khi các strategies khác không đủ.
     */
    private List<MessageParam> emergencyTrim(List<MessageParam> history,
                                              String currentTask) {
        log.error("Emergency context trim activated! Task: {}", currentTask);

        var taskMessage = MessageParam.builder()
                .role(Role.USER)
                .content("[CONTEXT TRIMMED DUE TO TOKEN LIMIT]\n"
                       + "Original task: " + currentTask + "\n"
                       + "Continuing from most recent state...")
                .build();

        var recent = slidingWindow.apply(history, 5);
        var result = new ArrayList<MessageParam>();
        result.add(taskMessage);
        result.addAll(recent);
        return result;
    }
}
```

---

## ReAct Agent với Context Management

Kết hợp với Lesson 01 — thêm context management vào ReAct loop.

```java
@Service
@Slf4j
public class ContextAwareReActAgent {

    private final AnthropicClient claude;
    private final ToolExecutor toolExecutor;
    private final AdaptiveContextManager contextManager;
    private final ContextBudgetManager budgetManager;

    private static final int MAX_ITERATIONS = 20;

    public AgentResult run(String task) {
        var history = new ArrayList<MessageParam>();
        history.add(MessageParam.builder()
                .role(Role.USER)
                .content(task)
                .build());

        for (int iteration = 0; iteration < MAX_ITERATIONS; iteration++) {

            // === CONTEXT MANAGEMENT STEP (mới so với Lesson 01) ===
            var optimizedHistory = contextManager.optimize(history, task);
            if (optimizedHistory.size() < history.size()) {
                log.info("Context compressed: {} → {} messages",
                         history.size(), optimizedHistory.size());
            }

            // === LLM CALL ===
            var response = claude.messages().create(
                MessageCreateParams.builder()
                    .model("claude-sonnet-4-5")
                    .maxTokens(4096)
                    .system(SYSTEM_PROMPT)
                    .messages(optimizedHistory)
                    .tools(TOOLS)
                    .build()
            );

            // === RECORD USAGE ===
            budgetManager.recordUsage(response.usage());

            // === HANDLE RESPONSE ===
            if (response.stopReason() == StopReason.END_TURN) {
                return AgentResult.success(extractText(response), iteration + 1,
                                           budgetManager.generateReport());
            }

            if (response.stopReason() == StopReason.TOOL_USE) {
                // Execute tools và add results to history
                var toolResults = executeTools(response);

                history.add(MessageParam.builder()
                        .role(Role.ASSISTANT)
                        .content(response.content())
                        .build());
                history.add(MessageParam.builder()
                        .role(Role.USER)
                        .content(toolResults)
                        .build());
            }

            // === BUDGET GUARD ===
            if (budgetManager.getRemainingBudget() < 5000) {
                log.warn("Token budget nearly exhausted. Forcing conclusion.");
                return forcedConclusion(history, task, budgetManager.generateReport());
            }
        }

        return AgentResult.maxIterationsReached(MAX_ITERATIONS,
                                                budgetManager.generateReport());
    }

    /**
     * Khi budget gần cạn, yêu cầu agent kết luận ngay.
     */
    private AgentResult forcedConclusion(List<MessageParam> history,
                                          String originalTask,
                                          AgentBudgetReport budget) {
        history.add(MessageParam.builder()
                .role(Role.USER)
                .content("""
                    Token budget is nearly exhausted. Please provide your FINAL answer now.
                    Summarize what you found and any conclusions, even if incomplete.
                    Original task: """ + originalTask)
                .build());

        var finalResponse = claude.messages().create(
            MessageCreateParams.builder()
                .model("claude-sonnet-4-5")
                .maxTokens(1024)  // Giới hạn output tokens cho forced conclusion
                .messages(history)
                .build()
        );

        return AgentResult.budgetForced(extractText(finalResponse), budget);
    }
}
```

---

## Monitoring Context Usage

Để optimize hiệu quả, cần **observe** context usage trong production.

```java
@Component
public class ContextUsageMonitor {

    private final MeterRegistry meterRegistry;

    public void recordAgentRun(AgentBudgetReport report) {
        // Metrics for Prometheus/Grafana
        meterRegistry.counter("agent.token.input",
            "model", "claude-sonnet")
            .increment(report.spentTokens());

        meterRegistry.gauge("agent.context.usage_pct",
            report.usagePct());

        meterRegistry.counter("agent.api.calls")
            .increment(report.apiCalls());

        // Alert nếu agent thường xuyên hit budget limit
        if (report.usagePct() > 80) {
            log.warn("Agent ran at {}% context capacity — consider optimization",
                     report.usagePct());
        }
    }

    /**
     * Log context breakdown — giúp identify what's consuming tokens.
     */
    public void logContextBreakdown(List<MessageParam> messages,
                                     TokenCounter counter) {
        log.info("=== Context Breakdown ===");
        messages.forEach(m -> {
            int tokens = counter.estimateTokens(extractText(m));
            log.info("  [{}] ~{} tokens: {}",
                     m.role(),
                     tokens,
                     extractText(m).substring(0, Math.min(60, extractText(m).length())));
        });
        log.info("  Total: ~{} tokens", counter.estimateTotal(messages));
    }
}
```

---

## Bảng Quyết định: Chọn Strategy nào?

```
┌────────────────────────────────────────────────────────────────┐
│              CONTEXT STRATEGY DECISION GUIDE                    │
├─────────────────────────┬──────────────────────────────────────┤
│ Scenario                │ Strategy                              │
├─────────────────────────┼──────────────────────────────────────┤
│ Short task (< 30 min)   │ No management needed                  │
│ Simple chatbot          │ Sliding window (N=20)                 │
├─────────────────────────┼──────────────────────────────────────┤
│ Multi-phase task        │ Hierarchical summarization            │
│ (plan → execute → review│ Summarize each phase                  │
├─────────────────────────┼──────────────────────────────────────┤
│ Tool-heavy agent        │ Selective retention                   │
│ (many tool calls)       │ Keep tool results classified CRITICAL │
├─────────────────────────┼──────────────────────────────────────┤
│ Codebase Q&A            │ External memory (pointer-based)       │
│                         │ + on-demand retrieval tools           │
├─────────────────────────┼──────────────────────────────────────┤
│ Production agent        │ Adaptive (combine all strategies)     │
│ (long-running)          │ + prompt caching + monitoring         │
├─────────────────────────┼──────────────────────────────────────┤
│ Repeated similar tasks  │ Prompt caching (system prompt)        │
│ (100+ calls/day)        │ → 89% cost reduction                  │
└─────────────────────────┴──────────────────────────────────────┘
```

---

## Lab Exercise: Context Management cho ReAct Agent

Nâng cấp ReAct agent từ Lesson 01 với đầy đủ context management.

### Yêu cầu

1. **Tích hợp ContextBudgetManager:**
   - Inject vào ReAct loop
   - Log token usage sau mỗi call
   - Alert khi vượt 50% budget

2. **Implement Auto-summarization:**
   - Khi history > 15 messages, trigger summarization
   - Summarize batch messages cũ (5 messages/lần)
   - Verify: agent behavior không đổi sau khi summarize

3. **Prompt Caching:**
   - Add `cache_control` vào system prompt (static part)
   - Log cache hits vs cache misses
   - Measure: sau 10 calls cùng system prompt, bao nhiêu % được cache?

4. **Forced Conclusion:**
   - Khi remaining budget < 5,000 tokens, inject forced conclusion message
   - Agent phải kết luận gracefully thay vì bị cắt giữa chừng

### Test Scenario

```
Task: "Analyze the payments microservice for performance bottlenecks.
       Check logs, metrics, and code. Provide recommendations."

Expected behavior:
- Iteration 1-5: Full context, no compression
- Iteration 10+: Sliding window activated (50% warning)
- Iteration 15+: Summarization activated
- At any point if budget < 5K: forced conclusion
```

### Checklist

- [ ] ContextBudgetManager tracks tokens across all calls
- [ ] Warning logged tại 50% budget
- [ ] Auto-summarization triggers tại message threshold
- [ ] Summarization preserves key decisions (verify manually)
- [ ] Prompt caching: cache_control added to static system prompt
- [ ] Cache hit rate > 80% sau 5+ calls với same system prompt
- [ ] Forced conclusion không crash — trả về partial result gracefully
- [ ] Total cost of 20-call session < half of uncached/unmanaged baseline

---

## Tóm tắt Module 06

Sau ba lessons này, bạn có đủ công cụ để xây dựng **production-grade agent** với:

| Capability | Lesson | Công cụ |
|---|---|---|
| Agent nhớ past interactions | 04 - Memory | pgvector + Spring AI |
| Agent hiểu codebase | 05 - Ingestion | JavaParser + embeddings |
| Agent chạy dài không hết token | 06 - Context | Budget manager + caching |

**Kết hợp cả ba:** Agent IntelliOps đầy đủ = Memory system (nhớ past incidents) + Ingestion pipeline (hiểu codebase hiện tại) + Context management (chạy sessions dài không cạn token).

---

> **Next Steps:** Module 07 — Production Deployment: monitoring, cost tracking, circuit breakers, và graceful degradation cho agents chạy 24/7 trong production environment.
