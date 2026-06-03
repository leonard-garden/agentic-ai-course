# Lesson 04: Agent Memory Systems — Beyond the Context Window

> **Module 06 — Advanced Patterns**
> Prerequisite: Đã hoàn thành Module 00–05, đặc biệt Module 05 (IntelliOps incident response agent)
> Thời gian ước tính: 90 phút đọc + 60 phút lab

---

## Mở đầu: Vấn đề bộ nhớ của Agent

Hãy tưởng tượng bạn thuê một kỹ sư mới. Tuần đầu, bạn giải thích toàn bộ kiến trúc hệ thống. Tuần sau, họ quên sạch — bạn phải giải thích lại từ đầu. Tuần tiếp theo, lại quên. Đó chính xác là vấn đề của LLM-based agent khi không có memory system.

**Context window là bộ nhớ làm việc (working memory)** của LLM. Claude 3.5 Sonnet có 200K tokens — khá lớn, nhưng:

- Một cuộc hội thoại dài 10 ngày sẽ vượt quá giới hạn này
- Chi phí tỷ lệ thuận với số token: 200K tokens mỗi request = rất đắt
- Mỗi conversation là một "session" độc lập — agent không nhớ gì từ session trước

Đây không phải lỗi thiết kế — đây là đặc tính cơ bản của transformer architecture. Giải pháp không phải là "tăng context window mãi lên" mà là **thiết kế memory system bên ngoài LLM**.

---

## 4 Loại Agent Memory

Lấy cảm hứng từ cognitive science (khoa học nhận thức ở người), chúng ta phân loại bộ nhớ agent thành 4 tầng:

```
┌─────────────────────────────────────────────────────────────┐
│                    AGENT MEMORY TAXONOMY                     │
├──────────────────────┬──────────────────────────────────────┤
│  Loại               │  Đặc điểm                             │
├──────────────────────┼──────────────────────────────────────┤
│  In-Context          │  RAM: nhanh, đắt, mất khi tắt        │
│  (Working Memory)    │  → current conversation messages      │
├──────────────────────┼──────────────────────────────────────┤
│  External            │  Ổ cứng: bền vững, truy xuất chậm hơn│
│  (Long-term Memory)  │  → vector DB, SQL, file system        │
├──────────────────────┼──────────────────────────────────────┤
│  Episodic Memory     │  Nhật ký sự kiện cụ thể               │
│                      │  → "Tôi đã giúp user X xử lý lỗi Y"  │
├──────────────────────┼──────────────────────────────────────┤
│  Semantic Memory     │  Kiến thức/sự thật đã chắt lọc        │
│                      │  → "User X thích constructor injection"│
└──────────────────────┴──────────────────────────────────────┘
```

### 1. In-Context Memory (Working Memory)

Đây là những gì bạn truyền trực tiếp vào mảng `messages[]`. Agent "nhớ" mọi thứ trong đó — cho đến khi request kết thúc.

**Ưu điểm:** Không cần infrastructure, zero latency, LLM có thể reasoning trực tiếp trên đó.

**Nhược điểm:** Tốn token (= tốn tiền), mất khi session kết thúc, bị giới hạn bởi context window.

**Dùng khi:** Single-session tasks, short conversations, khi bạn cần agent có đầy đủ context ngay lập tức.

### 2. External Memory (Long-term Memory)

Lưu trữ ngoài LLM — thường là vector database hoặc SQL. Agent truy xuất thông tin liên quan trước khi generate response.

**Ưu điểm:** Persistent (bền vững), scalable, có thể lưu hàng triệu documents.

**Nhược điểm:** Thêm latency (retrieval step), chất lượng phụ thuộc vào embedding model và chunking strategy.

**Dùng khi:** Multi-session assistants, enterprise knowledge bases, codebase Q&A.

### 3. Episodic Memory

Ghi lại **các sự kiện cụ thể đã xảy ra**: "Ngày 15/3, tôi debug incident OOM trên service payments. Root cause là connection pool leak. Fix: tăng timeout + thêm pool eviction policy."

Đây là bộ nhớ **theo dạng tường thuật** (narrative). Agent dùng episodic memory để tránh lặp lại sai lầm, học từ kinh nghiệm, và cung cấp context cho tasks tương tự trong tương lai.

### 4. Semantic Memory

Là **kiến thức đã được chắt lọc và tổng quát hóa** từ nhiều episodes. Ví dụ sau 20 lần debug cùng một user, agent rút ra: "User này làm việc trên monorepo Gradle, prefer Kotlin coroutines, không dùng Lombok."

Semantic memory ít volatile hơn episodic — nó thay đổi chậm và được cập nhật thông qua quá trình "reflection" (agent tự tổng kết).

---

## RAG cho Agent: Không chỉ để search documents

RAG (Retrieval-Augmented Generation) thường được biết đến trong context "chatbot với tài liệu công ty". Nhưng trong agent systems, RAG mạnh hơn nhiều — nó là **backbone của external memory**.

### Flow chuẩn của RAG trong Agent

```
User/Agent query
      │
      ▼
  Embed query          ← text → vector (768-3072 dimensions)
      │
      ▼
  Vector search        ← cosine similarity trong DB
      │
      ▼
  Retrieve top-K docs  ← thường K = 3-10
      │
      ▼
  Inject vào context   ← thêm vào system prompt hoặc user message
      │
      ▼
  LLM generates        ← với context đã được enrich
```

### Công cụ vector DB phổ biến

| Tool | Đặc điểm | Java Integration |
|---|---|---|
| **pgvector** | PostgreSQL extension, no extra infra | Spring AI PgVectorStore |
| **ChromaDB** | Python-native, tốt cho prototype | REST API |
| **Pinecone** | Managed, scalable | Official Java SDK |
| **Weaviate** | GraphQL API, hybrid search built-in | Java client |

**Khuyến nghị cho Java backend engineer:** Bắt đầu với `pgvector` — bạn đã có PostgreSQL, không cần infra mới, Spring AI hỗ trợ first-class.

---

## Triển khai pgvector với Spring AI

### Dependency

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
        <!-- hoặc spring-ai-anthropic-spring-boot-starter -->
    </dependency>
</dependencies>
```

### application.yml

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/agentdb
    username: postgres
    password: secret
  ai:
    vectorstore:
      pgvector:
        index-type: HNSW
        distance-type: COSINE_DISTANCE
        dimensions: 1536   # text-embedding-3-small
```

### Schema SQL

```sql
-- Chạy migration này trước
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE IF NOT EXISTS vector_store (
    id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    content     TEXT NOT NULL,
    metadata    JSONB,
    embedding   VECTOR(1536)
);

CREATE INDEX ON vector_store USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

### AgentMemoryService — Core Service

```java
@Service
@Slf4j
public class AgentMemoryService {

    private final VectorStore vectorStore;
    private final EmbeddingClient embeddingClient;

    public AgentMemoryService(VectorStore vectorStore,
                               EmbeddingClient embeddingClient) {
        this.vectorStore = vectorStore;
        this.embeddingClient = embeddingClient;
    }

    /**
     * Lưu một memory vào vector store.
     * agentId dùng để phân tách memory của các agent khác nhau.
     */
    public void remember(String agentId, String content,
                         Map<String, Object> metadata) {
        var enrichedMetadata = new HashMap<>(metadata);
        enrichedMetadata.put("agentId", agentId);
        enrichedMetadata.put("timestamp", Instant.now().toString());
        enrichedMetadata.put("contentHash", sha256(content));

        var doc = new Document(content, enrichedMetadata);
        vectorStore.add(List.of(doc));
        log.debug("Stored memory for agent={}: {}", agentId,
                  content.substring(0, Math.min(80, content.length())));
    }

    /**
     * Truy xuất các memories liên quan nhất với query hiện tại.
     */
    public List<String> recall(String agentId, String query, int topK) {
        var request = SearchRequest.query(query)
                .withTopK(topK)
                .withSimilarityThreshold(0.7)   // lọc bỏ kết quả kém liên quan
                .withFilterExpression("agentId == '" + agentId + "'");

        return vectorStore.similaritySearch(request)
                .stream()
                .map(Document::getContent)
                .toList();
    }

    /**
     * Inject relevant memories vào đầu danh sách messages.
     * Agent sẽ thấy past context trước khi xử lý task hiện tại.
     */
    public void injectMemory(String agentId, String currentTask,
                              List<MessageParam> messages) {
        var relevant = recall(agentId, currentTask, 5);
        if (relevant.isEmpty()) return;

        var memoryContext = """
                === Relevant past context (from memory) ===
                %s
                === End of memory context ===
                """.formatted(String.join("\n---\n", relevant));

        // Thêm vào đầu messages dưới dạng system message
        messages.add(0, MessageParam.builder()
                .role(Role.USER)
                .content(memoryContext + "\n\nNow, current task: " + currentTask)
                .build());
    }

    private String sha256(String input) {
        try {
            var digest = MessageDigest.getInstance("SHA-256");
            var hash = digest.digest(input.getBytes(StandardCharsets.UTF_8));
            return HexFormat.of().formatHex(hash);
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException(e);
        }
    }
}
```

---

## Episodic Memory: Học từ Quá khứ

Episodic memory là "nhật ký" của agent — ghi lại chính xác những gì đã xảy ra, quyết định gì đã được đưa ra, và kết quả như thế nào.

### Episode Record

```java
/**
 * Một episode = một lần agent hoàn thành (hoặc thất bại) một task.
 */
public record Episode(
    String id,
    String agentId,
    String task,               // mô tả task
    String approach,           // agent đã làm gì
    String outcome,            // kết quả (tóm tắt)
    boolean successful,        // thành công hay thất bại
    Map<String, Object> tags,  // metadata: service, errorType, severity...
    Instant timestamp
) {
    public static Episode success(String agentId, String task,
                                   String approach, String outcome,
                                   Map<String, Object> tags) {
        return new Episode(
            UUID.randomUUID().toString(), agentId, task,
            approach, outcome, true, tags, Instant.now()
        );
    }

    public static Episode failure(String agentId, String task,
                                   String approach, String failureReason,
                                   Map<String, Object> tags) {
        return new Episode(
            UUID.randomUUID().toString(), agentId, task,
            approach, "FAILED: " + failureReason, false, tags, Instant.now()
        );
    }

    /**
     * Chuyển episode thành text để embedding và lưu vào vector store.
     */
    public String toMemoryText() {
        return """
            Task: %s
            Approach: %s
            Outcome: %s
            Status: %s
            Time: %s
            """.formatted(task, approach, outcome,
                          successful ? "SUCCESS" : "FAILURE",
                          timestamp.toString());
    }
}
```

### EpisodicMemoryStore

```java
@Service
public class EpisodicMemoryStore {

    private final AgentMemoryService memoryService;

    public EpisodicMemoryStore(AgentMemoryService memoryService) {
        this.memoryService = memoryService;
    }

    /**
     * Sau mỗi task completion, ghi episode vào memory.
     */
    public void recordEpisode(Episode episode) {
        var metadata = new HashMap<String, Object>(episode.tags());
        metadata.put("type", "episode");
        metadata.put("successful", String.valueOf(episode.successful()));
        metadata.put("episodeId", episode.id());

        memoryService.remember(
            episode.agentId(),
            episode.toMemoryText(),
            metadata
        );
    }

    /**
     * Trước khi xử lý task mới, lấy các episodes tương tự để học từ đó.
     */
    public List<String> recallSimilarEpisodes(String agentId,
                                               String newTask, int topK) {
        return memoryService.recall(agentId, newTask, topK);
    }

    /**
     * Chỉ lấy các episodes thành công — để học cách làm đúng.
     */
    public List<String> recallSuccessfulApproaches(String agentId,
                                                     String taskDescription) {
        // Retrieval theo metadata filter
        var request = SearchRequest.query(taskDescription)
                .withTopK(5)
                .withFilterExpression(
                    "agentId == '" + agentId + "' && successful == 'true'"
                );

        return vectorStore.similaritySearch(request)
                .stream()
                .map(Document::getContent)
                .toList();
    }
}
```

### Sử dụng trong Incident Response Agent (kết nối với Module 05)

```java
@Service
public class IntelliOpsMemoryEnhancedAgent {

    private final ClaudeClient claude;
    private final EpisodicMemoryStore episodicStore;
    private final AgentMemoryService memoryService;

    private static final String AGENT_ID = "intelliops-incident-v2";

    public IncidentResolution handleIncident(IncidentReport incident) {
        // 1. Recall similar past incidents
        var pastEpisodes = episodicStore.recallSimilarEpisodes(
            AGENT_ID,
            "incident: " + incident.errorMessage() + " service: " + incident.service(),
            5
        );

        // 2. Build context với past experience
        var systemPrompt = buildSystemPromptWithMemory(pastEpisodes);

        // 3. Run agent (giống Module 05 nhưng với enriched context)
        var result = runIncidentResponseAgent(incident, systemPrompt);

        // 4. Record episode để học cho lần sau
        var episode = result.resolved()
            ? Episode.success(AGENT_ID,
                "Incident in " + incident.service() + ": " + incident.errorMessage(),
                result.approachSummary(),
                result.resolution(),
                Map.of("service", incident.service(), "severity", incident.severity()))
            : Episode.failure(AGENT_ID,
                "Incident in " + incident.service() + ": " + incident.errorMessage(),
                result.approachSummary(),
                result.failureReason(),
                Map.of("service", incident.service()));

        episodicStore.recordEpisode(episode);

        return result;
    }

    private String buildSystemPromptWithMemory(List<String> pastEpisodes) {
        var basePrompt = """
            You are IntelliOps, an expert incident response agent for Java microservices.
            You analyze alerts, investigate root causes, and implement fixes.
            """;

        if (pastEpisodes.isEmpty()) return basePrompt;

        return basePrompt + """

            === PAST INCIDENT EXPERIENCE ===
            You have handled similar incidents before. Learn from these:

            %s
            === END OF PAST EXPERIENCE ===

            Use your past experience to inform your approach,
            but always verify assumptions with current data.
            """.formatted(String.join("\n\n", pastEpisodes));
    }
}
```

---

## Semantic Memory: Kiến thức Chắt lọc

Trong khi episodic memory lưu **sự kiện cụ thể**, semantic memory lưu **kiến thức tổng quát** đã được rút ra từ nhiều sự kiện.

Ví dụ: Sau 30 lần handle incidents trên service `payments-service`, agent có thể rút ra:
- "payments-service thường bị OOM vào peak hours (9-11am)"
- "Root cause thường là connection pool exhaustion, không phải memory leak"
- "Fix nhanh nhất: restart pod + tăng HikariCP pool size từ 10 lên 20"

Đây là semantic memory — không gắn với một incident cụ thể, mà là **pattern knowledge**.

### Reflection: Quá trình tạo Semantic Memory

```java
@Component
public class AgentReflectionService {

    private final EpisodicMemoryStore episodicStore;
    private final AgentMemoryService memoryService;
    private final ClaudeClient claude;

    /**
     * Chạy định kỳ (ví dụ: mỗi ngày) để distill episodes thành semantic facts.
     * Đây là quá trình "reflection" — agent tự học từ kinh nghiệm.
     */
    @Scheduled(cron = "0 0 2 * * *")   // 2am mỗi ngày
    public void reflectAndLearn() {
        var agentId = "intelliops-incident-v2";

        // Lấy các episodes trong 7 ngày gần đây
        var recentEpisodes = episodicStore.recallSimilarEpisodes(
            agentId, "incident service failure", 50
        );
        if (recentEpisodes.size() < 5) return;  // chưa đủ data

        // Nhờ Claude tổng kết và rút ra learnings
        var reflectionPrompt = """
            You are analyzing your own past incident responses to extract learnings.

            Recent episodes:
            %s

            Extract 3-5 concrete, actionable facts you've learned:
            - Patterns about which services are prone to which failures
            - Effective approaches that worked repeatedly
            - Approaches that failed and why
            - User/team preferences and constraints

            Format each fact as a single sentence starting with a service name or pattern.
            Example: "payments-service typically fails due to connection pool exhaustion during peak hours."
            """.formatted(String.join("\n---\n", recentEpisodes));

        var learnings = claude.complete(reflectionPrompt);

        // Lưu vào semantic memory
        memoryService.remember(
            agentId,
            "SEMANTIC KNOWLEDGE (distilled from experience):\n" + learnings,
            Map.of("type", "semantic", "reflectedAt", Instant.now().toString())
        );
    }
}
```

---

## Memory Decay và Cleanup

Memory vô hạn không phải là điều tốt — stale memories có thể mislead agent. Cần có chiến lược dọn dẹp.

### TTL-based Cleanup

```java
@Component
public class MemoryMaintenanceService {

    private final JdbcTemplate jdbc;

    /**
     * Xóa memories cũ hơn N ngày.
     * Semantic memories giữ lâu hơn episodic memories.
     */
    @Scheduled(cron = "0 0 3 * * *")   // 3am mỗi ngày
    public void cleanupStaleMemories() {
        // Episodic: giữ 90 ngày
        int deletedEpisodic = jdbc.update("""
            DELETE FROM vector_store
            WHERE metadata->>'type' = 'episode'
            AND (metadata->>'timestamp')::timestamptz < NOW() - INTERVAL '90 days'
            """);

        // Semantic: giữ 1 năm (ít thay đổi hơn)
        int deletedSemantic = jdbc.update("""
            DELETE FROM vector_store
            WHERE metadata->>'type' = 'semantic'
            AND (metadata->>'reflectedAt')::timestamptz < NOW() - INTERVAL '365 days'
            """);

        log.info("Memory cleanup: deleted {} episodic, {} semantic memories",
                 deletedEpisodic, deletedSemantic);
    }

    /**
     * Compression: tóm tắt nhiều episodes cũ thành một semantic summary.
     * Giảm storage cost, giữ knowledge.
     */
    public void compressOldEpisodes(String agentId, String topic) {
        // Lấy episodes cũ liên quan đến topic
        var oldEpisodes = episodicStore.recallSimilarEpisodes(agentId, topic, 20);
        if (oldEpisodes.size() < 10) return;

        // Tóm tắt
        var summary = claude.complete(
            "Compress these incident episodes into a concise summary "
            + "preserving key patterns and learnings:\n\n"
            + String.join("\n---\n", oldEpisodes)
        );

        // Lưu summary, xóa originals
        memoryService.remember(agentId, "COMPRESSED: " + summary,
            Map.of("type", "compressed", "topic", topic));
        // ... xóa các episodes gốc (bỏ qua chi tiết)
    }
}
```

---

## SYNAPSE: Graph-based Memory (Approach mới nhất — Jan 2026)

Flat vector similarity search có một hạn chế: nó chỉ tìm documents **tương tự về text**, không tìm được documents **liên quan về mặt logic**.

Ví dụ: Query "payments service error" sẽ tìm được incidents về payments, nhưng có thể bỏ qua một incident về database (root cause thực sự) nếu text không giống nhau.

**SYNAPSE** (Spreading Activation Network for Agent Personal Semantic Experience) dùng **graph structure** thay vì flat vector store:

```
[Incident: payments OOM]──────────────[Service: payments-service]
           │                                        │
     [Cause: DB pool]──────────[Config: HikariCP]──┘
           │
     [Fix: pool size +10]──────[Past Fix: 2025-01-15]
```

Khi query "payments issue", activation **spreads** qua graph — tìm được cả DB pool config, past fixes, và related services.

**Implementation consideration:** Graph-based memory phức tạp hơn đáng kể. Cần:
- Graph database (Neo4j) hoặc pgvector + relationship table
- Entity extraction để xây dựng graph từ episodes
- Graph traversal algorithm (BFS/DFS with relevance scoring)

**Khuyến nghị:** Dùng SYNAPSE khi flat vector search không đủ (production systems with complex interdependencies). Bắt đầu với pgvector + flat search — đủ tốt cho 80% use cases.

---

## Khi nào dùng loại Memory nào?

```
┌────────────────────────────────────────────────────────────┐
│                MEMORY SELECTION GUIDE                       │
├─────────────────────────┬──────────────────────────────────┤
│  Scenario               │  Memory Strategy                  │
├─────────────────────────┼──────────────────────────────────┤
│  One-off task           │  In-context only                  │
│  (< 30 min, single use) │                                   │
├─────────────────────────┼──────────────────────────────────┤
│  Multi-session          │  External memory (pgvector)       │
│  user assistant         │  + Episodic                       │
├─────────────────────────┼──────────────────────────────────┤
│  Enterprise knowledge   │  RAG với semantic chunking        │
│  base / codebase Q&A    │  pgvector + HNSW index            │
├─────────────────────────┼──────────────────────────────────┤
│  Agent that improves    │  Episodic + Semantic              │
│  over time              │  + Scheduled reflection           │
├─────────────────────────┼──────────────────────────────────┤
│  Complex multi-agent    │  Shared external memory           │
│  systems                │  + SYNAPSE (production)           │
└─────────────────────────┴──────────────────────────────────┘
```

---

## Lab Exercise: Thêm Memory vào IntelliOps Agent

Bạn đã xây dựng IntelliOps incident response agent ở Module 05. Bây giờ, nâng cấp nó để **nhớ và học từ quá khứ**.

### Yêu cầu

1. **Setup pgvector:**
   - Thêm pgvector extension vào PostgreSQL
   - Thêm Spring AI pgvector dependency
   - Tạo schema và HNSW index

2. **Integrate AgentMemoryService:**
   - Sau mỗi incident resolution → ghi Episode
   - Trước khi xử lý incident mới → recall 5 similar past incidents
   - Inject past incidents vào system prompt

3. **Implement Reflection:**
   - Scheduled job mỗi ngày: Claude tổng kết 10 recent episodes → extract 3 learnings → lưu semantic memory

4. **Verify agent improves:**
   - Simulate 5 incidents cùng loại (ví dụ: OOM trên same service)
   - Incident thứ 5 agent phải nhận ra pattern và đề xuất fix nhanh hơn

### Checklist

- [ ] pgvector schema created và migrated
- [ ] AgentMemoryService injected vào IntelliOpsAgent
- [ ] Mỗi incident resolution tạo Episode record
- [ ] System prompt enriched với relevant past episodes
- [ ] Reflection job chạy (hoặc manual trigger cho test)
- [ ] Test: incident thứ 5 nhắc đến pattern từ past

---

## Tóm tắt

| Concept | Key Takeaway |
|---|---|
| Context window | Working memory — fast nhưng ephemeral |
| External memory | pgvector + Spring AI = production-ready cho Java |
| Episodic memory | Record sau mỗi task → agent học từ lịch sử |
| Semantic memory | Reflection job → distill episodes → lasting knowledge |
| Memory decay | TTL cleanup + compression → tránh stale memories |
| SYNAPSE | Graph > flat vector khi relationships matter |

**Lesson tiếp theo:** Lesson 05 — Data Ingestion Pipeline: cách feed knowledge base cho agent một cách hệ thống và hiệu quả.

---

> **Ghi chú cho instructor:** Lab exercise có thể kết hợp với Lesson 05 (ingestion pipeline) để tạo một end-to-end project: ingest codebase → agent có memory về codebase → xử lý incidents với context đầy đủ.
