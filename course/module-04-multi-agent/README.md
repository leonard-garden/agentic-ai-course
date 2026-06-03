# Module 04: Multi-Agent Systems

> **Thời lượng**: 4 tuần | **Level**: Advanced | **Prerequisites**: Module 01-03

---

## Tổng quan

Trong module này, bạn sẽ học cách thiết kế và vận hành các hệ thống **multi-agent** — nơi nhiều AI agent phối hợp với nhau để giải quyết những bài toán phức tạp mà một agent đơn lẻ không thể xử lý hiệu quả.

Đây là kỹ năng phân biệt một **AI engineer** thực sự với người chỉ biết gọi API. Trong production, các hệ thống multi-agent đang xử lý code review, research, data pipeline, và deployment automation ở quy mô lớn.

---

## Tại sao Multi-Agent?

Hãy nghĩ về cách một team engineering hoạt động: Tech Lead không tự mình viết toàn bộ code. Tech Lead **phân rã** vấn đề, **giao việc** cho các specialist, **theo dõi tiến độ**, và **tổng hợp kết quả**. Multi-agent AI hoạt động theo đúng nguyên tắc đó.

**4 lý do chính để dùng multi-agent:**

1. **Parallelization** — Các subtask độc lập chạy đồng thời, giảm latency tổng thể lên đến 60-90%
2. **Specialization** — Mỗi agent có system prompt và tool set riêng, tối ưu cho đúng task của nó
3. **Context Isolation** — Mỗi agent có context window sạch, không bị nhiễu bởi context của task khác
4. **Scale** — Một orchestrator có thể điều phối hàng chục worker agents đồng thời

---

## Lessons trong Module này

### Lesson 01: Orchestrator-Worker Pattern
**Tuần 1** — Kiến trúc nền tảng của multi-agent systems

Học cách một **Orchestrator** (thường là Opus) phân rã task, giao việc cho các **Worker agents** (Sonnet/Haiku), theo dõi tiến độ, và tổng hợp kết quả cuối cùng.

**Worked example**: Java Code Review Pipeline — SecurityAuditor, PerformanceAnalyzer, TestCoverageChecker, ReportSynthesizer chạy song song.

📄 [01-orchestrator-worker-pattern.md](./lessons/01-orchestrator-worker-pattern.md)

---

### Lesson 02: Model Routing Strategy
**Tuần 2** — Chọn đúng model cho đúng task

Opus tốn gấp 25x Haiku. Nếu bạn dùng Opus cho mọi thứ, bạn đang đốt tiền. Lesson này dạy bạn **routing logic** để tự động chọn model phù hợp dựa trên task type — và cách tính toán cost savings thực tế.

**Worked example**: ModelRouter utility class trong Java, phân tích cost 1000 API calls/ngày.

📄 [02-model-routing-strategy.md](./lessons/02-model-routing-strategy.md)

---

### Lesson 03: Failure Modes & Production Readiness
**Tuần 3** — Vận hành multi-agent trong production

Non-determinism + emergent behavior + cost unpredictability = production nightmare nếu không chuẩn bị. Lesson này đi qua **7 failure modes** quan trọng nhất và cách phòng chống từng loại.

**Worked example**: Thêm production safeguards vào Java Code Review Pipeline từ Lesson 01.

📄 [03-failure-modes-and-production.md](./lessons/03-failure-modes-and-production.md)

---

### Lesson 04: Evaluator-Optimizer Pattern
**Tuần 4** — Quality feedback loops trong AI systems

Pattern Generator → Evaluator → Generator cho phép AI tự cải thiện output qua nhiều vòng lặp cho đến khi đạt ngưỡng chất lượng. Đặc biệt hữu ích cho code generation, document writing, test case generation.

**Worked example**: JUnit 5 test generation với evaluator kiểm tra coverage và assertion quality.

📄 [04-evaluator-optimizer-pattern.md](./lessons/04-evaluator-optimizer-pattern.md)

---

## Project Cuối Module

### Production Java Code Review Pipeline
Xây dựng một hệ thống review code production-grade với multi-agent architecture, model routing, failure recovery, và observability đầy đủ.

📄 [exercises/project-java-review-pipeline.md](./exercises/project-java-review-pipeline.md)

---

## Learning Path

```
Week 1: Orchestrator-Worker Pattern
  → Hiểu kiến trúc, implement basic pipeline
  → Exercise: Java Code Review với 4 specialized agents

Week 2: Model Routing Strategy  
  → Cost optimization, routing logic
  → Exercise: Tính toán và implement routing cho pipeline từ Week 1

Week 3: Failure Modes & Production
  → 7 failure modes, production checklist
  → Exercise: Thêm safeguards, observability vào pipeline

Week 4: Evaluator-Optimizer Pattern
  → Quality feedback loops
  → Exercise: Thêm evaluator vào code review pipeline
  
Final Project: Production-grade Java Code Review Pipeline
  → Full implementation, deployment, monitoring
```

---

## Tech Stack cho Module này

| Tool | Mục đích |
|------|----------|
| Anthropic SDK (Python) | Agent orchestration |
| Claude API (Java client) | Worker agents trong Java context |
| Claude Code Agent Teams | Experimental multi-agent primitives |
| Prometheus + Grafana | Observability (optional) |

---

## Prerequisites

Trước khi bắt đầu module này, bạn cần:

- [ ] Hoàn thành Module 01 (Claude API basics, tool use)
- [ ] Hoàn thành Module 02 (Claude Code)
- [ ] Hoàn thành Module 03 (MCP)
- [ ] Comfortable với async/await trong Python hoặc CompletableFuture trong Java
- [ ] Hiểu cơ bản về system design (microservices, message queues)

---

## Kết quả học tập

Sau module này, bạn có thể:

- [ ] Thiết kế multi-agent architecture cho complex business problems
- [ ] Implement Orchestrator-Worker pattern với Claude API
- [ ] Tối ưu cost với model routing (Haiku/Sonnet/Opus)
- [ ] Xử lý 7 failure modes phổ biến nhất trong production
- [ ] Build Evaluator-Optimizer loops cho quality assurance
- [ ] Deploy và monitor multi-agent systems trong production

---

## Resources

- [Anthropic Multi-agent docs](https://docs.anthropic.com/en/docs/build-with-claude/agents)
- [Claude Code Agent Teams (experimental)](https://docs.anthropic.com/en/docs/claude-code/agent-teams)
- [AWS reference implementation](https://github.com/aws-samples/sample-claude-code-agent-team)
- [Anthropic Research Feature case study (April 2025)](https://www.anthropic.com/research)
