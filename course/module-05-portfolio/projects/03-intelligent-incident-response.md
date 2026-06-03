# Project 03: Intelligent Incident Response Agent

> **Tên project:** IntelliOps — AI-Powered Incident Response Assistant
> **Tagline:** Diagnose production incidents with AI, act only with human approval
> **Độ khó:** Advanced
> **Thời gian build:** 4-5 ngày
> **Stack:** Spring Boot (webhook receiver + orchestrator), Claude API (tool use), PagerDuty, Slack, Datadog
> **Portfolio value:** ⭐⭐⭐⭐⭐ — Production maturity signal

---

## Tại sao đây là portfolio piece đặc biệt?

Hầu hết AI engineering portfolios demo những thứ **safe** — chatbots, doc generators, code helpers. IntelliOps demo một thứ khác hoàn toàn: **AI trong high-stakes environment với human oversight design**.

**1. Demonstrates production AI maturity**
Khi build IntelliOps, bạn phải nghĩ về: "What if Claude diagnoses wrong? What if it suggests a fix that makes things worse?" Những câu hỏi này — và cách bạn answer chúng — là dấu hiệu của một AI engineer mature.

**2. Human-in-the-loop design**
Đây là pattern mà enterprise companies *require* cho AI systems touching production. Biết design và implement HITL đúng cách là differentiated skill.

**3. Real business value với measurable metric**
MTTR (Mean Time to Resolution) là metric mà mọi engineering leader đều care. "IntelliOps giảm MTTR 40%" — câu đó có sức nặng trong mọi interview.

**4. Integration breadth**
PagerDuty + Datadog + Slack + Claude — integrating multiple enterprise tools là kỹ năng thực tế mà companies need.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      PagerDuty                                  │
│           Incident triggered (alert fires)                      │
└────────────────────────┬────────────────────────────────────────┘
                         │ webhook POST
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│           IntelliOps Spring Boot Service                        │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │               WebhookController                         │   │
│  │  POST /webhooks/pagerduty                               │   │
│  └───────────────────────┬─────────────────────────────────┘   │
│                          │                                      │
│  ┌───────────────────────▼─────────────────────────────────┐   │
│  │              IncidentOrchestrator                        │   │
│  │                                                         │   │
│  │  1. Extract incident context from PagerDuty payload     │   │
│  │  2. Create Claude agent session                         │   │
│  │  3. Agent calls tools to gather context                 │   │
│  │  4. Generate diagnosis + remediation suggestions        │   │
│  │  5. Send to Slack with Approve/Reject buttons           │   │
│  │  6. Wait for human approval (timeout: 10 min)           │   │
│  │  7. If approved: execute remediation                    │   │
│  │  8. Log outcome for learning                            │   │
│  └───────────────────────┬─────────────────────────────────┘   │
│                          │                                      │
│         ┌────────────────┼────────────────┐                    │
│         ▼                ▼                ▼                     │
│  ┌──────────┐   ┌──────────────┐   ┌──────────┐               │
│  │read_logs │   │query_metrics │   │search_   │               │
│  │(Datadog) │   │(Datadog API) │   │runbooks  │               │
│  └──────────┘   └──────────────┘   └──────────┘               │
└─────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Slack                                    │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  🚨 IntelliOps Incident Report                           │  │
│  │                                                          │  │
│  │  Incident: HIGH — API latency spike (P95 > 5000ms)      │  │
│  │  Service: order-service | Started: 10:23 UTC            │  │
│  │                                                          │  │
│  │  Diagnosis: OOM condition in OrderService. GC pressure  │  │
│  │  visible in metrics. Root cause: memory leak in         │  │
│  │  getOrderHistory() — loading full order history without │  │
│  │  pagination.                                            │  │
│  │                                                          │  │
│  │  Suggested Remediation:                                  │  │
│  │  1. Restart order-service pod (immediate relief)        │  │
│  │  2. Scale replicas from 2 → 4 (prevent recurrence)     │  │
│  │                                                          │  │
│  │  [✅ Approve Restart]  [🔄 Approve Scale]  [❌ Decline] │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                         │ human approves
                         ▼
                  Execute remediation
                  (kubectl restart / scale)
```

---

## Maven Dependencies

```xml
<dependencies>
    <!-- Spring Boot core -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Async support -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>

    <!-- Persistence (incident state, feedback logs) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>

    <!-- Claude API via HTTP (no official Java SDK yet, use HTTP client) -->
    <dependency>
        <groupId>com.squareup.okhttp3</groupId>
        <artifactId>okhttp</artifactId>
        <version>4.12.0</version>
    </dependency>

    <!-- Slack SDK -->
    <dependency>
        <groupId>com.slack.api</groupId>
        <artifactId>slack-api-client</artifactId>
        <version>1.38.0</version>
    </dependency>

    <!-- JSON -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
</dependencies>
```

---

## Core Domain Model

```java
// domain/Incident.java
@Entity
@Table(name = "incidents")
public class Incident {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    private String pagerdutyId;
    private String title;
    private String severity;  // P1, P2, P3, P4
    private String service;
    private Instant triggeredAt;

    @Enumerated(EnumType.STRING)
    private IncidentStatus status;  // ANALYZING, AWAITING_APPROVAL, REMEDIATING, RESOLVED, DECLINED

    @Column(columnDefinition = "TEXT")
    private String diagnosis;       // Claude's analysis

    @Column(columnDefinition = "TEXT")
    private String remediationPlan; // Claude's suggested actions

    private Boolean diagnosisAccurate;  // Human feedback after resolution
    private String engineerNotes;       // Free-text feedback

    private Instant resolvedAt;
    private Long mttrSeconds;           // Actual time to resolution

    // Audit fields
    private String approvedBy;
    private Instant approvedAt;
}

public enum IncidentStatus {
    ANALYZING,
    AWAITING_APPROVAL,
    REMEDIATING,
    RESOLVED,
    DECLINED,
    FAILED
}
```

---

## PagerDuty Webhook Controller

```java
// controller/WebhookController.java
@RestController
@RequestMapping("/webhooks")
@Slf4j
public class WebhookController {

    private final IncidentOrchestrator orchestrator;
    private final String pagerdutySecret;

    public WebhookController(IncidentOrchestrator orchestrator,
                              @Value("${intelliops.pagerduty.webhook-secret}") String secret) {
        this.orchestrator = orchestrator;
        this.pagerdutySecret = secret;
    }

    @PostMapping("/pagerduty")
    public ResponseEntity<String> handlePagerDutyWebhook(
            @RequestBody String payload,
            @RequestHeader("X-PagerDuty-Signature") String signature) {

        // Verify webhook signature
        if (!verifySignature(payload, signature)) {
            log.warn("Invalid PagerDuty webhook signature");
            return ResponseEntity.status(401).body("Invalid signature");
        }

        try {
            var event = parseWebhookPayload(payload);

            // Only process high-severity triggers
            if (shouldProcess(event)) {
                // Async — don't block webhook response
                CompletableFuture.runAsync(() ->
                    orchestrator.processIncident(event)
                );
            }

            return ResponseEntity.ok("Accepted");
        } catch (Exception e) {
            log.error("Error processing PagerDuty webhook", e);
            return ResponseEntity.status(500).body("Error");
        }
    }

    @PostMapping("/slack/actions")
    public ResponseEntity<String> handleSlackAction(
            @RequestBody String payload) {
        // Slack sends URL-encoded payload
        var action = parseSlackAction(payload);
        orchestrator.handleHumanDecision(action.incidentId(), action.action(), action.userId());
        return ResponseEntity.ok("");
    }

    private boolean shouldProcess(PagerDutyEvent event) {
        // Process P1 and P2 incidents only
        return List.of("P1", "P2").contains(event.severity())
               && "incident.triggered".equals(event.eventType());
    }

    private boolean verifySignature(String payload, String signature) {
        // HMAC-SHA256 verification
        try {
            Mac mac = Mac.getInstance("HmacSHA256");
            mac.init(new SecretKeySpec(pagerdutySecret.getBytes(), "HmacSHA256"));
            String expected = "v1=" + HexFormat.of().formatHex(mac.doFinal(payload.getBytes()));
            return MessageDigest.isEqual(expected.getBytes(), signature.getBytes());
        } catch (Exception e) {
            return false;
        }
    }
}
```

---

## Claude Agent Tools

```java
// agent/IncidentTools.java
@Component
@Slf4j
public class IncidentTools {

    private final DatadogClient datadogClient;
    private final RunbookRepository runbookRepo;

    // Tool 1: Read logs from Datadog
    public String readLogs(String service, String timeRangeMinutes, String errorKeyword) {
        try {
            int minutes = Integer.parseInt(timeRangeMinutes);
            Instant from = Instant.now().minusSeconds(minutes * 60L);

            var query = String.format("service:%s status:error %s", service,
                errorKeyword != null ? errorKeyword : "");

            var logs = datadogClient.queryLogs(query, from, Instant.now(), 100);

            if (logs.isEmpty()) {
                return String.format("No error logs found for service '%s' in last %d minutes", service, minutes);
            }

            var result = new StringBuilder();
            result.append(String.format("=== Logs: %s (last %d min) ===\n\n", service, minutes));
            result.append(String.format("Found %d log entries\n\n", logs.size()));

            // Group by error message
            var grouped = logs.stream()
                .collect(Collectors.groupingBy(
                    log -> extractErrorKey(log.message()),
                    Collectors.counting()
                ));

            result.append("Error frequency:\n");
            grouped.entrySet().stream()
                .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
                .limit(10)
                .forEach(e -> result.append(String.format("  %3d× %s\n", e.getValue(), e.getKey())));

            result.append("\nRecent entries:\n");
            logs.stream().limit(20).forEach(l ->
                result.append(String.format("[%s] %s\n", l.timestamp(), l.message()))
            );

            return result.toString();
        } catch (Exception e) {
            return "Error reading logs: " + e.getMessage();
        }
    }

    // Tool 2: Query metrics from Datadog
    public String queryMetrics(String metricName, String service, String timeRangeMinutes) {
        try {
            int minutes = Integer.parseInt(timeRangeMinutes);
            var metrics = datadogClient.queryMetric(metricName, service,
                Instant.now().minusSeconds(minutes * 60L), Instant.now());

            var result = new StringBuilder();
            result.append(String.format("=== Metric: %s (%s, last %d min) ===\n\n",
                metricName, service, minutes));

            if (metrics.isEmpty()) {
                return result.append("No data found").toString();
            }

            // Calculate stats
            DoubleSummaryStatistics stats = metrics.stream()
                .mapToDouble(DataPoint::value)
                .summaryStatistics();

            result.append(String.format("Min:  %.2f\n", stats.getMin()));
            result.append(String.format("Max:  %.2f\n", stats.getMax()));
            result.append(String.format("Avg:  %.2f\n", stats.getAverage()));

            // Recent trend (last 5 points)
            result.append("\nRecent trend (oldest → newest):\n");
            int size = metrics.size();
            metrics.subList(Math.max(0, size - 5), size)
                .forEach(dp -> result.append(String.format("  %s: %.2f\n", dp.timestamp(), dp.value())));

            // Anomaly detection (simple: is current > 2x average?)
            double current = metrics.get(size - 1).value();
            if (current > stats.getAverage() * 2.0) {
                result.append(String.format("\n⚠️  ANOMALY: Current value (%.2f) is %.1fx the average",
                    current, current / stats.getAverage()));
            }

            return result.toString();
        } catch (Exception e) {
            return "Error querying metrics: " + e.getMessage();
        }
    }

    // Tool 3: Search runbooks
    public String searchRunbooks(String incidentType, String service) {
        try {
            var runbooks = runbookRepo.findByKeywords(
                List.of(incidentType, service), PageRequest.of(0, 3)
            );

            if (runbooks.isEmpty()) {
                return String.format("No runbooks found for '%s' / '%s'. " +
                    "Consider checking general runbooks for this service type.", incidentType, service);
            }

            var result = new StringBuilder("=== Relevant Runbooks ===\n\n");
            for (var rb : runbooks) {
                result.append(String.format("**%s** (last updated: %s)\n", rb.title(), rb.updatedAt()));
                result.append(rb.content());
                result.append("\n\n---\n\n");
            }

            return result.toString();
        } catch (Exception e) {
            return "Error searching runbooks: " + e.getMessage();
        }
    }

    // Tool 4: Create Jira ticket
    public String createJiraTicket(String summary, String description, String priority) {
        try {
            // Validate priority
            if (!List.of("Highest", "High", "Medium", "Low").contains(priority)) {
                priority = "High";
            }

            var ticketKey = jiraClient.createTicket(
                summary, description, priority,
                "INCIDENT"  // Jira project key
            );

            return String.format("Jira ticket created: %s\nURL: https://your-jira.atlassian.net/browse/%s",
                ticketKey, ticketKey);
        } catch (Exception e) {
            return "Error creating Jira ticket: " + e.getMessage()
                + "\nManually create ticket if needed.";
        }
    }

    private String extractErrorKey(String message) {
        // Extract first meaningful part of error message for grouping
        if (message.contains("Exception")) {
            int idx = message.indexOf("Exception");
            return message.substring(Math.max(0, idx - 30), Math.min(message.length(), idx + 15));
        }
        return message.substring(0, Math.min(message.length(), 80));
    }
}
```

---

## Incident Orchestrator — Core Logic

```java
// orchestrator/IncidentOrchestrator.java
@Service
@Slf4j
public class IncidentOrchestrator {

    private final ClaudeAgentClient claudeClient;
    private final IncidentTools tools;
    private final SlackNotifier slack;
    private final IncidentRepository incidentRepo;
    private final RemediationExecutor remediationExecutor;

    public void processIncident(PagerDutyEvent event) {
        log.info("Processing incident: {}", event.id());

        // Save incident record
        var incident = new Incident();
        incident.setPagerdutyId(event.id());
        incident.setTitle(event.title());
        incident.setSeverity(event.severity());
        incident.setService(event.service());
        incident.setTriggeredAt(Instant.now());
        incident.setStatus(IncidentStatus.ANALYZING);
        incidentRepo.save(incident);

        // Notify Slack: analysis started
        slack.sendAnalysisStarted(incident);

        try {
            // Run Claude agent with tool use
            var diagnosis = runDiagnosisAgent(incident, event);

            incident.setDiagnosis(diagnosis.explanation());
            incident.setRemediationPlan(diagnosis.remediationPlan());
            incident.setStatus(IncidentStatus.AWAITING_APPROVAL);
            incidentRepo.save(incident);

            // Send to Slack with approval buttons
            slack.sendDiagnosisForApproval(incident, diagnosis);

            // Set timeout: if no response in 10 min, escalate
            scheduleApprovalTimeout(incident.getId(), Duration.ofMinutes(10));

        } catch (Exception e) {
            log.error("Error during incident analysis", e);
            incident.setStatus(IncidentStatus.FAILED);
            incident.setDiagnosis("Analysis failed: " + e.getMessage());
            incidentRepo.save(incident);
            slack.sendAnalysisFailed(incident, e.getMessage());
        }
    }

    private DiagnosisResult runDiagnosisAgent(Incident incident, PagerDutyEvent event) {
        // Build tool definitions for Claude
        var toolDefs = buildToolDefinitions();

        // Initial message to Claude
        String systemPrompt = """
            You are an expert SRE (Site Reliability Engineer) diagnosing a production incident.
            
            Your task:
            1. Gather relevant information using the available tools (logs, metrics, runbooks)
            2. Identify the root cause based on evidence
            3. Propose specific, safe remediation steps
            
            IMPORTANT constraints:
            - Only call tools that are relevant to the incident
            - Limit to 6 tool calls maximum (cost and time constraint)
            - Be specific about evidence — quote actual log lines and metric values
            - If uncertain, say so explicitly
            - Never suggest irreversible actions without strong evidence
            
            Output format (after gathering data):
            DIAGNOSIS: [root cause in 2-3 sentences]
            EVIDENCE: [specific logs/metrics supporting diagnosis]
            REMEDIATION: [numbered list of actions, safest first]
            CONFIDENCE: [HIGH/MEDIUM/LOW] — [reason]
            """;

        String userMessage = String.format("""
            Production incident triggered:
            
            Title: %s
            Severity: %s
            Service: %s
            Time: %s
            
            Alert details: %s
            
            Please diagnose this incident and suggest remediation.
            """, event.title(), event.severity(), event.service(),
            event.triggeredAt(), event.alertDetails());

        // Agentic loop
        var messages = new ArrayList<Map<String, Object>>();
        messages.add(Map.of("role", "user", "content", userMessage));

        int toolCallCount = 0;
        int maxToolCalls = 6;

        while (toolCallCount < maxToolCalls) {
            var response = claudeClient.createMessage(systemPrompt, messages, toolDefs, 4000);

            if (response.stopReason().equals("end_turn")) {
                // Claude finished analysis
                return parseDiagnosisFromResponse(response.text());
            }

            if (response.stopReason().equals("tool_use")) {
                // Execute tool calls
                var toolResults = new ArrayList<Map<String, Object>>();

                for (var toolCall : response.toolUses()) {
                    toolCallCount++;
                    log.info("Incident {}: Claude calling tool {} (call {}/{})",
                        incident.getId(), toolCall.name(), toolCallCount, maxToolCalls);

                    String result = executeTool(toolCall.name(), toolCall.input());
                    toolResults.add(Map.of(
                        "type", "tool_result",
                        "tool_use_id", toolCall.id(),
                        "content", result
                    ));
                }

                // Add assistant response and tool results to conversation
                messages.add(Map.of("role", "assistant", "content", response.rawContent()));
                messages.add(Map.of("role", "user", "content", toolResults));
            }
        }

        // Max tool calls reached — ask Claude to conclude with what it has
        messages.add(Map.of("role", "user",
            "content", "Tool call limit reached. Please provide your best diagnosis with available information."));
        var finalResponse = claudeClient.createMessage(systemPrompt, messages, List.of(), 2000);
        return parseDiagnosisFromResponse(finalResponse.text());
    }

    public void handleHumanDecision(String incidentId, String action, String userId) {
        var incident = incidentRepo.findById(incidentId)
            .orElseThrow(() -> new IllegalArgumentException("Incident not found: " + incidentId));

        if (incident.getStatus() != IncidentStatus.AWAITING_APPROVAL) {
            log.warn("Received approval for incident {} in wrong state: {}", incidentId, incident.getStatus());
            return;
        }

        incident.setApprovedBy(userId);
        incident.setApprovedAt(Instant.now());

        switch (action) {
            case "approve" -> {
                incident.setStatus(IncidentStatus.REMEDIATING);
                incidentRepo.save(incident);
                log.info("Incident {} approved by {}. Executing remediation.", incidentId, userId);
                executeRemediation(incident);
            }
            case "decline" -> {
                incident.setStatus(IncidentStatus.DECLINED);
                incidentRepo.save(incident);
                slack.sendDeclinedNotification(incident, userId);
                log.info("Incident {} declined by {}.", incidentId, userId);
            }
        }
    }

    private void executeRemediation(Incident incident) {
        try {
            // Safety check before any action
            validateSafetyConstraints(incident);

            remediationExecutor.execute(incident);

            incident.setStatus(IncidentStatus.RESOLVED);
            incident.setResolvedAt(Instant.now());
            incident.setMttrSeconds(
                Duration.between(incident.getTriggeredAt(), incident.getResolvedAt()).getSeconds()
            );
            incidentRepo.save(incident);

            slack.sendResolutionNotification(incident);
            log.info("Incident {} resolved. MTTR: {}s", incident.getId(), incident.getMttrSeconds());

        } catch (SafetyConstraintException e) {
            log.error("Safety constraint violated for incident {}: {}", incident.getId(), e.getMessage());
            incident.setStatus(IncidentStatus.FAILED);
            incidentRepo.save(incident);
            slack.sendSafetyViolationAlert(incident, e.getMessage());
        }
    }
}
```

---

## Safety Guardrails

Đây là phần **quan trọng nhất** của IntelliOps — và là thứ làm cho nó production-ready.

```java
// safety/SafetyGuard.java
@Component
public class SafetyGuard {

    private static final Set<String> FORBIDDEN_ACTIONS = Set.of(
        "DROP TABLE",
        "DELETE FROM",
        "TRUNCATE",
        "ALTER TABLE",
        "db_migration",
        "schema_change"
    );

    private static final Set<String> REQUIRE_DOUBLE_APPROVAL = Set.of(
        "restart_all_replicas",
        "scale_down_to_zero",
        "clear_cache_all",
        "revoke_all_tokens"
    );

    public void validateRemediation(RemediationPlan plan) {
        // Rule 1: Never auto-apply DB migrations
        if (plan.involvesDatabaseMigration()) {
            throw new SafetyConstraintException(
                "Database migrations require manual execution. " +
                "IntelliOps cannot auto-apply schema changes. " +
                "Please execute manually: " + plan.migrationScript()
            );
        }

        // Rule 2: Never restart production without approval
        if (plan.involvesRestartAll() && !plan.hasDoubleApproval()) {
            throw new SafetyConstraintException(
                "Restarting all replicas requires double approval. " +
                "Current approval: single. Request second approval."
            );
        }

        // Rule 3: Check for forbidden SQL patterns
        if (plan.hasSqlCommands()) {
            for (String sql : plan.sqlCommands()) {
                for (String forbidden : FORBIDDEN_ACTIONS) {
                    if (sql.toUpperCase().contains(forbidden)) {
                        throw new SafetyConstraintException(
                            "Forbidden SQL operation detected: " + forbidden +
                            ". This action requires manual execution."
                        );
                    }
                }
            }
        }

        // Rule 4: Scale down limit — never below minimum replicas
        if (plan.involvesScaleDown()) {
            int targetReplicas = plan.targetReplicaCount();
            int minReplicas = getMinReplicasForService(plan.serviceName());
            if (targetReplicas < minReplicas) {
                throw new SafetyConstraintException(
                    String.format("Cannot scale %s below minimum %d replicas. " +
                        "Requested: %d", plan.serviceName(), minReplicas, targetReplicas)
                );
            }
        }

        // Rule 5: Time-based safety (no auto-actions during peak hours unless P1)
        LocalTime now = LocalTime.now(ZoneId.of("UTC"));
        if (isPeakHours(now) && !plan.incidentSeverity().equals("P1")) {
            throw new SafetyConstraintException(
                "Automated remediation during peak hours (09:00-18:00 UTC) " +
                "requires P1 severity. Current severity: " + plan.incidentSeverity()
            );
        }
    }

    private boolean isPeakHours(LocalTime time) {
        return time.isAfter(LocalTime.of(9, 0)) && time.isBefore(LocalTime.of(18, 0));
    }
}
```

---

## Slack Notification Design

```java
// notification/SlackNotifier.java
@Component
public class SlackNotifier {

    private final Slack slack;
    private final String channelId;

    public void sendDiagnosisForApproval(Incident incident, DiagnosisResult diagnosis) {
        String severityEmoji = switch (incident.getSeverity()) {
            case "P1" -> "🔴";
            case "P2" -> "🟠";
            case "P3" -> "🟡";
            default -> "⚪";
        };

        // Build Block Kit message with action buttons
        var blocks = List.of(
            headerBlock(severityEmoji + " IntelliOps Incident Report"),
            sectionBlock(String.format("*Incident:* %s\n*Service:* `%s` | *Severity:* %s | *Started:* <!date^%d^{time}|%s> UTC",
                incident.getTitle(), incident.getService(), incident.getSeverity(),
                incident.getTriggeredAt().getEpochSecond(), incident.getTriggeredAt())),
            dividerBlock(),
            sectionBlock("*🔍 Diagnosis*\n" + incident.getDiagnosis()),
            sectionBlock("*📋 Evidence*\n```" + diagnosis.evidenceSummary() + "```"),
            sectionBlock("*🔧 Suggested Remediation*\n" + formatRemediationPlan(incident.getRemediationPlan())),
            sectionBlock("*Confidence:* " + diagnosis.confidence() + " — " + diagnosis.confidenceReason()),
            dividerBlock(),
            actionsBlock(
                approveButton("✅ Approve & Execute", "approve_" + incident.getId()),
                approveButton("🔄 Approve Partial", "approve_partial_" + incident.getId()),
                denyButton("❌ Decline", "decline_" + incident.getId())
            ),
            contextBlock("⏰ Auto-escalate to on-call in 10 minutes if no response | IntelliOps v1.0")
        );

        slack.methods().chatPostMessage(r -> r
            .channel(channelId)
            .blocks(blocks)
        );
    }

    public void sendResolutionNotification(Incident incident) {
        String mttr = formatDuration(incident.getMttrSeconds());
        slack.methods().chatPostMessage(r -> r
            .channel(channelId)
            .text(String.format(
                "✅ *Incident Resolved* — %s\n" +
                "Service: `%s` | MTTR: *%s* | Resolved by: @%s\n" +
                "_Please provide feedback: was the diagnosis accurate?_",
                incident.getTitle(), incident.getService(),
                mttr, incident.getApprovedBy()
            ))
        );
    }
}
```

---

## Feedback Loop — Cải thiện theo thời gian

```java
// feedback/FeedbackController.java
@RestController
@RequestMapping("/api/incidents")
public class FeedbackController {

    private final IncidentRepository incidentRepo;
    private final PromptImprovementService promptService;

    @PostMapping("/{id}/feedback")
    public ResponseEntity<Void> submitFeedback(
            @PathVariable String id,
            @RequestBody FeedbackRequest feedback) {

        var incident = incidentRepo.findById(id).orElseThrow();
        incident.setDiagnosisAccurate(feedback.diagnosisAccurate());
        incident.setEngineerNotes(feedback.notes());
        incidentRepo.save(incident);

        // If diagnosis was wrong, log for prompt improvement
        if (!feedback.diagnosisAccurate()) {
            promptService.logMisdiagnosis(
                incident.getTitle(),
                incident.getDiagnosis(),
                feedback.actualRootCause(),
                feedback.notes()
            );
        }

        return ResponseEntity.ok().build();
    }

    // Dashboard: diagnosis accuracy over time
    @GetMapping("/metrics")
    public Map<String, Object> getMetrics(
            @RequestParam @DateTimeFormat(iso = ISO.DATE) LocalDate from,
            @RequestParam @DateTimeFormat(iso = ISO.DATE) LocalDate to) {

        var incidents = incidentRepo.findByDateRange(from, to);
        long total = incidents.size();
        long accurate = incidents.stream().filter(i -> Boolean.TRUE.equals(i.getDiagnosisAccurate())).count();
        long withFeedback = incidents.stream().filter(i -> i.getDiagnosisAccurate() != null).count();

        OptionalDouble avgMttr = incidents.stream()
            .filter(i -> i.getMttrSeconds() != null)
            .mapToLong(Incident::getMttrSeconds)
            .average();

        return Map.of(
            "total_incidents", total,
            "diagnosis_accuracy", withFeedback > 0 ? (double) accurate / withFeedback : 0.0,
            "avg_mttr_seconds", avgMttr.orElse(0),
            "incidents_by_severity", groupBySeverity(incidents)
        );
    }
}
```

---

## Demo Scenario: OOM Incident

**Setup:** Cố tình gây OOM error trong một Spring Boot app:

```java
// Trong demo app — cố tình gây memory leak
@GetMapping("/demo/oom")
public List<byte[]> triggerOom() {
    List<byte[]> leak = new ArrayList<>();
    while (true) {  // Sẽ trigger OOM trong vài giây
        leak.add(new byte[1024 * 1024]);
    }
}
```

**Demo flow:**
```
1. Call /demo/oom → service bắt đầu degrade
2. PagerDuty alert fires (P2: high latency)
3. IntelliOps webhook receives alert
4. Slack message: "IntelliOps is analyzing..."
5. [30 seconds] Slack message với diagnosis:
   "OOM condition in order-service. JVM heap at 98%.
    Root cause: unbounded list growth in /demo/oom endpoint.
    Remediation: restart pod (immediate), scale from 2→3 replicas (preventive)"
6. Click [Approve Restart]
7. Slack: "Executing remediation..."
8. kubectl rollout restart deployment/order-service
9. Slack: "✅ Incident resolved. MTTR: 2m 34s"
```

---

## Metrics: What to Show in Portfolio

Sau khi chạy demo:

| Metric | Target | How to achieve |
|--------|--------|----------------|
| MTTR reduction | 40-60% | Compare với manual incident time |
| Diagnosis accuracy | >70% | Track với feedback form |
| False positive rate | <20% | Log declines + feedback |
| Avg analysis time | <90 seconds | Measure từ webhook→Slack |
| Cost per incident | <$0.20 | Claude token tracking |

---

## How to Present in Portfolio

**One-liner:** "AI incident response assistant — diagnoses production issues với Claude tool use, sends findings to Slack với Approve/Decline, only acts with human approval. Human-in-the-loop design, zero unsafe auto-actions."

**The safety story is your differentiator:**
> "Điều tôi proud nhất về IntelliOps không phải là AI diagnosis — mà là safety architecture. System never touches database schema, never restarts all replicas without confirmation, never acts during peak hours for non-P1 incidents. AI mạnh nhất khi người ta trust nó, và trust đến từ predictable safety boundaries."

Câu đó, nói trong interview, sẽ differentiate bạn từ mọi AI engineer khác trong pool.
