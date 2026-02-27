# 🔔 Notification Prioritization Engine
### Cyepro Solutions — Round 1 AI-Native Solution Crafting Test

---

## 📌 Problem Overview

Modern products send notifications from dozens of feature sources — messages, reminders, alerts, promotions, system events, and more. This creates a compounding problem:

- Users get **too many notifications**, leading to alert fatigue
- **Repetitive or near-duplicate** messages erode trust
- **Low-value notifications** arrive at the wrong time while critical ones get buried
- There is no intelligent **priority arbitration** between competing notification sources

The goal is to build a **Notification Prioritization Engine** — a decision service that evaluates every incoming notification event and classifies it as:

| Decision | Meaning |
|----------|---------|
| **Now** | Send immediately |
| **Later** | Defer/schedule for an optimal time |
| **Never** | Suppress entirely |

---

## 🧠 Understanding of Requirements

### Core Functional Requirements
- **Classify** each notification: `Now` / `Later` / `Never`
- **Deduplicate**: Handle both exact and near-duplicate notifications
- **Reduce alert fatigue**: Respect cooldowns, caps, and user notification history
- **Conflict resolution**: Handle urgency vs. noisiness tradeoffs intelligently
- **Human-configurable rules**: Operators can tune logic without code deployments
- **Explainability**: Every decision must be logged with a clear reason
- **Graceful degradation**: Fail safely when AI or dependent services are unavailable

### Non-Functional Requirements
- Handle **thousands of events per minute** (high throughput)
- **Low-latency** decisions (target < 100ms p99 for rule-based path)
- Time-sensitive notifications must **not be delayed** beyond expiry
- System must be **auditable and explainable**
- Important notifications must **never be silently lost**

---

## 💡 Assumptions Made

1. A **user profile service** exists and can provide basic preferences (do-not-disturb windows, opted-out channels, etc.).
2. A **Redis-compatible cache** is available for storing recent notification history and counters.
3. Each service/source calling this engine is **authenticated** and trusted.
4. The AI/LLM component is used for **borderline or ambiguous cases only** — deterministic rules handle the majority of traffic.
5. `dedupe_key` may be missing or unreliable, so **semantic similarity** is used as a fallback deduplication strategy.
6. Notification **history is retained for a rolling 24-hour window** per user (configurable).
7. All times are in **UTC**; user timezone offsets are stored in the user profile.

---

## 🏗️ High-Level Architecture

```
                        ┌─────────────────────────────────────────┐
                        │         Notification Event               │
                        │  (user_id, event_type, channel, etc.)   │
                        └───────────────────┬─────────────────────┘
                                            │
                                    ┌───────▼────────┐
                                    │  Ingestion API  │  (REST/gRPC)
                                    └───────┬────────┘
                                            │
                          ┌─────────────────▼──────────────────────┐
                          │         Decision Orchestrator           │
                          │  ┌──────────┐  ┌──────────────────┐    │
                          │  │  Rule    │  │  AI Scoring       │    │
                          │  │  Engine  │  │  (LLM / ML Model) │    │
                          │  └────┬─────┘  └────────┬─────────┘    │
                          │       │  ←── Merge ───── │              │
                          │  ┌────▼──────────────────▼──────────┐  │
                          │  │     Final Decision Resolver       │  │
                          │  │  (Now / Later / Never + reason)   │  │
                          │  └──────────────────────────────────┘  │
                          └──────────────────┬─────────────────────┘
                                             │
              ┌─────────────────────┬────────┴──────────────────┐
              ▼                     ▼                            ▼
     ┌────────────────┐  ┌──────────────────┐      ┌───────────────────┐
     │ Delivery Queue │  │  Scheduler Queue │      │   Audit Log       │
     │  (Send Now)    │  │  (Send Later)    │      │   (Suppressed)    │
     └────────────────┘  └──────────────────┘      └───────────────────┘
```

### Key Components

| Component | Responsibility |
|-----------|---------------|
| **Ingestion API** | Validates and normalizes incoming notification events |
| **Deduplication Service** | Detects exact and semantic near-duplicates |
| **Rule Engine** | Fast, configurable rule evaluation (YAML/JSON rules, no deploy needed) |
| **AI Scoring Module** | LLM/ML-based classification for ambiguous cases |
| **Decision Orchestrator** | Merges rule + AI signals into a final verdict |
| **Scheduler** | Manages deferred notifications and optimal delivery times |
| **Audit Logger** | Records every decision with full reasoning |
| **Config Store** | Human-editable rules and thresholds (e.g., Firestore, Redis, or a CMS) |

---

## ⚙️ Decision Logic Design

### Priority Classification Flow

```
Step 1: Validate & Normalize Input
Step 2: Check expires_at → if expired, decision = NEVER (stale)
Step 3: Check deduplication → if duplicate, decision = NEVER (duplicate)
Step 4: Check user DND / channel opt-outs → if blocked, decision = NEVER or LATER
Step 5: Run Rule Engine
        → HIGH priority + urgent event type → NOW
        → PROMO / low-value during noisy period → LATER or NEVER
Step 6: Check alert fatigue counters
        → If user received >N notifications in last M minutes → LATER
Step 7: If undecided, call AI Scoring Module (with timeout fallback)
Step 8: Resolve final decision + generate reason log
```

### Rule Engine (Human-Configurable)

Rules are stored as structured YAML/JSON and evaluated without code redeployment:

```yaml
rules:
  - name: "Critical Security Alert"
    condition:
      event_type: ["account_breach", "suspicious_login"]
    action: NOW
    override_fatigue: true

  - name: "Promotional Suppression"
    condition:
      event_type: ["promo", "marketing"]
      recent_count_1h: { gte: 3 }
    action: NEVER

  - name: "Defer During DND"
    condition:
      user_dnd: true
      priority_hint: { not_in: ["critical", "urgent"] }
    action: LATER
    defer_until: "next_active_window"
```

### AI Scoring Module (Used for Ambiguous Cases)

The AI module receives borderline notifications — those not clearly handled by rules — and returns a classification with a confidence score. It considers:
- Semantic similarity to recent notifications (near-duplicate detection)
- Content tone and urgency signals
- User engagement patterns

**Timeout:** 300ms hard timeout. On timeout or error, falls back to rule engine default or `LATER`.

---

## 🗄️ Minimal Data Model

### `notification_events` (Incoming log)
| Field | Type | Description |
|-------|------|-------------|
| `event_id` | UUID | Primary key |
| `user_id` | String | Target user |
| `event_type` | String | Category of notification |
| `message` | Text | Notification body |
| `source` | String | Originating service |
| `priority_hint` | Enum | low / medium / high / critical |
| `channel` | Enum | push / email / sms / in-app |
| `timestamp` | Timestamp | When event was received |
| `expires_at` | Timestamp | Optional expiry |
| `dedupe_key` | String | Optional client-provided key |
| `metadata` | JSON | Extra context fields |

### `notification_decisions` (Audit log)
| Field | Type | Description |
|-------|------|-------------|
| `decision_id` | UUID | Primary key |
| `event_id` | UUID | FK to notification_events |
| `decision` | Enum | NOW / LATER / NEVER |
| `reason` | Text | Human-readable explanation |
| `rule_matched` | String | Which rule triggered (if any) |
| `ai_score` | Float | Confidence from AI module |
| `decided_at` | Timestamp | Decision timestamp |
| `scheduled_for` | Timestamp | For LATER decisions |

### `user_notification_history` (In Redis / Cache)
| Key Pattern | Value | TTL |
|-------------|-------|-----|
| `user:{id}:count:{channel}` | Integer counter | 1 hour rolling |
| `user:{id}:last_sent` | Timestamp | 24 hours |
| `user:{id}:dedupe:{hash}` | Boolean | 24 hours |

### `suppression_rules` (Config store)
| Field | Type | Description |
|-------|------|-------------|
| `rule_id` | UUID | Primary key |
| `name` | String | Human-readable label |
| `conditions` | JSON | Rule conditions |
| `action` | Enum | NOW / LATER / NEVER |
| `is_active` | Boolean | Enable/disable without delete |
| `updated_by` | String | Operator who last changed it |

---

## 🔌 API / Service Interfaces

### 1. `POST /v1/notifications/evaluate`
Evaluates a single incoming notification event.

**Request:**
```json
{
  "user_id": "u_12345",
  "event_type": "message",
  "message": "You have a new message from Alex",
  "source": "chat-service",
  "priority_hint": "high",
  "channel": "push",
  "timestamp": "2025-02-25T10:00:00Z",
  "dedupe_key": "msg_9876"
}
```

**Response:**
```json
{
  "decision": "NOW",
  "reason": "High-priority direct message; user active; no duplicates detected.",
  "decision_id": "d_abc123",
  "scheduled_for": null
}
```

---

### 2. `POST /v1/notifications/batch-evaluate`
Evaluates multiple events in one call (for burst ingestion).

---

### 3. `GET /v1/decisions/{decision_id}`
Retrieves a past decision with full audit trail.

---

### 4. `GET /v1/users/{user_id}/history`
Returns recent notification history and current fatigue counters for a user.

---

### 5. `PUT /v1/rules/{rule_id}`
Updates a suppression/prioritization rule without code deployment. Requires operator authentication.

---

## 🔁 Duplicate Prevention Approach

### Exact Duplicates
- Hash the combination of `(user_id + event_type + dedupe_key OR message_hash)` 
- Check against Redis key: `user:{id}:dedupe:{hash}` with 24h TTL
- If hash exists → decision is `NEVER` with reason: "Exact duplicate suppressed"

### Near-Duplicates (Semantic)
- Generate an **embedding vector** of the notification message (via lightweight embedding model or LLM)
- Compare cosine similarity against the last N notifications for the user
- If similarity > configurable threshold (e.g., 0.92) → flag as near-duplicate → suppress or batch

### Batching
- Near-duplicates from the same source within a short window (e.g., 5 min) can be **collapsed into a digest**: "You have 5 new updates from X"

---

## 😮‍💨 Alert Fatigue Strategy

| Strategy | Implementation |
|----------|---------------|
| **Per-channel caps** | Max N notifications per channel per hour per user (configurable) |
| **Global user cap** | Max M total notifications per hour; excess → `LATER` |
| **Cooldown periods** | After a high-volume burst, enforce a quiet period |
| **Priority bypass** | `critical` events bypass fatigue limits with `override_fatigue: true` |
| **Digest batching** | Low-priority notifications accumulate and send as a single digest |
| **DND windows** | No non-critical notifications during user-defined quiet hours |
| **Engagement signals** | Reduce frequency for users with low open rates (future ML signal) |

---

## 🛡️ Fallback Strategy (AI/Service Unavailable)

```
Scenario                  │ Fallback Behavior
──────────────────────────┼──────────────────────────────────────────────────
AI module timeout (>300ms)│ Use rule engine result only; log AI as "skipped"
AI module error           │ Same as above
Rule engine failure       │ Default to LATER (never silently drop; log error)
History service down      │ Skip fatigue checks; apply conservative caps
Config store unreachable  │ Use last cached rule set (stale-reads allowed)
Full system degradation   │ Apply emergency defaults: critical=NOW, rest=LATER
```

**Key Principle:** Important notifications must **never be silently lost**. If in doubt, defer — don't suppress.

---

## 📊 Metrics and Monitoring Plan

### Key Metrics

| Metric | Description |
|--------|-------------|
| `decisions_total{result}` | Count of NOW / LATER / NEVER decisions |
| `ai_module_latency_ms` | P50/P95/P99 AI response times |
| `rule_engine_latency_ms` | Rule evaluation latency |
| `dedup_hit_rate` | % of events caught as duplicates |
| `fatigue_suppression_rate` | % suppressed due to alert fatigue |
| `fallback_invocation_count` | Times the fallback path was triggered |
| `notification_expiry_rate` | % that expired before delivery |

### Alerting

- AI module error rate > 5% → PagerDuty alert
- Decision latency P99 > 500ms → warning
- Dedup cache miss rate spike → investigate data model
- Any `NEVER` decision on `critical` priority → immediate alert

### Dashboards

- Real-time decision distribution (Now/Later/Never ratio)
- Per-user notification volume heatmap
- Rule match frequency (which rules are triggering most)
- AI vs. rule-based decision split

---

## 🛠️ Tools & Technologies

| Layer | Technology |
|-------|-----------|
| **API** | FastAPI (Python) or Node.js + Express |
| **Rule Engine** | Custom YAML-driven evaluator or open-source (e.g., `py-rules-engine`) |
| **AI/LLM** | Claude API or OpenAI (for ambiguous classification + semantic similarity) |
| **Embeddings** | `text-embedding-3-small` or `sentence-transformers` for near-dedup |
| **Cache/History** | Redis (counters, dedup hashes, DND state) |
| **Message Queue** | Kafka or RabbitMQ (for async delivery and scheduling) |
| **Audit Store** | PostgreSQL (decisions log) |
| **Config Store** | Firestore / Redis / internal CMS (human-editable rules) |
| **Monitoring** | Prometheus + Grafana + PagerDuty |

---

## ✅ How This Solution Meets All Constraints

| Constraint | How Addressed |
|-----------|--------------|
| High event volume (thousands/min) | Stateless decision service + Redis for sub-millisecond lookups |
| Low-latency decisions | Rule engine handles ~90% of traffic without AI; AI path has 300ms timeout |
| Time-sensitive notifications | `expires_at` checked first; stale events → NEVER immediately |
| Optional/promotional deprioritization | Rule-based suppression during noisy periods |
| Multiple services per user | Per-user fatigue counters aggregate across all sources |
| Unreliable dedupe keys | Fallback to content hashing and semantic embedding similarity |
| Explainability & auditability | Every decision logged with matched rule, AI score, and plain-text reason |
| Important notifications must not be lost | Fallback defaults to LATER; critical events bypass all fatigue limits |

---

## 🔚 Final Conclusion

This Notification Prioritization Engine takes a **layered, pragmatic approach**:

1. **Deterministic rules first** — fast, explainable, human-configurable
2. **AI for the hard cases** — handles semantic nuance and ambiguity
3. **Graceful degradation** — the system is designed to never silently lose critical notifications
4. **Observable by design** — every decision is auditable, every metric is exportable

The architecture is horizontally scalable, ops-friendly (rules without code deploys), and balances user experience (less noise) with reliability (nothing important is lost). It's built to evolve — as user behavior data accumulates, the AI scoring module can be continuously improved without changing the core decision flow.


