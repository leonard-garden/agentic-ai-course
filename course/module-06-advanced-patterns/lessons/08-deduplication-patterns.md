# Lesson 08: Deduplication in Multi-Agent Pipelines

> **Module 06 — Advanced Agentic Patterns**
> Prerequisite: Lessons 01-07
> Thời lượng ước tính: 75 phút

---

## Tại Sao Deduplication Quan Trọng?

Khi bạn chạy nhiều agents song song — ví dụ 4 agents cùng review một Java codebase từ các góc độ khác nhau (security, performance, architecture, testing) — một vấn đề tự nhiên xuất hiện: **tất cả 4 agents có thể phát hiện ra cùng một lỗi**.

Agent A nói: "Method `processPayment()` thiếu null check."
Agent B nói: "Có thể xảy ra NullPointerException trong `processPayment()`."
Agent C nói: "Lỗ hổng tiềm ẩn: không kiểm tra input trước khi xử lý trong payment flow."

Đây là cùng một issue, được nói theo 3 cách khác nhau. Nếu không dedup, người dùng nhận được report với 3 "findings" riêng biệt — confusing, và tốn tokens để generate.

Deduplication trong multi-agent systems có 3 cấp độ:

```
┌─────────────────────────────────────────────────────────────┐
│                  DEDUPLICATION LEVELS                        │
├───────────────────┬─────────────────────────────────────────┤
│ Level 1           │ Tool Call Deduplication                 │
│                   │ Cùng tool call không chạy 2 lần         │
├───────────────────┼─────────────────────────────────────────┤
│ Level 2           │ Finding Deduplication                   │
│                   │ Cùng issue không báo cáo 2 lần          │
├───────────────────┼─────────────────────────────────────────┤
│ Level 3           │ Knowledge Base Deduplication            │
│                   │ Cùng document không index 2 lần         │
└───────────────────┴─────────────────────────────────────────┘
```

---

## Level 1: Tool Call Deduplication

### Vấn Đề

Trong một pipeline với 4 agents chạy song song, tất cả đều cần đọc file `PaymentService.java`. Không có caching, file này sẽ được đọc 4 lần từ disk (hoặc gọi GitHub API 4 lần nếu đây là remote codebase). Lãng phí I/O, lãng phí cost, lãng phí latency.

Tương tự, nếu có tool `searchVectorStore("null pointer exception handling")`, 3 agents có thể gọi tool này với gần như cùng query. Kết quả giống nhau nhưng vector search được thực hiện 3 lần.

### Solution: Shared Tool Result Cache

```java
@Component
public class CachingToolExecutor {

    private final ToolExecutor underlying;
    private final Cache<String, ToolResult> resultCache;

    public CachingToolExecutor(ToolExecutor underlying) {
        this.underlying = underlying;
        this.resultCache = Caffeine.newBuilder()
            // TTL khác nhau cho các loại tool khác nhau
            .expireAfterWrite(5, TimeUnit.MINUTES)
            .maximumSize(1000)
            .recordStats() // Theo dõi hit rate
            .build();
    }

    public ToolResult execute(ToolCall toolCall) {
        // Cache key = tên tool + hash của tất cả arguments
        String cacheKey = buildCacheKey(toolCall);

        return resultCache.get(cacheKey, key -> {
            log.debug("Cache MISS for tool call: {}", toolCall.name());
            return underlying.execute(toolCall);
        });
    }

    private String buildCacheKey(ToolCall toolCall) {
        // Hash toàn bộ input để đảm bảo uniqueness
        String inputJson = JsonUtils.toJson(toolCall.input());
        String inputHash = Hashing.sha256()
            .hashString(inputJson, StandardCharsets.UTF_8)
            .toString()
            .substring(0, 16); // 16 chars đủ để unique
        return toolCall.name() + ":" + inputHash;
    }

    // Expose stats để monitor
    public CacheStats getCacheStats() {
        return resultCache.stats();
    }
}
```

### TTL Strategy Theo Loại Data

Không phải tất cả tool results đều có TTL giống nhau. Data thay đổi nhanh cần TTL ngắn; data stable có thể cache lâu hơn:

```java
@Component
public class TtlAwareCachingToolExecutor {

    private static final Map<String, Duration> TTL_BY_TOOL = Map.of(
        "read_file",         Duration.ofHours(1),      // File ít thay đổi
        "get_schema",        Duration.ofHours(4),      // Schema rất stable
        "search_logs",       Duration.ofSeconds(30),   // Logs thay đổi liên tục
        "get_recent_errors", Duration.ofSeconds(60),   // Error logs, cần fresh
        "search_docs",       Duration.ofMinutes(30),   // Documentation
        "run_static_analysis", Duration.ofMinutes(10)  // Analysis có thể stale sau commit
    );

    private final LoadingCache<String, ToolResult> tieredCache;

    public TtlAwareCachingToolExecutor(ToolExecutor underlying) {
        // Caffeine không hỗ trợ per-entry TTL trực tiếp
        // Workaround: store result với expiry timestamp
        this.tieredCache = Caffeine.newBuilder()
            .expireAfter(new Expiry<String, ToolResult>() {
                @Override
                public long expireAfterCreate(String key, ToolResult value, long currentTime) {
                    String toolName = extractToolName(key);
                    Duration ttl = TTL_BY_TOOL.getOrDefault(toolName, Duration.ofMinutes(5));
                    return ttl.toNanos();
                }
                @Override
                public long expireAfterUpdate(String key, ToolResult value,
                                               long currentTime, long currentDuration) {
                    return currentDuration; // Không extend khi update
                }
                @Override
                public long expireAfterRead(String key, ToolResult value,
                                             long currentTime, long currentDuration) {
                    return currentDuration; // Không extend khi read
                }
            })
            .build(key -> underlying.execute(parseToolCall(key)));
    }
}
```

### Measuring Cache Effectiveness

```java
@Scheduled(fixedRate = 60_000) // Mỗi phút
public void logCacheMetrics() {
    CacheStats stats = cachingExecutor.getCacheStats();
    log.info("Tool Cache Stats — Hit rate: {:.1f}%, Hits: {}, Misses: {}, Evictions: {}",
        stats.hitRate() * 100,
        stats.hitCount(),
        stats.missCount(),
        stats.evictionCount()
    );

    // Alert nếu hit rate quá thấp (cache không hiệu quả)
    if (stats.hitRate() < 0.3 && stats.requestCount() > 100) {
        alerting.sendAlert("Tool cache hit rate below 30% — consider reviewing cache key strategy");
    }
}
```

**Expected hit rates**: Với 4 agents chạy song song trên cùng codebase, bạn nên thấy hit rate > 60% cho file reads. Nếu thấp hơn, check xem agents có đang gọi tool với arguments hơi khác nhau không.

---

## Level 2: Finding Deduplication

### Vấn Đề Semantic Duplicate

Exact-match deduplication quá đơn giản. Hai findings có thể là cùng một issue nhưng được diễn đạt khác nhau:

- "Method lacks input validation" 
- "No null check before calling processPayment"
- "Potential NullPointerException in payment processing"

String equality không bắt được đây là cùng một vấn đề. Cần **semantic similarity**.

### Solution: Embedding-Based Deduplication

```java
@Service
public class FindingDeduplicator {

    private final EmbeddingClient embeddingClient;
    private static final float DEFAULT_SIMILARITY_THRESHOLD = 0.92f;

    /**
     * Loại bỏ duplicate findings dựa trên semantic similarity.
     *
     * @param findings  Danh sách findings từ nhiều agents
     * @param threshold Cosine similarity threshold (0.0 - 1.0)
     *                  0.95+ = near-exact duplicate
     *                  0.85-0.95 = semantic duplicate
     *                  < 0.85 = different findings
     */
    public List<Finding> deduplicate(List<Finding> findings, float threshold) {
        if (findings.size() <= 1) return findings;

        List<Finding> unique = new ArrayList<>();
        // Cache embeddings để không gọi API nhiều lần cho cùng finding
        Map<String, float[]> embeddingCache = new HashMap<>();

        for (Finding candidate : findings) {
            float[] candidateEmbedding = getOrComputeEmbedding(
                candidate, embeddingCache
            );

            boolean isDuplicate = unique.stream().anyMatch(existing -> {
                float[] existingEmbedding = getOrComputeEmbedding(
                    existing, embeddingCache
                );
                float similarity = cosineSimilarity(existingEmbedding, candidateEmbedding);
                return similarity >= threshold;
            });

            if (!isDuplicate) {
                unique.add(candidate);
                log.debug("Added unique finding: {}", candidate.shortDescription());
            } else {
                log.debug("Deduplicated finding: {}", candidate.shortDescription());
            }
        }

        log.info("Deduplication: {} findings → {} unique findings (removed {})",
            findings.size(), unique.size(), findings.size() - unique.size());

        return unique;
    }

    private float[] getOrComputeEmbedding(Finding finding,
                                           Map<String, float[]> cache) {
        return cache.computeIfAbsent(
            finding.id(),
            id -> embeddingClient.embed(finding.description())
        );
    }

    private float cosineSimilarity(float[] a, float[] b) {
        if (a.length != b.length) throw new IllegalArgumentException("Vector dimensions must match");

        double dotProduct = 0.0;
        double normA = 0.0;
        double normB = 0.0;

        for (int i = 0; i < a.length; i++) {
            dotProduct += a[i] * b[i];
            normA += a[i] * a[i];
            normB += b[i] * b[i];
        }

        if (normA == 0 || normB == 0) return 0.0f;
        return (float) (dotProduct / (Math.sqrt(normA) * Math.sqrt(normB)));
    }

    // Default threshold overload
    public List<Finding> deduplicate(List<Finding> findings) {
        return deduplicate(findings, DEFAULT_SIMILARITY_THRESHOLD);
    }
}
```

### Merge Strategy: Giữ Finding Tốt Nhất

Khi phát hiện duplicates, không chỉ đơn giản bỏ đi — hãy **merge** evidence từ nhiều agents:

```java
@Service
public class FindingMerger {

    private final FindingDeduplicator deduplicator;

    /**
     * Deduplicate và merge findings, giữ highest confidence version
     * với evidence được gộp từ tất cả duplicates.
     */
    public List<MergedFinding> deduplicateAndMerge(List<Finding> findings,
                                                     float threshold) {
        // Nhóm findings theo cluster (duplicates cùng nhóm)
        List<List<Finding>> clusters = clusterFindings(findings, threshold);

        return clusters.stream()
            .map(this::mergeCluster)
            .sorted(Comparator.comparing(MergedFinding::severity).reversed())
            .toList();
    }

    private List<List<Finding>> clusterFindings(List<Finding> findings, float threshold) {
        List<List<Finding>> clusters = new ArrayList<>();
        Set<String> assigned = new HashSet<>();

        for (Finding finding : findings) {
            if (assigned.contains(finding.id())) continue;

            List<Finding> cluster = new ArrayList<>();
            cluster.add(finding);
            assigned.add(finding.id());

            // Tìm tất cả duplicates
            for (Finding other : findings) {
                if (!assigned.contains(other.id()) &&
                    isSemanticallyDuplicate(finding, other, threshold)) {
                    cluster.add(other);
                    assigned.add(other.id());
                }
            }

            clusters.add(cluster);
        }

        return clusters;
    }

    private MergedFinding mergeCluster(List<Finding> cluster) {
        if (cluster.size() == 1) {
            return MergedFinding.fromSingle(cluster.get(0));
        }

        // Chọn finding có confidence cao nhất làm primary
        Finding primary = cluster.stream()
            .max(Comparator.comparing(Finding::confidence))
            .orElseThrow();

        // Gộp tất cả evidence
        List<String> allEvidence = cluster.stream()
            .flatMap(f -> f.evidence().stream())
            .distinct()
            .toList();

        // Gộp tất cả source agents
        List<String> sourceAgents = cluster.stream()
            .map(Finding::sourceAgent)
            .distinct()
            .toList();

        // Nếu nhiều agents cùng phát hiện → severity tăng lên
        Severity adjustedSeverity = cluster.size() >= 3
            ? escalateSeverity(primary.severity())
            : primary.severity();

        return new MergedFinding(
            primary.description(),
            primary.location(),
            adjustedSeverity,
            primary.confidence(),
            allEvidence,
            sourceAgents,
            cluster.size() // Số agents đồng ý
        );
    }

    private Severity escalateSeverity(Severity current) {
        return switch (current) {
            case LOW -> Severity.MEDIUM;
            case MEDIUM -> Severity.HIGH;
            case HIGH -> Severity.CRITICAL;
            case CRITICAL -> Severity.CRITICAL; // Không escalate cao hơn
        };
    }
}
```

### Threshold Calibration

Chọn threshold đúng rất quan trọng:

```
Threshold 0.98+ : Chỉ bắt near-exact duplicates (copy-paste findings)
                  Risk: nhiều semantic duplicates vẫn lọt qua

Threshold 0.90  : Bắt được hầu hết semantic duplicates
                  Risk: thỉnh thoảng merge findings khác nhau (false positive)

Threshold 0.85  : Aggressive dedup — ít duplicates nhưng mất some nuance
                  Use case: khi token budget rất chặt

Recommended: 0.90-0.92 cho code review findings
             0.88-0.90 cho security findings (nuance quan trọng hơn)
```

---

## Level 3: Knowledge Base Deduplication

### Vấn Đề Trong Ingestion Pipeline

Khi bạn chạy ingestion pipeline nhiều lần (scheduled re-indexing), hoặc khi nhiều sources có overlapping content, vector store sẽ tích lũy duplicates. Kết quả: RAG search trả về redundant chunks, wasting context window.

Hai loại duplicates cần handle:

1. **Exact duplicates**: Cùng file, chưa thay đổi — không cần re-index
2. **Updated content**: Cùng source nhưng nội dung đã thay đổi — cần xóa cũ, thêm mới
3. **Near-duplicates**: Hai documents khác nhau nhưng nội dung gần giống nhau

### Incremental Ingester với Content Hashing

```java
@Service
public class IncrementalIngester {

    private final VectorStore vectorStore;
    private final TextChunker chunker;
    // Persistent map: source URL/path → SHA-256 hash của content
    private final SourceHashRepository sourceHashRepo;

    /**
     * Ingest content chỉ khi thực sự có thay đổi.
     * Skip nếu content không đổi, replace nếu content mới.
     */
    public IngestionResult ingest(String source, String content) {
        String newHash = computeHash(content);
        Optional<String> existingHash = sourceHashRepo.findHash(source);

        // Case 1: Source chưa từng được index
        if (existingHash.isEmpty()) {
            log.info("New source, indexing: {}", source);
            return doIngest(source, content, newHash);
        }

        // Case 2: Content không thay đổi — skip
        if (existingHash.get().equals(newHash)) {
            log.debug("Skipping unchanged source: {}", source);
            return IngestionResult.skipped(source);
        }

        // Case 3: Content đã thay đổi — xóa cũ, thêm mới
        log.info("Content changed, re-indexing: {}", source);
        vectorStore.deleteByMetadata("source", source);
        return doIngest(source, content, newHash);
    }

    private IngestionResult doIngest(String source, String content, String hash) {
        List<TextChunk> chunks = chunker.chunk(content, ChunkConfig.builder()
            .chunkSize(512)
            .overlap(64)
            .metadata(Map.of("source", source, "hash", hash))
            .build());

        vectorStore.addAll(chunks);
        sourceHashRepo.upsert(source, hash);

        log.info("Indexed {} chunks from: {}", chunks.size(), source);
        return IngestionResult.indexed(source, chunks.size());
    }

    private String computeHash(String content) {
        return Hashing.sha256()
            .hashString(content, StandardCharsets.UTF_8)
            .toString();
    }
}
```

### Batch Ingestion với Metrics

```java
@Service
public class BatchIngester {

    private final IncrementalIngester incrementalIngester;

    public BatchIngestionReport ingestAll(List<SourceDocument> documents) {
        int indexed = 0, skipped = 0, failed = 0;
        List<String> errors = new ArrayList<>();

        for (SourceDocument doc : documents) {
            try {
                IngestionResult result = incrementalIngester.ingest(
                    doc.source(), doc.content()
                );
                if (result.wasIndexed()) indexed++;
                else skipped++;
            } catch (Exception e) {
                failed++;
                errors.add(doc.source() + ": " + e.getMessage());
                log.error("Failed to ingest {}: {}", doc.source(), e.getMessage());
            }
        }

        log.info("Batch ingestion complete — Indexed: {}, Skipped: {}, Failed: {}",
            indexed, skipped, failed);

        return new BatchIngestionReport(
            documents.size(), indexed, skipped, failed, errors
        );
    }
}
```

### Near-Duplicate Detection Khi Ingesting

Đôi khi hai sources khác nhau có nội dung gần giống (e.g., hai versions của cùng runbook, hay một document được copy và slightly modified):

```java
@Service
public class NearDuplicateDetector {

    private final VectorStore vectorStore;
    private static final float NEAR_DUPLICATE_THRESHOLD = 0.98f;

    /**
     * Kiểm tra xem document có near-duplicate trong vector store không.
     * Dùng trước khi index để tránh redundancy.
     */
    public Optional<String> findNearDuplicate(String content, String sourceToExclude) {
        // Tạo embedding cho document mới
        float[] embedding = embeddingClient.embed(summarize(content));

        // Tìm documents gần nhất trong vector store
        List<SearchResult> candidates = vectorStore.searchByEmbedding(embedding, 3);

        return candidates.stream()
            .filter(r -> !r.metadata("source").equals(sourceToExclude))
            .filter(r -> r.score() >= NEAR_DUPLICATE_THRESHOLD)
            .findFirst()
            .map(r -> r.metadata("source"));
    }

    public IngestionDecision shouldIngest(String source, String content) {
        Optional<String> nearDuplicate = findNearDuplicate(content, source);

        if (nearDuplicate.isPresent()) {
            return IngestionDecision.skip(
                "Near-duplicate of existing document: " + nearDuplicate.get()
            );
        }

        return IngestionDecision.proceed();
    }
}
```

---

## Orchestrator-Level Deduplication

Ngoài các cấp độ trên, orchestrator cũng cần dedup ở mức **task assignment**. Không nên gửi cùng một task cho 2 workers khác nhau.

### Task Signature Deduplication

```java
@Service
public class DeduplicatingOrchestrator {

    private final Set<String> completedTaskSignatures = ConcurrentHashMap.newKeySet();
    private final Set<String> inProgressSignatures = ConcurrentHashMap.newKeySet();

    /**
     * Kiểm tra xem task có nên được assign không.
     * False nếu task đã hoàn thành hoặc đang được xử lý.
     */
    public boolean shouldAssign(Task task) {
        String signature = computeTaskSignature(task);

        // Task đã hoàn thành → skip
        if (completedTaskSignatures.contains(signature)) {
            log.debug("Skipping already-completed task: {}", task.description());
            return false;
        }

        // Task đang được process → skip (tránh double assignment)
        if (!inProgressSignatures.add(signature)) {
            log.debug("Skipping in-progress task: {}", task.description());
            return false;
        }

        return true;
    }

    public void markCompleted(Task task) {
        String signature = computeTaskSignature(task);
        inProgressSignatures.remove(signature);
        completedTaskSignatures.add(signature);
    }

    public void markFailed(Task task) {
        String signature = computeTaskSignature(task);
        inProgressSignatures.remove(signature);
        // Không add vào completed → task có thể được retry
    }

    private String computeTaskSignature(Task task) {
        // Signature dựa trên task type + target
        // Ví dụ: "REVIEW_SECURITY:PaymentService.java:sha256abc123"
        return task.type() + ":" +
               task.targetIdentifier() + ":" +
               computeHash(task.targetContent());
    }
}
```

### Timing: Khi Nào Dedup?

```
Before task assignment:  Dùng task signature — tránh duplicate work
After agent completion:  Dùng finding dedup — merge results từ nhiều agents
Before synthesis:        Final dedup sweep — đảm bảo final report clean
During ingestion:        Content hash + near-duplicate check
```

---

## Cost vs Quality Trade-off

Deduplication không free. Embedding-based dedup gọi embedding API, tốn tiền và latency. Cần cân nhắc:

```
┌───────────────────────────────────────────────────────────┐
│                  DEDUP COST-QUALITY TRADEOFF               │
├────────────────────────┬──────────────────────────────────┤
│ Approach               │ Cost | Quality Impact             │
├────────────────────────┼──────────────────────────────────┤
│ No dedup               │ Low  | Duplicate findings confuse │
│ Exact string match     │ Low  | Misses semantic duplicates │
│ Embedding similarity   │ Med  | Good — catches most dups   │
│ LLM-based comparison   │ High | Best — understands nuance  │
└────────────────────────┴──────────────────────────────────┘
```

**Recommendation**: Dùng embedding similarity (Level 2) cho production. LLM-based comparison chỉ dùng cho high-value findings (Critical severity) khi bạn cần chắc chắn.

```java
@Service
public class HybridDeduplicator {

    private final FindingDeduplicator embeddingDeduplicator;
    private final LlmFindingComparator llmComparator;

    public List<Finding> deduplicate(List<Finding> findings) {
        // Bước 1: Fast embedding-based dedup cho tất cả
        List<Finding> embeddingDeduped = embeddingDeduplicator.deduplicate(findings, 0.90f);

        // Bước 2: Chỉ dùng LLM comparison cho CRITICAL findings
        //         để đảm bảo không miss any important nuance
        List<Finding> critical = embeddingDeduped.stream()
            .filter(f -> f.severity() == Severity.CRITICAL)
            .toList();

        List<Finding> nonCritical = embeddingDeduped.stream()
            .filter(f -> f.severity() != Severity.CRITICAL)
            .toList();

        List<Finding> criticalDeduped = llmComparator.deduplicate(critical);

        List<Finding> result = new ArrayList<>(nonCritical);
        result.addAll(criticalDeduped);
        return result;
    }
}
```

---

## Exercise: Finding Deduplication trong Code Review Pipeline

Bạn đã xây dựng Java Code Review Pipeline với 4 agents (Security, Performance, Architecture, Testing). Thêm deduplication:

**Yêu cầu:**
1. Sau khi tất cả 4 agents hoàn thành, collect tất cả findings
2. Chạy embedding-based deduplication với threshold 0.90
3. Merge duplicates: giữ highest confidence, gộp evidence
4. Nếu 3+ agents phát hiện cùng issue → tự động escalate severity lên một cấp
5. Log deduplication stats: bao nhiêu findings được merged

**Starter code:**
```java
@Service
public class CodeReviewPipeline {

    private final SecurityReviewAgent securityAgent;
    private final PerformanceReviewAgent performanceAgent;
    private final ArchitectureReviewAgent architectureAgent;
    private final TestingReviewAgent testingAgent;
    private final FindingMerger findingMerger;

    public CodeReviewReport review(String javaCode) {
        // Chạy tất cả agents song song
        List<Finding> allFindings = runAgentsInParallel(javaCode);

        // TODO: Add deduplication here
        List<MergedFinding> uniqueFindings = findingMerger.deduplicateAndMerge(
            allFindings, 0.90f
        );

        return CodeReviewReport.builder()
            .findings(uniqueFindings)
            .totalRaw(allFindings.size())
            .totalUnique(uniqueFindings.size())
            .build();
    }
}
```

**Tiêu chí hoàn thành:**
- [ ] Embedding dedup chạy sau tất cả agents complete
- [ ] Duplicate findings được merge, không chỉ dropped
- [ ] Severity escalation hoạt động khi 3+ agents agree
- [ ] Dedup stats được log
- [ ] Test với code sample có ít nhất 2 intentional duplicate findings

---

## Tóm Tắt

Deduplication là "invisible quality layer" trong multi-agent systems — người dùng không thấy nó hoạt động, nhưng sẽ nhận ra ngay khi nó vắng mặt qua reports rối rắm với nhiều redundant findings.

| Level | Vấn đề | Giải pháp | Khi nào áp dụng |
|-------|--------|-----------|-----------------|
| Tool Cache | Same tool called N times | Shared cache với TTL | Always, cheap |
| Finding Dedup | Same issue reported N times | Embedding similarity | After agent completion |
| KB Dedup | Same doc indexed N times | Content hash | During ingestion |
| Task Dedup | Same task assigned N times | Task signature set | Before assignment |

**Key insight**: Dedup không chỉ về tiết kiệm tokens — nó về **chất lượng output**. Một report với 10 unique findings rõ ràng tốt hơn nhiều so với report với 40 findings trong đó 30 là duplicates.

---

*Bài tiếp theo: Lesson 09 — Evaluation Frameworks: Measuring Agent Quality*
