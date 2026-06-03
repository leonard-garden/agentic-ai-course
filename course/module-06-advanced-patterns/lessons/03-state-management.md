# Lesson 03: Agent State Management — Checkpointing & Resumability

> **Module 06 — Advanced Agentic Patterns**
> **Thời gian**: ~ 4 giờ | **Difficulty**: Intermediate-Advanced

---

## Mục tiêu bài học

Sau bài này bạn sẽ:
- Hiểu tại sao state management là yếu tố sống còn của production agents
- Phân biệt stateless vs stateful agents và biết khi nào dùng cái nào
- Implement checkpointing pattern hoàn chỉnh với PostgreSQL và Redis
- Thiết kế AgentState record chuẩn để serialize/deserialize
- Hiểu idempotency và tại sao nó quan trọng khi retry
- Áp dụng checkpointing vào Java Code Review Pipeline từ Module 04

---

## 1. Tại sao State Management matters?

Hãy tưởng tượng bạn đang chạy một ReAct agent để review toàn bộ codebase của một project Java lớn — 200 files, 50,000 dòng code. Agent cần:

1. List tất cả files (bước 1)
2. Đọc và analyze từng file (bước 2-201)
3. Tổng hợp findings (bước 202)
4. Generate report (bước 203)

Bước 2-201 mất khoảng 45 phút. Khi agent đang ở bước 150, server bị timeout vì giới hạn 30 phút của HTTP request. **Toàn bộ 150 bước phải làm lại từ đầu.**

Với checkpointing:
- Sau mỗi bước, agent lưu state vào database
- Khi restart, agent đọc checkpoint và tiếp tục từ bước 151
- Tổng thời gian: 45 phút thay vì 90+ phút

**Đây không phải edge case — đây là reality của production agents.**

### Các lý do agent bị interrupt

```
Timeouts:
  - HTTP request timeout (30s, 60s, 5 phút — tùy config)
  - Lambda/Cloud Function execution limit (15 phút)
  - Database connection timeout

Failures:
  - Tool call throws uncaught exception
  - External API (GitHub, Jira) temporarily down
  - Out of memory
  - Claude API rate limit (429 error)

Infrastructure:
  - Server restart/redeploy
  - Container killed bởi Kubernetes
  - Network partition
```

Nếu không có state management, **tất cả những trường hợp trên đều mất toàn bộ work**.

---

## 2. Types of Agent State

Một agent đang chạy có nhiều loại state khác nhau, mỗi loại có đặc điểm và cách lưu trữ khác nhau:

```
┌─────────────────────────────────────────────────────────────┐
│                    Agent State Layers                         │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 1. Conversation History                               │   │
│  │    List<MessageParam> — mọi Thought/Action/Observation│   │
│  │    Size: có thể MB với long-running agents            │   │
│  │    Storage: PostgreSQL (JSON column) hoặc file        │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 2. Task Plan & Progress                               │   │
│  │    Danh sách steps, bước nào đã done, bước nào pending│   │
│  │    Size: nhỏ (KB)                                     │   │
│  │    Storage: PostgreSQL hoặc Redis                     │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 3. Tool Call Results (Cache)                          │   │
│  │    Kết quả của expensive tool calls để tránh re-run   │   │
│  │    Size: có thể lớn (file contents, API responses)    │   │
│  │    Storage: Redis (với TTL) hoặc PostgreSQL           │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 4. Working Memory                                     │   │
│  │    Intermediate reasoning, extracted entities         │   │
│  │    Size: nhỏ (KB)                                     │   │
│  │    Storage: trong AgentState object                   │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 5. External Artifacts                                 │   │
│  │    Files đã write, APIs đã call, emails đã send       │   │
│  │    Size: varies                                       │   │
│  │    Storage: separate tracking (idempotency keys)      │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Stateless vs Stateful Agents

### Stateless Agent

```
Request 1: run(task) → result
Request 2: run(task) → result (starts fresh, no memory of Request 1)
```

**Ưu điểm:**
- Đơn giản để implement và test
- Idempotent — safe to retry from scratch
- Dễ scale horizontal (bất kỳ instance nào cũng handle được request)
- Không cần cleanup

**Nhược điểm:**
- Tốn kém nếu task bị interrupt — phải làm lại từ đầu
- Không thể pause và resume
- Context bị mất giữa sessions

**Dùng khi:**
- Tasks ngắn (< 1 phút)
- Tasks idempotent (chạy lại không có side effects)
- High-throughput, independent requests

### Stateful Agent

```
Request 1: run(taskId, task) → partial result, checkpoint saved
[Server restart]
Request 2: run(taskId, task) → resume from checkpoint, continue
```

**Ưu điểm:**
- Resume sau interrupt
- Tiết kiệm tokens (không replay history)
- Human-in-the-loop: có thể pause, inspect, approve, continue

**Nhược điểm:**
- Phức tạp hơn đáng kể
- Cần storage layer (DB, Redis)
- State migration khi schema thay đổi
- Cleanup orphaned checkpoints

**Dùng khi:**
- Long-running tasks (> 5 phút)
- Expensive tool calls (cần cache results)
- Human-in-the-loop workflows
- Tasks với side effects (không muốn re-execute)

---

## 4. Designing AgentState

Trước khi implement storage, phải thiết kế state schema cẩn thận. Một `AgentState` tốt phải:
1. Capture đủ information để resume từ bất kỳ điểm nào
2. Serialize được sang JSON
3. Immutable (dùng Java records)
4. Versioned (để handle schema migrations)

```java
package com.course.module06.lesson03;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import java.time.Instant;
import java.util.*;

/**
 * Complete state snapshot của một agent run.
 * Đủ để resume từ bất kỳ điểm nào sau restart.
 */
@JsonIgnoreProperties(ignoreUnknown = true) // Forward compatibility
public record AgentState(

    // ── Identity ────────────────────────────────────────────────────────────
    String taskId,
    String agentType,       // "react", "plan-execute", v.v.
    int schemaVersion,      // Để handle migrations (current: 1)

    // ── Task ────────────────────────────────────────────────────────────────
    String originalTask,
    Map<String, Object> taskMetadata,  // PR number, repo URL, etc.

    // ── Progress ────────────────────────────────────────────────────────────
    List<CompletedStep> completedSteps,
    List<PendingStep> pendingSteps,
    int currentStepIndex,

    // ── Conversation ────────────────────────────────────────────────────────
    List<SerializedMessage> conversationHistory,

    // ── Working Memory ──────────────────────────────────────────────────────
    Map<String, Object> workingMemory, // Arbitrary KV store cho intermediate data

    // ── Completion ──────────────────────────────────────────────────────────
    boolean isComplete,
    String result,                     // null nếu chưa complete
    String failureReason,              // null nếu không fail

    // ── Timestamps ──────────────────────────────────────────────────────────
    Instant createdAt,
    Instant lastUpdatedAt,
    Instant completedAt               // null nếu chưa complete

) {

    // Factory method cho initial state
    public static AgentState initial(String taskId, String agentType, String task,
                                      Map<String, Object> metadata) {
        return new AgentState(
            taskId, agentType, 1,
            task, metadata,
            new ArrayList<>(), new ArrayList<>(), 0,
            new ArrayList<>(),
            new HashMap<>(),
            false, null, null,
            Instant.now(), Instant.now(), null
        );
    }

    // Immutable update: tạo state mới với một step completed
    public AgentState withCompletedStep(CompletedStep step) {
        List<CompletedStep> newCompleted = new ArrayList<>(this.completedSteps);
        newCompleted.add(step);

        List<PendingStep> newPending = new ArrayList<>(this.pendingSteps);
        if (!newPending.isEmpty()) {
            newPending.remove(0); // Remove first pending step
        }

        return new AgentState(
            taskId, agentType, schemaVersion,
            originalTask, taskMetadata,
            newCompleted, newPending, currentStepIndex + 1,
            conversationHistory,
            workingMemory,
            isComplete, result, failureReason,
            createdAt, Instant.now(), completedAt
        );
    }

    // Immutable update: thêm message vào conversation history
    public AgentState withMessage(SerializedMessage message) {
        List<SerializedMessage> newHistory = new ArrayList<>(this.conversationHistory);
        newHistory.add(message);
        return new AgentState(
            taskId, agentType, schemaVersion,
            originalTask, taskMetadata,
            completedSteps, pendingSteps, currentStepIndex,
            newHistory,
            workingMemory,
            isComplete, result, failureReason,
            createdAt, Instant.now(), completedAt
        );
    }

    // Immutable update: update working memory
    public AgentState withMemory(String key, Object value) {
        Map<String, Object> newMemory = new HashMap<>(this.workingMemory);
        newMemory.put(key, value);
        return new AgentState(
            taskId, agentType, schemaVersion,
            originalTask, taskMetadata,
            completedSteps, pendingSteps, currentStepIndex,
            conversationHistory,
            newMemory,
            isComplete, result, failureReason,
            createdAt, Instant.now(), completedAt
        );
    }

    // Immutable update: mark as complete
    public AgentState completed(String finalResult) {
        return new AgentState(
            taskId, agentType, schemaVersion,
            originalTask, taskMetadata,
            completedSteps, pendingSteps, currentStepIndex,
            conversationHistory,
            workingMemory,
            true, finalResult, null,
            createdAt, Instant.now(), Instant.now()
        );
    }

    // Immutable update: mark as failed
    public AgentState failed(String reason) {
        return new AgentState(
            taskId, agentType, schemaVersion,
            originalTask, taskMetadata,
            completedSteps, pendingSteps, currentStepIndex,
            conversationHistory,
            workingMemory,
            true, null, reason,
            createdAt, Instant.now(), Instant.now()
        );
    }
}
```

### Supporting Records

```java
package com.course.module06.lesson03;

import java.time.Instant;
import java.util.Map;

public record CompletedStep(
    int stepNumber,
    String description,
    String toolName,
    Map<String, Object> toolArgs,
    String toolResult,
    boolean success,
    long durationMs,
    Instant completedAt
) {}

public record PendingStep(
    int stepNumber,
    String description,
    String toolName,
    Map<String, Object> toolArgs
) {}

// Serializable version của Anthropic MessageParam
public record SerializedMessage(
    String role,           // "user" hoặc "assistant"
    String contentType,    // "text", "tool_use", "tool_result"
    String content,        // JSON string của content
    String toolUseId,      // Chỉ có với tool_use và tool_result
    String toolName        // Chỉ có với tool_use
) {}
```

---

## 5. CheckpointStore — Interface & Implementations

### Interface

```java
package com.course.module06.lesson03;

import java.util.List;
import java.util.Optional;

public interface CheckpointStore {

    /**
     * Lưu state (upsert: create hoặc update).
     */
    void save(AgentState state);

    /**
     * Load state theo taskId. Empty nếu chưa có checkpoint.
     */
    Optional<AgentState> load(String taskId);

    /**
     * Xóa checkpoint khi task hoàn thành hoặc hết hạn.
     */
    void delete(String taskId);

    /**
     * List tất cả incomplete tasks (để resume hoặc cleanup).
     */
    List<String> listIncomplete();

    /**
     * Xóa checkpoints cũ hơn maxAgeHours (cleanup job).
     */
    int deleteExpired(int maxAgeHours);
}
```

### Implementation 1: PostgreSQL (Recommended cho Production)

**Database schema:**

```sql
CREATE TABLE agent_checkpoints (
    task_id         VARCHAR(255) PRIMARY KEY,
    agent_type      VARCHAR(100) NOT NULL,
    schema_version  INT NOT NULL DEFAULT 1,
    state_json      JSONB NOT NULL,
    is_complete     BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ  -- NULL = never expires
);

-- Index để cleanup job
CREATE INDEX idx_checkpoints_incomplete ON agent_checkpoints (is_complete, updated_at)
    WHERE is_complete = FALSE;

-- Index để TTL queries
CREATE INDEX idx_checkpoints_expires ON agent_checkpoints (expires_at)
    WHERE expires_at IS NOT NULL;
```

**Java Implementation:**

```java
package com.course.module06.lesson03;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.jdbc.core.JdbcTemplate;

import java.sql.ResultSet;
import java.sql.SQLException;
import java.time.Instant;
import java.util.List;
import java.util.Optional;

public class PostgresCheckpointStore implements CheckpointStore {

    private final JdbcTemplate jdbc;
    private final ObjectMapper mapper;

    public PostgresCheckpointStore(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
        this.mapper = new ObjectMapper()
            .registerModule(new JavaTimeModule());
    }

    @Override
    public void save(AgentState state) {
        try {
            String stateJson = mapper.writeValueAsString(state);

            // UPSERT: create hoặc update
            jdbc.update("""
                INSERT INTO agent_checkpoints
                    (task_id, agent_type, schema_version, state_json, is_complete, updated_at)
                VALUES (?, ?, ?, ?::jsonb, ?, NOW())
                ON CONFLICT (task_id) DO UPDATE SET
                    state_json     = EXCLUDED.state_json,
                    is_complete    = EXCLUDED.is_complete,
                    updated_at     = NOW(),
                    completed_at   = CASE WHEN EXCLUDED.is_complete THEN NOW() ELSE NULL END
                """,
                state.taskId(),
                state.agentType(),
                state.schemaVersion(),
                stateJson,
                state.isComplete()
            );

        } catch (Exception e) {
            throw new CheckpointException("Failed to save checkpoint for task: "
                + state.taskId(), e);
        }
    }

    @Override
    public Optional<AgentState> load(String taskId) {
        List<AgentState> results = jdbc.query(
            "SELECT state_json FROM agent_checkpoints WHERE task_id = ?",
            (rs, rowNum) -> deserializeState(rs),
            taskId
        );
        return results.isEmpty() ? Optional.empty() : Optional.of(results.get(0));
    }

    @Override
    public void delete(String taskId) {
        jdbc.update("DELETE FROM agent_checkpoints WHERE task_id = ?", taskId);
    }

    @Override
    public List<String> listIncomplete() {
        return jdbc.queryForList(
            """
            SELECT task_id FROM agent_checkpoints
            WHERE is_complete = FALSE
            ORDER BY updated_at DESC
            """,
            String.class
        );
    }

    @Override
    public int deleteExpired(int maxAgeHours) {
        return jdbc.update(
            """
            DELETE FROM agent_checkpoints
            WHERE is_complete = TRUE
               AND completed_at < NOW() - INTERVAL '? hours'
            """,
            maxAgeHours
        );
    }

    private AgentState deserializeState(ResultSet rs) throws SQLException {
        try {
            return mapper.readValue(rs.getString("state_json"), AgentState.class);
        } catch (Exception e) {
            throw new SQLException("Failed to deserialize agent state", e);
        }
    }

    public static class CheckpointException extends RuntimeException {
        public CheckpointException(String message, Throwable cause) {
            super(message, cause);
        }
    }
}
```

### Implementation 2: Redis (Cho Speed — Cache Layer)

```java
package com.course.module06.lesson03;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.data.redis.core.RedisTemplate;

import java.time.Duration;
import java.util.List;
import java.util.Optional;
import java.util.Set;

public class RedisCheckpointStore implements CheckpointStore {

    private static final String KEY_PREFIX = "agent:checkpoint:";
    private static final String INCOMPLETE_SET = "agent:incomplete_tasks";
    private static final Duration DEFAULT_TTL = Duration.ofDays(7);

    private final RedisTemplate<String, String> redis;
    private final ObjectMapper mapper;

    public RedisCheckpointStore(RedisTemplate<String, String> redis) {
        this.redis = redis;
        this.mapper = new ObjectMapper()
            .registerModule(new JavaTimeModule());
    }

    @Override
    public void save(AgentState state) {
        try {
            String key = KEY_PREFIX + state.taskId();
            String json = mapper.writeValueAsString(state);

            redis.opsForValue().set(key, json, DEFAULT_TTL);

            // Track incomplete tasks trong a Set
            if (!state.isComplete()) {
                redis.opsForSet().add(INCOMPLETE_SET, state.taskId());
            } else {
                redis.opsForSet().remove(INCOMPLETE_SET, state.taskId());
            }

        } catch (Exception e) {
            throw new RuntimeException("Failed to save Redis checkpoint: " + state.taskId(), e);
        }
    }

    @Override
    public Optional<AgentState> load(String taskId) {
        String json = redis.opsForValue().get(KEY_PREFIX + taskId);
        if (json == null) return Optional.empty();

        try {
            return Optional.of(mapper.readValue(json, AgentState.class));
        } catch (Exception e) {
            throw new RuntimeException("Failed to deserialize checkpoint: " + taskId, e);
        }
    }

    @Override
    public void delete(String taskId) {
        redis.delete(KEY_PREFIX + taskId);
        redis.opsForSet().remove(INCOMPLETE_SET, taskId);
    }

    @Override
    public List<String> listIncomplete() {
        Set<String> members = redis.opsForSet().members(INCOMPLETE_SET);
        return members == null ? List.of() : List.copyOf(members);
    }

    @Override
    public int deleteExpired(int maxAgeHours) {
        // Redis tự handle TTL, không cần manual cleanup
        // Nhưng có thể remove từ INCOMPLETE_SET những keys đã expire
        Set<String> incomplete = redis.opsForSet().members(INCOMPLETE_SET);
        if (incomplete == null) return 0;

        int cleaned = 0;
        for (String taskId : incomplete) {
            if (!Boolean.TRUE.equals(redis.hasKey(KEY_PREFIX + taskId))) {
                redis.opsForSet().remove(INCOMPLETE_SET, taskId);
                cleaned++;
            }
        }
        return cleaned;
    }
}
```

### Implementation 3: File-Based (Cho Development/Testing)

```java
package com.course.module06.lesson03;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;

import java.io.IOException;
import java.nio.file.*;
import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.stream.Collectors;

public class FileCheckpointStore implements CheckpointStore {

    private final Path checkpointDir;
    private final ObjectMapper mapper;

    public FileCheckpointStore(Path checkpointDir) {
        this.checkpointDir = checkpointDir;
        this.mapper = new ObjectMapper()
            .registerModule(new JavaTimeModule());
        try {
            Files.createDirectories(checkpointDir);
        } catch (IOException e) {
            throw new RuntimeException("Cannot create checkpoint directory", e);
        }
    }

    @Override
    public void save(AgentState state) {
        Path file = checkpointDir.resolve(state.taskId() + ".json");
        try {
            mapper.writerWithDefaultPrettyPrinter()
                .writeValue(file.toFile(), state);
        } catch (IOException e) {
            throw new RuntimeException("Cannot save checkpoint: " + state.taskId(), e);
        }
    }

    @Override
    public Optional<AgentState> load(String taskId) {
        Path file = checkpointDir.resolve(taskId + ".json");
        if (!Files.exists(file)) return Optional.empty();

        try {
            return Optional.of(mapper.readValue(file.toFile(), AgentState.class));
        } catch (IOException e) {
            throw new RuntimeException("Cannot load checkpoint: " + taskId, e);
        }
    }

    @Override
    public void delete(String taskId) {
        try {
            Files.deleteIfExists(checkpointDir.resolve(taskId + ".json"));
        } catch (IOException e) {
            throw new RuntimeException("Cannot delete checkpoint: " + taskId, e);
        }
    }

    @Override
    public List<String> listIncomplete() {
        try {
            return Files.list(checkpointDir)
                .filter(p -> p.toString().endsWith(".json"))
                .map(p -> {
                    try {
                        AgentState state = mapper.readValue(p.toFile(), AgentState.class);
                        return state.isComplete() ? null : state.taskId();
                    } catch (IOException e) {
                        return null;
                    }
                })
                .filter(id -> id != null)
                .collect(Collectors.toList());
        } catch (IOException e) {
            throw new RuntimeException("Cannot list checkpoints", e);
        }
    }

    @Override
    public int deleteExpired(int maxAgeHours) {
        Instant cutoff = Instant.now().minusSeconds(maxAgeHours * 3600L);
        int count = 0;
        try {
            for (Path p : Files.list(checkpointDir).collect(Collectors.toList())) {
                AgentState state = mapper.readValue(p.toFile(), AgentState.class);
                if (state.isComplete()
                        && state.completedAt() != null
                        && state.completedAt().isBefore(cutoff)) {
                    Files.delete(p);
                    count++;
                }
            }
        } catch (IOException e) {
            throw new RuntimeException("Error during cleanup", e);
        }
        return count;
    }
}
```

---

## 6. CheckpointedAgent — Putting It All Together

Đây là agent template tích hợp checkpointing với ReAct loop:

```java
package com.course.module06.lesson03;

import com.anthropic.client.AnthropicClient;
import com.anthropic.models.messages.*;

import java.util.*;

public class CheckpointedReActAgent {

    private final AnthropicClient client;
    private final CheckpointStore checkpointStore;
    private final Map<String, ToolExecutor> tools;
    private static final int MAX_STEPS = 20;

    public CheckpointedReActAgent(
            AnthropicClient client,
            CheckpointStore checkpointStore,
            Map<String, ToolExecutor> tools) {
        this.client = client;
        this.checkpointStore = checkpointStore;
        this.tools = tools;
    }

    /**
     * Run hoặc resume agent cho taskId.
     * Nếu có checkpoint, resume từ đó. Nếu không, bắt đầu fresh.
     *
     * @param taskId  Unique ID cho task (dùng để lookup/save checkpoint)
     * @param task    Task description (chỉ dùng nếu không có checkpoint)
     * @return        Final result
     */
    public String run(String taskId, String task) {
        // 1. Try to resume từ checkpoint
        AgentState state = checkpointStore.load(taskId)
            .orElseGet(() -> {
                System.out.println("No checkpoint found, starting fresh for: " + taskId);
                return AgentState.initial(taskId, "react", task, Map.of());
            });

        if (state.isComplete()) {
            System.out.println("Task already complete, returning cached result");
            return state.result();
        }

        System.out.printf("Resuming task %s from step %d/%d%n",
            taskId, state.currentStepIndex(), MAX_STEPS);

        // 2. Reconstruct conversation history từ state
        List<MessageParam> history = deserializeHistory(state.conversationHistory());

        // 3. Nếu history empty (fresh start), add initial user message
        if (history.isEmpty()) {
            history.add(buildUserMessage(state.originalTask()));
            // Save initial state
            state = state.withMessage(new SerializedMessage(
                "user", "text", state.originalTask(), null, null));
            checkpointStore.save(state);
        }

        // 4. Resume ReAct loop
        for (int step = state.currentStepIndex(); step < MAX_STEPS; step++) {
            System.out.printf("--- Step %d/%d ---%n", step + 1, MAX_STEPS);

            // Call Claude
            Message response = callClaude(history);

            // Serialize và save response to state
            state = addResponseToState(state, response);
            history.add(buildAssistantMessage(response));

            // Check stopping conditions
            if (response.stopReason() == StopReason.END_TURN) {
                String text = extractText(response);

                if (text.contains("Final Answer:")) {
                    String finalAnswer = extractFinalAnswer(text);

                    // Mark complete và save
                    state = state.completed(finalAnswer);
                    checkpointStore.save(state);

                    System.out.println("Task completed: " + taskId);
                    return finalAnswer;
                }
            }

            // Handle tool calls
            if (response.stopReason() == StopReason.TOOL_USE) {
                List<ToolResultBlockParam> toolResults = executeTools(response, state);

                // Save state AFTER each tool execution (checkpoint!)
                state = state.withCompletedStep(buildCompletedStep(step, response, toolResults));
                checkpointStore.save(state); // <- Critical: checkpoint sau mỗi step

                // Add tool results vào history
                history.add(buildToolResultMessage(toolResults));
            }
        }

        // Max steps exceeded
        state = state.failed("Exceeded maximum steps: " + MAX_STEPS);
        checkpointStore.save(state);
        throw new RuntimeException("Max steps exceeded for task: " + taskId);
    }

    private List<ToolResultBlockParam> executeTools(Message response, AgentState state) {
        List<ToolResultBlockParam> results = new ArrayList<>();

        for (ContentBlock block : response.content()) {
            if (!(block instanceof ToolUseBlock toolUse)) continue;

            System.out.printf("Tool: %s(%s)%n", toolUse.name(), toolUse.input());

            // Check working memory cache trước (tránh re-run expensive tools)
            String cacheKey = "tool_result:" + toolUse.name() + ":" + toolUse.input().hashCode();
            String cachedResult = (String) state.workingMemory().get(cacheKey);

            String result;
            if (cachedResult != null) {
                System.out.println("  (from cache) " + truncate(cachedResult, 100));
                result = cachedResult;
            } else {
                result = runTool(toolUse);
                System.out.println("  → " + truncate(result, 200));
                // Cache result (nhưng không lưu vào DB ngay, sẽ lưu cùng state)
            }

            results.add(ToolResultBlockParam.builder()
                .toolUseId(toolUse.id())
                .content(result)
                .build());
        }

        return results;
    }

    private String runTool(ToolUseBlock toolUse) {
        ToolExecutor executor = tools.get(toolUse.name());
        if (executor == null) {
            return "Error: Unknown tool '" + toolUse.name() + "'";
        }
        try {
            return executor.execute(toolUse.input());
        } catch (Exception e) {
            return "Tool error: " + e.getMessage();
        }
    }

    private Message callClaude(List<MessageParam> history) {
        return client.messages().create(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_SONNET_4_6)
                .system(ReActPrompts.REACT_SYSTEM_PROMPT)
                .messages(history)
                .tools(new ArrayList<>(buildToolDefinitions()))
                .maxTokens(2048)
                .build()
        );
    }

    // ── Serialization helpers ───────────────────────────────────────────────

    private List<MessageParam> deserializeHistory(List<SerializedMessage> serialized) {
        // Convert SerializedMessage → MessageParam
        // Implementation details depend on Anthropic SDK version
        // Simplified: rebuild from serialized JSON
        return serialized.stream()
            .map(this::deserializeMessage)
            .filter(Objects::nonNull)
            .toList();
    }

    private MessageParam deserializeMessage(SerializedMessage msg) {
        return switch (msg.contentType()) {
            case "text" -> MessageParam.builder()
                .role(MessageParam.Role.valueOf(msg.role().toUpperCase()))
                .content(msg.content())
                .build();
            // tool_use and tool_result require more complex deserialization
            default -> null; // Simplified for example
        };
    }

    private AgentState addResponseToState(AgentState state, Message response) {
        // Serialize response content và thêm vào state
        String serialized = serializeContent(response.content());
        return state.withMessage(new SerializedMessage(
            "assistant", "mixed", serialized, null, null));
    }

    private CompletedStep buildCompletedStep(int stepNumber, Message response,
                                              List<ToolResultBlockParam> results) {
        String toolName = response.content().stream()
            .filter(b -> b instanceof ToolUseBlock)
            .map(b -> ((ToolUseBlock) b).name())
            .findFirst().orElse("unknown");

        return new CompletedStep(
            stepNumber, "Tool call: " + toolName,
            toolName, Map.of(), results.get(0).content().toString(),
            true, 0L, java.time.Instant.now()
        );
    }

    // ... other helpers (buildUserMessage, buildAssistantMessage, extractText, etc.)

    @FunctionalInterface
    interface ToolExecutor {
        String execute(Map<String, Object> input) throws Exception;
    }

    private String truncate(String s, int max) {
        return s.length() > max ? s.substring(0, max) + "..." : s;
    }
}
```

---

## 7. Idempotency — Thiết kế Tools an toàn để Retry

Khi agent bị interrupt và resume, **tool calls có thể bị execute nhiều lần**. Tool cần được thiết kế để safe khi call nhiều lần với cùng arguments.

### Bad: Non-idempotent tool

```java
// NGUY HIỂM: Nếu agent resume, email sẽ bị gửi lần 2!
public String sendReviewComment(String prId, String comment) {
    githubClient.createComment(prId, comment);
    return "Comment posted";
}
```

### Good: Idempotent tool với idempotency key

```java
// SAFE: Nếu execute lần 2 với cùng key, skip
public String sendReviewComment(String prId, String comment) {
    // Tạo idempotency key từ content hash
    String idempotencyKey = "comment:" + prId + ":" + hash(comment);

    // Check xem đã execute chưa
    if (idempotencyStore.exists(idempotencyKey)) {
        return "Comment already posted (idempotent skip)";
    }

    // Execute và mark
    String commentId = githubClient.createComment(prId, comment);
    idempotencyStore.mark(idempotencyKey, commentId, Duration.ofDays(7));
    return "Comment posted: " + commentId;
}
```

### Idempotency patterns cho common tool types

| Tool type | Idempotency approach |
|-----------|---------------------|
| Read operations | Tự nhiên idempotent |
| Write to file | Overwrite = idempotent (content-based) |
| Create DB record | Unique constraint + ON CONFLICT DO NOTHING |
| Send email/notification | Idempotency key trong request |
| Call external API | Idempotency key header (nhiều APIs support) |
| Run shell command | Check output/side-effect trước khi run |

---

## 8. Cleanup Strategy

Checkpoints cần được cleanup để tránh database bloat:

```java
package com.course.module06.lesson03;

import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Component
public class CheckpointCleanupJob {

    private final CheckpointStore store;

    public CheckpointCleanupJob(CheckpointStore store) {
        this.store = store;
    }

    /**
     * Chạy hằng ngày lúc 2 giờ sáng.
     * Xóa checkpoints đã complete hơn 24 giờ.
     */
    @Scheduled(cron = "0 0 2 * * *")
    public void cleanupCompletedCheckpoints() {
        int deleted = store.deleteExpired(24); // 24 hours
        System.out.printf("Checkpoint cleanup: deleted %d completed checkpoints%n", deleted);
    }

    /**
     * Chạy hằng tuần để alert về orphaned tasks.
     * Incomplete tasks > 7 ngày có thể là bugs.
     */
    @Scheduled(cron = "0 0 9 * * MON")
    public void reportOrphanedTasks() {
        List<String> incomplete = store.listIncomplete();

        // Alert nếu có quá nhiều incomplete tasks
        if (incomplete.size() > 10) {
            System.err.printf("WARNING: %d incomplete agent tasks detected. " +
                "Possible agent failures: %s%n",
                incomplete.size(), incomplete.subList(0, Math.min(5, incomplete.size())));
            // Trong production: gửi alert đến PagerDuty, Slack, etc.
        }
    }
}
```

---

## 9. Exercise: Add Checkpointing to Code Review Pipeline

### Bài tập

Lấy Java Code Review Pipeline từ Module 04 và thêm checkpointing vào.

**Yêu cầu:**

1. **Wrap pipeline trong CheckpointedReActAgent**: Pipeline hiện tại chạy end-to-end mà không có checkpoint. Wrap nó để:
   - Mỗi file được review xong → save checkpoint
   - Nếu pipeline bị interrupt giữa chừng → resume từ file tiếp theo

2. **Implement PostgreSQL CheckpointStore**: Dùng schema SQL được cung cấp ở trên.

3. **Test interruption và resumption**:
   ```java
   @Test
   public void testResumeAfterInterruption() throws Exception {
       String taskId = "test-task-" + UUID.randomUUID();
       String task = "Review PR #123 with 10 Java files";

       // Simulate first run — fails after step 5
       CheckpointStore store = new PostgresCheckpointStore(jdbc);
       agent.runUntilStep(taskId, task, 5); // Custom test method

       // Verify checkpoint was saved
       Optional<AgentState> checkpoint = store.load(taskId);
       assertTrue(checkpoint.isPresent());
       assertEquals(5, checkpoint.get().currentStepIndex());

       // Resume from checkpoint
       String result = agent.run(taskId, task); // Should resume from step 6
       assertNotNull(result);

       // Verify all 10 files were reviewed
       AgentState finalState = store.load(taskId).orElseThrow();
       assertTrue(finalState.isComplete());
       assertEquals(10, finalState.completedSteps().size());
   }
   ```

4. **Idempotency**: Đảm bảo `postGitHubComment` tool là idempotent.

5. **Cleanup**: Thêm `@Scheduled` cleanup job.

**Checklist:**
- [ ] Pipeline resume đúng từ step N sau restart
- [ ] Không có duplicate GitHub comments (idempotent)
- [ ] Unit tests cho PostgresCheckpointStore
- [ ] Integration test cho resume behavior
- [ ] Cleanup job xóa checkpoints cũ hơn 24h

---

## Tóm tắt

| Khái niệm | Key takeaway |
|-----------|-------------|
| **Tại sao cần** | Long-running agents fail; restart tốn kém |
| **State types** | History, plan, tool cache, working memory, artifacts |
| **Stateless vs Stateful** | Stateless = đơn giản, Stateful = resumable |
| **AgentState design** | Immutable records, versioned, fully serializable |
| **PostgreSQL store** | Production default — JSONB, upsert, indexed |
| **Redis store** | Fast cache layer, TTL built-in |
| **File store** | Dev/test chỉ |
| **Idempotency** | Tools phải safe to re-execute sau resume |
| **Cleanup** | Scheduled jobs để delete expired checkpoints |

---

> **Trước đó**: [Lesson 02 — Planning Patterns](02-planning-patterns.md)
> **Tiếp theo**: Lesson 04 — Structured Output & Output Validation _(coming soon)_
