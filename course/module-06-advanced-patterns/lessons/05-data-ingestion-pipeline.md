# Lesson 05: Data Ingestion Pipeline — Feeding Your Agent's Knowledge Base

> **Module 06 — Advanced Patterns**
> Prerequisite: Lesson 04 (Agent Memory Systems), hiểu về vector embeddings và pgvector
> Thời gian ước tính: 90 phút đọc + 90 phút lab

---

## Mở đầu: Chất lượng Agent = Chất lượng Dữ liệu

Bạn có thể dùng Claude Opus, HNSW index tốt nhất, retrieval pipeline tinh vi nhất — nhưng nếu knowledge base chứa rác, agent sẽ trả lời bằng rác. **Garbage in, garbage out.**

Đây là nguyên tắc cơ bản nhưng thường bị bỏ qua. Engineers tập trung vào LLM prompting và retrieval tuning, nhưng phần thực sự tạo ra sự khác biệt là **data ingestion pipeline**.

Ví dụ thực tế: Một team xây chatbot Q&A cho codebase Spring Boot của họ. Agent trả lời sai về 40% câu hỏi. Sau khi audit, họ phát hiện:
- PDF docs có header/footer lặp lại hàng nghìn lần sau khi chunk
- Java files bị chunk theo fixed size, cắt giữa method signatures
- Nhiều auto-generated files (Lombok, MapStruct) được ingested gây noise

Sau khi fix ingestion pipeline, accuracy tăng lên 78% — không thay đổi gì về LLM hay prompting.

### Pipeline Overview

```
Source → Load → Clean → Chunk → Embed → Index → Serve
  │         │       │       │        │       │       │
Files     Parse   Remove  Split   Vector  Store  Retrieve
URLs      Text    Noise   Smart   ize    DB     + Rank
DB        Extract Normal  Code-   Batch        Re-rank
Code              ize     aware
```

Mỗi stage có thể là bottleneck. Chúng ta sẽ đi qua từng stage.

---

## Stage 1: Load — Đọc dữ liệu từ mọi nguồn

### Nguồn dữ liệu phổ biến trong môi trường enterprise Java

| Source | Format | Loader |
|---|---|---|
| Java source code | `.java` files | Custom + JavaParser |
| Documentation | PDF, Word, HTML | Apache Tika |
| Internal wiki | Confluence pages | Confluence REST API |
| Architecture docs | Markdown | Spring AI MarkdownDocumentReader |
| API specs | OpenAPI YAML/JSON | Custom YAML loader |
| Database | PostgreSQL tables | JDBC ResultSet loader |
| Issue tracker | Jira tickets | Jira REST API |

### Spring AI Document Readers

Spring AI cung cấp built-in readers cho các format phổ biến:

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-tika-document-reader</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-pdf-document-reader</artifactId>
</dependency>
```

```java
@Component
public class DocumentLoaderFactory {

    /**
     * Load PDF file — dùng cho architecture docs, runbooks.
     */
    public List<Document> loadPdf(Resource pdfResource) {
        var config = PdfDocumentReaderConfig.builder()
                .withPageTopMargin(72)       // bỏ header (72pt = 1 inch)
                .withPageBottomMargin(72)    // bỏ footer
                .withPagesPerDocument(1)     // mỗi page = 1 Document
                .build();

        return new PagePdfDocumentReader(pdfResource, config).get();
    }

    /**
     * Load HTML page — dùng cho Confluence, internal wikis.
     */
    public List<Document> loadHtml(Resource htmlResource) {
        return new TikaDocumentReader(htmlResource).get();
    }

    /**
     * Load Markdown — dùng cho README, ADRs, runbooks.
     */
    public List<Document> loadMarkdown(Resource mdResource) {
        return new MarkdownDocumentReader(mdResource,
                MarkdownDocumentReaderConfig.builder()
                        .withHorizontalRuleCreateDocument(true)
                        .withIncludeCodeBlock(true)
                        .build()
        ).get();
    }
}
```

### Java Codebase Loader

Đây là loader quan trọng nhất cho Java backend teams — cho phép agent hiểu codebase của bạn.

```java
@Component
@Slf4j
public class JavaCodebaseLoader {

    private static final Set<String> SKIP_PATTERNS = Set.of(
        "target/", "build/", ".git/", "generated-sources/",
        "generated/", "node_modules/", ".idea/", "*.class"
    );

    /**
     * Load toàn bộ .java files từ một repository path.
     * Tự động skip auto-generated files và build artifacts.
     */
    public List<Document> loadRepository(Path repoPath) throws IOException {
        log.info("Loading Java codebase from: {}", repoPath);

        try (var files = Files.walk(repoPath)) {
            var docs = files
                    .filter(Files::isRegularFile)
                    .filter(path -> path.toString().endsWith(".java"))
                    .filter(path -> !isSkipped(path))
                    .map(path -> loadJavaFile(repoPath, path))
                    .filter(Optional::isPresent)
                    .map(Optional::get)
                    .toList();

            log.info("Loaded {} Java files from {}", docs.size(), repoPath);
            return docs;
        }
    }

    private Optional<Document> loadJavaFile(Path repoRoot, Path filePath) {
        try {
            var content = Files.readString(filePath, StandardCharsets.UTF_8);
            var relativePath = repoRoot.relativize(filePath).toString();

            var metadata = new HashMap<String, Object>();
            metadata.put("source", relativePath);
            metadata.put("type", "java_source");
            metadata.put("fileName", filePath.getFileName().toString());
            metadata.put("package", extractPackage(content));
            metadata.put("lastModified",
                         Files.getLastModifiedTime(filePath).toInstant().toString());

            return Optional.of(new Document(content, metadata));
        } catch (IOException e) {
            log.warn("Failed to load {}: {}", filePath, e.getMessage());
            return Optional.empty();
        }
    }

    private boolean isSkipped(Path path) {
        var pathStr = path.toString().replace('\\', '/');
        return SKIP_PATTERNS.stream().anyMatch(pattern ->
            pathStr.contains(pattern.replace("*", ""))
        );
    }

    private String extractPackage(String javaSource) {
        return javaSource.lines()
                .filter(line -> line.startsWith("package "))
                .findFirst()
                .map(line -> line.replace("package ", "").replace(";", "").trim())
                .orElse("unknown");
    }
}
```

### Confluence Loader (REST API)

```java
@Component
public class ConfluenceLoader {

    private final RestTemplate restTemplate;
    private final String baseUrl;
    private final String token;

    public List<Document> loadSpace(String spaceKey) {
        var url = "%s/rest/api/content?spaceKey=%s&type=page&limit=100"
                  .formatted(baseUrl, spaceKey);

        var response = restTemplate.getForObject(url, ConfluencePageListResponse.class);
        if (response == null || response.results().isEmpty()) return List.of();

        return response.results().stream()
                .map(page -> loadPageContent(page.id()))
                .filter(Optional::isPresent)
                .map(Optional::get)
                .toList();
    }

    private Optional<Document> loadPageContent(String pageId) {
        var url = "%s/rest/api/content/%s?expand=body.storage,version"
                  .formatted(baseUrl, pageId);

        var page = restTemplate.getForObject(url, ConfluencePage.class);
        if (page == null) return Optional.empty();

        // Strip HTML tags từ Confluence storage format
        var plainText = Jsoup.parse(page.body().storage().value()).text();

        var metadata = Map.<String, Object>of(
            "source", "confluence",
            "pageId", page.id(),
            "title", page.title(),
            "type", "confluence_page"
        );

        return Optional.of(new Document(plainText, metadata));
    }
}
```

---

## Stage 2: Clean — Loại bỏ Nhiễu

Dữ liệu thô chứa nhiều noise: header/footer lặp lại trong PDF, HTML boilerplate, khoảng trắng dư thừa, encoding inconsistency. Clean stage loại bỏ tất cả.

```java
@Component
public class DocumentCleaner {

    private static final Pattern REPEATED_WHITESPACE = Pattern.compile("\\s{3,}");
    private static final Pattern PAGE_NUMBERS = Pattern.compile("\\bPage\\s+\\d+\\s+of\\s+\\d+\\b");
    private static final Pattern CONFIDENTIAL_HEADER = 
        Pattern.compile("(?i)CONFIDENTIAL|INTERNAL USE ONLY|PROPRIETARY");

    /**
     * Clean một document — remove noise, normalize whitespace, filter boilerplate.
     */
    public Optional<Document> clean(Document doc) {
        var content = doc.getContent();

        // 1. Normalize encoding
        content = new String(content.getBytes(StandardCharsets.UTF_8),
                             StandardCharsets.UTF_8);

        // 2. Remove page numbers
        content = PAGE_NUMBERS.matcher(content).replaceAll("");

        // 3. Remove confidentiality headers (không có thông tin hữu ích)
        content = CONFIDENTIAL_HEADER.matcher(content).replaceAll("");

        // 4. Normalize whitespace
        content = REPEATED_WHITESPACE.matcher(content).replaceAll("\n\n");
        content = content.strip();

        // 5. Filter: skip nếu content quá ngắn sau khi clean
        if (content.length() < 100) {
            return Optional.empty();   // không đủ nội dung để embed
        }

        // 6. Filter: skip auto-generated Java files
        if (isAutoGenerated(doc, content)) {
            return Optional.empty();
        }

        return Optional.of(new Document(content, doc.getMetadata()));
    }

    private boolean isAutoGenerated(Document doc, String content) {
        // Detect Lombok, MapStruct, JPA metamodel generated files
        var source = (String) doc.getMetadata().getOrDefault("source", "");
        if (source.contains("generated") || source.contains("mappers/")) return true;

        // Detect generated comment patterns
        return content.contains("@Generated") ||
               content.contains("DO NOT EDIT") ||
               content.contains("This file was automatically generated");
    }

    public List<Document> cleanAll(List<Document> docs) {
        return docs.stream()
                .map(this::clean)
                .filter(Optional::isPresent)
                .map(Optional::get)
                .toList();
    }
}
```

---

## Stage 3: Chunk — Bước Quan trọng nhất

**Chunking có tác động lớn nhất đến retrieval quality.** Chunk sai = retrieval sai = agent sai.

### Vì sao chunking quan trọng?

Vector store lưu và retrieve theo **chunk**, không theo toàn bộ document. Khi agent query "how is retry logic implemented?", vector store tìm chunk nào giống query nhất.

Nếu chunk cắt giữa một method:
```java
// Chunk 1:
public void processPayment(Payment payment) {
    try {
        // ... 500 lines of logic ...

// Chunk 2 (tiếp theo):
    } catch (PaymentException e) {
        retryService.schedule(payment);
    }
}
```

Chunk 2 chứa retry logic nhưng thiếu context — LLM không biết method này là gì.

### 4 Chiến lược Chunking

#### Chiến lược 1: Fixed Size (tệ — chỉ dùng khi không còn lựa chọn)

```java
public List<Chunk> fixedSize(String text, int chunkSize, int overlap) {
    var chunks = new ArrayList<Chunk>();
    var tokens = tokenize(text);   // tính bằng token, không phải character

    for (int i = 0; i < tokens.size(); i += (chunkSize - overlap)) {
        int end = Math.min(i + chunkSize, tokens.size());
        chunks.add(new Chunk(detokenize(tokens.subList(i, end))));
    }
    return chunks;
}
```

Vấn đề: cắt giữa câu, giữa method, giữa concept. Kết quả retrieval kém.

#### Chiến lược 2: Recursive Character Chunking (tốt hơn — dùng cho prose text)

Split theo ưu tiên: `\n\n` → `\n` → `. ` → ` ` → character. Cố gắng giữ đoạn văn nguyên vẹn.

```java
@Component
public class RecursiveCharacterChunker {

    private static final List<String> SEPARATORS = List.of(
        "\n\n",    // paragraph break — ưu tiên cao nhất
        "\n",      // line break
        ". ",      // sentence end
        " ",       // word boundary
        ""         // character (fallback)
    );

    public List<String> chunk(String text, int maxChunkSize, int overlap) {
        return splitRecursively(text, SEPARATORS, maxChunkSize, overlap);
    }

    private List<String> splitRecursively(String text,
                                           List<String> separators,
                                           int maxSize, int overlap) {
        if (text.length() <= maxSize) return List.of(text);
        if (separators.isEmpty()) {
            // Fallback: hard split
            return List.of(text.substring(0, maxSize));
        }

        var separator = separators.get(0);
        var remainingSeps = separators.subList(1, separators.size());
        var parts = text.split(Pattern.quote(separator), -1);

        var result = new ArrayList<String>();
        var current = new StringBuilder();

        for (var part : parts) {
            if (current.length() + part.length() + separator.length() > maxSize) {
                if (!current.isEmpty()) {
                    result.add(current.toString());
                    // Overlap: giữ lại N ký tự cuối để maintain context
                    var overlapText = current.substring(
                        Math.max(0, current.length() - overlap)
                    );
                    current = new StringBuilder(overlapText);
                }
            }
            current.append(part).append(separator);
        }

        if (!current.isEmpty()) result.add(current.toString());
        return result;
    }
}
```

#### Chiến lược 3: Semantic Chunking (tốt nhất cho text — nhưng tốn kém)

Embed từng câu, split khi semantic similarity giữa các câu liên tiếp giảm đột ngột — tức là chuyển sang topic mới.

```java
@Component
public class SemanticChunker {

    private final EmbeddingClient embeddingClient;
    private static final double SIMILARITY_THRESHOLD = 0.85;

    public List<String> chunk(String text) {
        // 1. Split thành sentences
        var sentences = Arrays.asList(text.split("(?<=[.!?])\\s+"));
        if (sentences.size() <= 2) return List.of(text);

        // 2. Embed all sentences (batch call)
        var embeddings = embeddingClient.embed(sentences);

        // 3. Find split points: where similarity drops
        var splitPoints = new ArrayList<Integer>();
        for (int i = 1; i < sentences.size(); i++) {
            var sim = cosineSimilarity(embeddings.get(i - 1), embeddings.get(i));
            if (sim < SIMILARITY_THRESHOLD) {
                splitPoints.add(i);
            }
        }

        // 4. Build chunks from split points
        var chunks = new ArrayList<String>();
        int start = 0;
        for (int splitAt : splitPoints) {
            chunks.add(String.join(" ", sentences.subList(start, splitAt)));
            start = splitAt;
        }
        chunks.add(String.join(" ", sentences.subList(start, sentences.size())));

        return chunks;
    }

    private double cosineSimilarity(List<Double> a, List<Double> b) {
        double dot = 0, normA = 0, normB = 0;
        for (int i = 0; i < a.size(); i++) {
            dot += a.get(i) * b.get(i);
            normA += a.get(i) * a.get(i);
            normB += b.get(i) * b.get(i);
        }
        return dot / (Math.sqrt(normA) * Math.sqrt(normB));
    }
}
```

**Lưu ý:** Semantic chunking tốn N embedding calls (N = số sentences). Với codebase lớn, tốn cả tiền lẫn thời gian. Dùng cho documentation, không dùng cho code.

#### Chiến lược 4: Code-Aware Chunking (tốt nhất cho Java code)

Split Java files theo **class và method boundaries** — dùng JavaParser để parse AST, không phải string splitting.

```java
@Component
@Slf4j
public class JavaAwareChunker {

    /**
     * Chunk một Java source file theo class/method boundaries.
     * Mỗi method = 1 chunk với full context (class name, method signature, javadoc).
     */
    public List<Chunk> chunk(String javaSource, Map<String, Object> fileMetadata) {
        CompilationUnit cu;
        try {
            cu = StaticJavaParser.parse(javaSource);
        } catch (ParseProblemException e) {
            log.warn("Failed to parse Java source, falling back to recursive chunking");
            return fallbackChunk(javaSource, fileMetadata);
        }

        var chunks = new ArrayList<Chunk>();

        // Chunk 1: Class-level (imports + class declaration + fields)
        var classChunk = buildClassChunk(cu, fileMetadata);
        chunks.add(classChunk);

        // Chunk per method
        cu.findAll(MethodDeclaration.class).forEach(method -> {
            var chunk = buildMethodChunk(method, cu, fileMetadata);
            chunks.add(chunk);
        });

        // Chunk per constructor
        cu.findAll(ConstructorDeclaration.class).forEach(constructor -> {
            var chunk = buildConstructorChunk(constructor, cu, fileMetadata);
            chunks.add(chunk);
        });

        return chunks;
    }

    private Chunk buildMethodChunk(MethodDeclaration method,
                                    CompilationUnit cu,
                                    Map<String, Object> fileMetadata) {
        // Build enriched content: include context that helps retrieval
        var content = new StringBuilder();

        // Class context
        method.findAncestor(ClassOrInterfaceDeclaration.class).ifPresent(cls -> {
            content.append("// Class: ").append(cls.getNameAsString()).append("\n");
        });

        // Javadoc (crucial for understanding intent)
        method.getJavadoc().ifPresent(javadoc ->
            content.append(javadoc.toComment().getCommentContent()).append("\n")
        );

        // Annotations
        method.getAnnotations().forEach(ann ->
            content.append("@").append(ann.getNameAsString()).append("\n")
        );

        // Full method source
        content.append(method.toString());

        var metadata = new HashMap<>(fileMetadata);
        metadata.put("chunkType", "method");
        metadata.put("methodName", method.getNameAsString());
        metadata.put("returnType", method.getTypeAsString());
        metadata.put("visibility", method.getAccessSpecifier().asString());

        return new Chunk(content.toString(), metadata);
    }

    private Chunk buildClassChunk(CompilationUnit cu,
                                   Map<String, Object> fileMetadata) {
        // Class-level chunk: imports, class declaration, fields (no method bodies)
        var content = new StringBuilder();

        cu.getPackageDeclaration().ifPresent(pkg ->
            content.append(pkg.toString()).append("\n")
        );

        cu.getImports().forEach(imp -> content.append(imp.toString()));
        content.append("\n");

        cu.findAll(ClassOrInterfaceDeclaration.class).stream()
                .findFirst()
                .ifPresent(cls -> {
                    content.append(cls.getJavadoc()
                            .map(j -> j.toComment().getCommentContent())
                            .orElse("")).append("\n");
                    content.append("public class ").append(cls.getNameAsString());
                    if (!cls.getImplementedTypes().isEmpty()) {
                        content.append(" implements ");
                        cls.getImplementedTypes().forEach(t ->
                            content.append(t.getNameAsString()).append(", ")
                        );
                    }
                    content.append(" {\n");
                    cls.getFields().forEach(f -> content.append("    ").append(f).append("\n"));
                    content.append("}");
                });

        var metadata = new HashMap<>(fileMetadata);
        metadata.put("chunkType", "class_overview");

        return new Chunk(content.toString(), metadata);
    }

    private List<Chunk> fallbackChunk(String source,
                                       Map<String, Object> fileMetadata) {
        // Fallback sang recursive chunking nếu parse thất bại
        return List.of(new Chunk(source, fileMetadata));
    }
}

public record Chunk(String content, Map<String, Object> metadata) {}
```

### Chunk Size Guidelines

```
┌──────────────┬────────────────────────────────────────────┐
│ Chunk size   │ Vấn đề                                      │
├──────────────┼────────────────────────────────────────────┤
│ < 100 tokens │ Thiếu context, cosine similarity kém        │
│ 100-256 tok  │ Tốt cho Q&A ngắn, FAQ                      │
│ 256-512 tok  │ Sweet spot cho documentation (khuyến nghị)  │
│ 512-1000 tok │ OK cho code (methods thường 50-200 lines)   │
│ > 1000 tok   │ Quá lớn, dilutes relevance, retrieval kém   │
└──────────────┴────────────────────────────────────────────┘
Overlap: 10-20% của chunk size (giữ context across boundary)
```

---

## Stage 4: Embed — Chuyển Text thành Vector

### Embedding Models so sánh

| Model | Provider | Dimensions | Tốt cho | Cost |
|---|---|---|---|---|
| `text-embedding-3-small` | OpenAI | 1536 | General text | $ |
| `text-embedding-3-large` | OpenAI | 3072 | High accuracy | $$ |
| `voyage-code-2` | Voyage AI | 1536 | **Source code** | $$ |
| `voyage-3` | Voyage AI | 1024 | General text | $$ |
| `nomic-embed-text` | Nomic (local) | 768 | Local/offline | Free |

**Khuyến nghị cho Java codebase:** `voyage-code-2` được train đặc biệt cho code — hiểu Java syntax, method names, package structures tốt hơn đáng kể so với general models.

### Batch Embedding — Đừng Embed Từng Cái Một

```java
@Component
@Slf4j
public class BatchEmbeddingService {

    private final EmbeddingClient embeddingClient;
    private static final int BATCH_SIZE = 100;   // OpenAI limit: 2048 inputs/call

    /**
     * Embed nhiều documents theo batch — tiết kiệm API calls và thời gian.
     * Embedding riêng lẻ: 10,000 docs × 1 API call = 10,000 calls
     * Batch embedding: 10,000 docs / 100 per batch = 100 calls
     */
    public List<float[]> embedBatch(List<String> texts) {
        var result = new ArrayList<float[]>();

        var batches = partition(texts, BATCH_SIZE);
        log.info("Embedding {} texts in {} batches", texts.size(), batches.size());

        for (var batch : batches) {
            var embeddings = embeddingClient.embed(batch);
            embeddings.stream()
                    .map(e -> e.stream().mapToDouble(Double::doubleValue).toArray())
                    .map(this::toFloatArray)
                    .forEach(result::add);
        }

        return result;
    }

    private <T> List<List<T>> partition(List<T> list, int size) {
        var result = new ArrayList<List<T>>();
        for (int i = 0; i < list.size(); i += size) {
            result.add(list.subList(i, Math.min(i + size, list.size())));
        }
        return result;
    }

    private float[] toFloatArray(double[] doubles) {
        var floats = new float[doubles.length];
        for (int i = 0; i < doubles.length; i++) floats[i] = (float) doubles[i];
        return floats;
    }
}
```

---

## Stage 5: Index — Lưu vào Vector Store

### pgvector Index Types

```sql
-- IVFFlat: nhanh hơn nhưng kém chính xác hơn
-- Dùng khi: > 1M vectors, latency quan trọng hơn recall
CREATE INDEX ON vector_store USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);

-- HNSW: cân bằng tốt giữa speed và accuracy
-- Dùng khi: < 1M vectors, cần recall tốt (khuyến nghị cho hầu hết cases)
CREATE INDEX ON vector_store USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

Luôn lưu **metadata** phong phú để enable filtering:

```sql
-- Schema với metadata đầy đủ
CREATE TABLE vector_store (
    id           UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    content      TEXT NOT NULL,
    metadata     JSONB NOT NULL DEFAULT '{}',
    embedding    VECTOR(1536),
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    source_hash  VARCHAR(64)   -- để dedup
);

-- Partial index cho từng doc type (tăng tốc filtered queries)
CREATE INDEX ON vector_store ((metadata->>'type'))
    WHERE metadata->>'type' IS NOT NULL;

CREATE INDEX ON vector_store ((metadata->>'source'));
```

---

## Stage 6: Serve — Retrieval Pipeline

### Similarity Search cơ bản với Spring AI

```java
@Service
public class KnowledgeBaseRetriever {

    private final VectorStore vectorStore;

    /**
     * Basic semantic search.
     */
    public List<Document> search(String query, int topK) {
        return vectorStore.similaritySearch(
            SearchRequest.query(query)
                .withTopK(topK)
                .withSimilarityThreshold(0.7)
        );
    }

    /**
     * Filtered search — chỉ trong một loại document cụ thể.
     */
    public List<Document> searchByType(String query, String docType, int topK) {
        return vectorStore.similaritySearch(
            SearchRequest.query(query)
                .withTopK(topK)
                .withFilterExpression("type == '" + docType + "'")
        );
    }

    /**
     * Hybrid search: kết hợp semantic + keyword.
     * Tốt hơn pure semantic khi query chứa exact terms (method names, class names).
     */
    public List<Document> hybridSearch(String query, int topK) {
        // 1. Semantic search (broad recall)
        var semanticResults = search(query, topK * 2);

        // 2. Keyword search (exact term matching) — dùng PostgreSQL full-text search
        var keywordResults = keywordSearch(query, topK);

        // 3. Merge và deduplicate, giữ rank từ cả hai
        return mergeAndRerank(semanticResults, keywordResults, topK);
    }

    private List<Document> keywordSearch(String query, int topK) {
        // Dùng trực tiếp PostgreSQL FTS — nhanh, built-in
        return jdbcTemplate.query("""
            SELECT content, metadata::text
            FROM vector_store
            WHERE to_tsvector('english', content) @@ plainto_tsquery('english', ?)
            ORDER BY ts_rank(to_tsvector('english', content),
                             plainto_tsquery('english', ?)) DESC
            LIMIT ?
            """,
            (rs, row) -> new Document(rs.getString("content"),
                                       parseMetadata(rs.getString("metadata"))),
            query, query, topK
        );
    }

    private List<Document> mergeAndRerank(List<Document> semantic,
                                           List<Document> keyword,
                                           int topK) {
        // Reciprocal Rank Fusion — combine ranks from two lists
        var scores = new HashMap<String, Double>();

        IntStream.range(0, semantic.size()).forEach(i ->
            scores.merge(semantic.get(i).getId(),
                         1.0 / (60 + i + 1), Double::sum)
        );
        IntStream.range(0, keyword.size()).forEach(i ->
            scores.merge(keyword.get(i).getId(),
                         1.0 / (60 + i + 1), Double::sum)
        );

        var allDocs = Stream.concat(semantic.stream(), keyword.stream())
                .collect(Collectors.toMap(Document::getId, d -> d, (a, b) -> a));

        return scores.entrySet().stream()
                .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
                .limit(topK)
                .map(e -> allDocs.get(e.getKey()))
                .filter(Objects::nonNull)
                .toList();
    }
}
```

---

## Deduplication: Đừng Ingest Cùng Content 2 Lần

Deduplication quan trọng vì:
1. **Storage waste:** same content embedded nhiều lần tốn disk space
2. **Retrieval noise:** duplicate chunks làm lộn xộn results
3. **Cost waste:** re-embedding unchanged content tốn tiền API

### 3 Cấp độ Deduplication

```java
@Service
@Slf4j
public class DeduplicatingIngester {

    private final VectorStore vectorStore;
    private final BatchEmbeddingService embeddingService;
    private final JdbcTemplate jdbc;

    /**
     * Ingest với đầy đủ deduplication checks.
     */
    public IngestionResult ingest(List<Document> docs) {
        int skippedExact = 0, skippedNearDup = 0, ingested = 0;

        for (var doc : docs) {
            // Level 1: Exact hash check (O(1), rất nhanh)
            if (isExactDuplicate(doc)) {
                skippedExact++;
                continue;
            }

            // Level 2: Near-duplicate check (cần embedding + vector search)
            if (isNearDuplicate(doc)) {
                skippedNearDup++;
                continue;
            }

            // Level 3: Ingest và lưu hash
            embedAndStore(doc);
            ingested++;
        }

        log.info("Ingestion complete: {} ingested, {} exact dups, {} near-dups skipped",
                 ingested, skippedExact, skippedNearDup);

        return new IngestionResult(ingested, skippedExact, skippedNearDup);
    }

    /**
     * Level 1: SHA-256 hash của content. Nếu đã tồn tại → skip.
     */
    private boolean isExactDuplicate(Document doc) {
        var hash = sha256(doc.getContent());
        var count = jdbc.queryForObject(
            "SELECT COUNT(*) FROM vector_store WHERE source_hash = ?",
            Integer.class, hash
        );
        return count != null && count > 0;
    }

    /**
     * Level 2: Near-duplicate bằng cosine similarity > 0.98.
     * Catch được paraphrased content, minor edits.
     */
    private boolean isNearDuplicate(Document doc) {
        var similar = vectorStore.similaritySearch(
            SearchRequest.query(doc.getContent())
                .withTopK(1)
                .withSimilarityThreshold(0.98)
        );
        return !similar.isEmpty();
    }

    private void embedAndStore(Document doc) {
        var hash = sha256(doc.getContent());
        doc.getMetadata().put("sourceHash", hash);
        vectorStore.add(List.of(doc));
    }

    private String sha256(String input) {
        try {
            var digest = MessageDigest.getInstance("SHA-256");
            return HexFormat.of().formatHex(
                digest.digest(input.getBytes(StandardCharsets.UTF_8))
            );
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException(e);
        }
    }
}

public record IngestionResult(int ingested, int exactDuplicates, int nearDuplicates) {}
```

### Incremental Ingestion: Chỉ Re-ingest Thay đổi

```java
@Component
public class IncrementalIngestionTracker {

    private final JdbcTemplate jdbc;

    /**
     * Lưu "fingerprint" của file sau khi ingest.
     * Lần sau, so sánh fingerprint để quyết định có cần re-ingest không.
     */
    public void markIngested(String sourceId, String contentHash,
                              Instant lastModified) {
        jdbc.update("""
            INSERT INTO ingestion_tracker (source_id, content_hash, last_modified, ingested_at)
            VALUES (?, ?, ?, NOW())
            ON CONFLICT (source_id)
            DO UPDATE SET content_hash = ?, last_modified = ?, ingested_at = NOW()
            """,
            sourceId, contentHash, lastModified,
            contentHash, lastModified
        );
    }

    /**
     * True nếu file chưa thay đổi kể từ lần ingest cuối.
     */
    public boolean isUpToDate(String sourceId, String currentHash) {
        var storedHash = jdbc.queryForObject(
            "SELECT content_hash FROM ingestion_tracker WHERE source_id = ?",
            String.class, sourceId
        );
        return currentHash.equals(storedHash);
    }

    /**
     * Sử dụng: chỉ ingest files đã thay đổi.
     */
    public List<Document> filterChanged(List<Document> docs) {
        return docs.stream()
                .filter(doc -> {
                    var sourceId = (String) doc.getMetadata().get("source");
                    var hash = sha256(doc.getContent());
                    return !isUpToDate(sourceId, hash);
                })
                .toList();
    }
}
```

---

## Putting It All Together: Complete Ingestion Pipeline

```java
@Service
@Slf4j
public class CodebaseIngestionPipeline {

    private final JavaCodebaseLoader loader;
    private final DocumentCleaner cleaner;
    private final JavaAwareChunker chunker;
    private final DeduplicatingIngester ingester;
    private final IncrementalIngestionTracker tracker;

    /**
     * Full pipeline: Load → Clean → Chunk → Deduplicate → Embed → Index
     */
    public PipelineResult ingestCodebase(Path repoPath) throws IOException {
        log.info("Starting codebase ingestion: {}", repoPath);
        var startTime = Instant.now();

        // Stage 1: Load
        var rawDocs = loader.loadRepository(repoPath);
        log.info("Loaded {} Java files", rawDocs.size());

        // Stage 1b: Incremental — skip unchanged files
        var changedDocs = tracker.filterChanged(rawDocs);
        log.info("{} files changed since last ingestion", changedDocs.size());

        // Stage 2: Clean
        var cleanedDocs = cleaner.cleanAll(changedDocs);
        log.info("After cleaning: {} documents", cleanedDocs.size());

        // Stage 3: Chunk (code-aware)
        var chunks = cleanedDocs.stream()
                .flatMap(doc -> chunker.chunk(doc.getContent(), doc.getMetadata()).stream())
                .map(chunk -> new Document(chunk.content(), chunk.metadata()))
                .toList();
        log.info("After chunking: {} chunks", chunks.size());

        // Stage 4+5+6: Deduplicate + Embed + Index
        var result = ingester.ingest(chunks);

        // Track completion
        changedDocs.forEach(doc -> {
            var sourceId = (String) doc.getMetadata().get("source");
            tracker.markIngested(sourceId, sha256(doc.getContent()),
                                  Instant.now());
        });

        var duration = Duration.between(startTime, Instant.now());
        log.info("Ingestion complete in {}s: {}", duration.getSeconds(), result);

        return new PipelineResult(
            rawDocs.size(), changedDocs.size(), chunks.size(),
            result.ingested(), result.exactDuplicates(),
            duration
        );
    }
}

public record PipelineResult(
    int totalFiles,
    int changedFiles,
    int totalChunks,
    int ingestedChunks,
    int skippedDuplicates,
    Duration duration
) {}
```

---

## Lab Exercise: Java Codebase Q&A Agent

Xây dựng agent có thể trả lời câu hỏi về codebase Spring Boot của bạn.

### Yêu cầu

1. **Ingestion Pipeline:**
   - Load tất cả `.java` files từ một Spring Boot project
   - Java-aware chunking (JavaParser)
   - Deduplication với content hash
   - Incremental tracking

2. **Q&A Agent:**
   - User hỏi: "Where is payment retry logic implemented?"
   - Agent query vector store với hybrid search
   - Inject top-5 chunks vào context
   - Claude trả lời với code references cụ thể

3. **Test với các queries:**
   - "How is authentication handled?"
   - "What happens when a payment fails?"
   - "Show me the database transaction handling"
   - "Which services use the EventPublisher?"

4. **Đo accuracy:**
   - So sánh agent answers với actual codebase
   - Mục tiêu: > 75% accurate responses

### Checklist

- [ ] JavaCodebaseLoader hoạt động (load + skip generated files)
- [ ] DocumentCleaner loại bỏ boilerplate
- [ ] JavaAwareChunker split đúng theo class/method
- [ ] Content hash deduplication hoạt động
- [ ] Incremental tracking: re-run pipeline, chỉ ingest changed files
- [ ] Hybrid search (semantic + keyword) trả về kết quả tốt hơn pure semantic
- [ ] Agent trả lời câu hỏi với code references chính xác

---

## Tóm tắt

| Stage | Công cụ Java | Pitfall phổ biến |
|---|---|---|
| Load | Spring AI Readers, Apache Tika | Bỏ sót sources, encoding issues |
| Clean | Custom DocumentCleaner | Không filter auto-generated code |
| **Chunk** | **JavaParser (code-aware)** | **Fixed-size chunking = root cause của nhiều bugs** |
| Embed | Spring AI EmbeddingClient | Embed 1-by-1 thay vì batch |
| Index | pgvector + HNSW | Thiếu metadata → không filter được |
| Serve | Hybrid search | Pure semantic bỏ qua exact term matches |

**Lesson tiếp theo:** Lesson 06 — Context Management: khi agent đã có memory và knowledge base, làm thế nào để manage token budget hiệu quả trong các agent runs dài.

---

> **Ghi chú production:** Pipeline này nên chạy trong background job (Spring `@Scheduled` hoặc message queue). Re-ingestion toàn bộ codebase mỗi khi có commit là không thực tế. Kết hợp incremental tracking với Git webhook để trigger ingestion chỉ khi có file changes.
