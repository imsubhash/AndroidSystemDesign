# 📊 Android Analytics Library – System Design

---

## 1. Business Idea

Create a lightweight, extensible Android analytics library that helps app teams log events (e.g., button clicks, screen views), store and batch them, and send them to configured analytics endpoints like Firebase or Mixpanel—while being privacy-compliant, offline-resilient, and easy to integrate.

---

## 2. Requirements Clarification

### ✅ Functional Requirements:
- `logEvent()` with custom parameters
- `setUserProperty()` and `setUserId()`
- Support for screen views, session tracking
- Batching + retry with exponential backoff
- Offline support: store events if offline
- Configurable endpoints and batch settings
- Privacy toggle: explicit user consent required

### ❗️Non-Functional Requirements:
- Thread-safe, low memory, no UI jank
- Extensible (support 3rd-party providers)
- Pluggable storage/network
- Background-safe (e.g., WorkManager optional)
- Secure: no PII leakage, configurable privacy

---

## 3. Mathematical Model (Optional but Smart)

Assume:
- 1 user = 20 events/day  
- MAU = 1M  
- Total events/day = 20M  
If each event ≈ 512B → ~10GB/day across all users

Batching 50 events → ~25KB per batch  
=> 400,000 network calls/day vs 20M if unbatched  
**Impact: ~98% fewer calls**

---

## 4. Client–Server Split

| Client (Android SDK)                | Server (Firebase, Mixpanel, or Custom) |
|------------------------------------|----------------------------------------|
| `logEvent()`, batching, session mgmt | Receives event payloads                |
| `StorageManager`, retry, privacy    | Auth + validation + aggregation        |
| `ConfigManager`                    | Stores rules, thresholds               |

---

## 5. API Design (Client)

```kotlin
interface Analytics {
    fun init(token: String)
    fun logEvent(eventName: String, params: Map<String, Any>? = null)
    fun setUserProperty(property: String, value: String)
    fun setUserId(userId: String)
}
```

### AnalyticsConfig
```kotlin
data class AnalyticsConfig(
    val batchSize: Int,
    val batchIntervalMs: Long,
    val maxQueueSize: Int,
    val providers: List<AnalyticsProvider>,
    val privacySettings: PrivacySettings
)
```

---

## 6. High-Level Architecture

### 🧱 Layered Diagram

```
             +--------------------------+
             |   Public API (Analytics) |
             +--------------------------+
                         |
       +----------------+----------------+
       |                                 |
+---------------+               +------------------+
| QueueManager  |<------------->| StorageManager   |
+---------------+               +------------------+
       |                                 |
       v                                 v
+------------------+             +------------------+
| NetworkManager   |             | Room/SharedPrefs |
+------------------+             +------------------+
       |
       v
+------------------+
|  Analytics Server |
+------------------+
```

### Supporting Modules:
- SessionManager → UUID + timestamps
- PrivacyManager → `canCollectData()`
- ConfigManager → dynamic endpoints, batch size

---

## 7. Deep Dive: **QueueManager**

### 🎯 Responsibilities:
- Accept new events
- Enqueue in memory
- Batch and flush to network
- Persist if failure (Room fallback)
- Triggered by `batchSize` or `batchInterval`

### 🔁 Event Lifecycle:
1. `logEvent()` calls `enqueue()`
2. If queue full → evict oldest
3. Flush if:
   - Size ≥ batchSize OR
   - Timer triggered
4. On flush:
   - Try network → success = clear batch
   - Failure = persist to disk (Room)

### ⚖️ Trade-offs:
| Strategy          | Pros                  | Cons                          |
|------------------|-----------------------|-------------------------------|
| In-memory queue  | Fast, async           | Lost on crash                 |
| Room fallback    | Reliable, retry-safe  | Slightly slower I/O           |
| Exponential Backoff | Resilient           | Delay in eventual delivery    |

---

## 8. Tricky Case: **Handling Consent at Runtime**

**Scenario:**  
User opens app → logs events → then revokes consent

**Design Decision:**
- Events captured before consent are not flushed.
- `PrivacyManager.canCollectData()` check at log time & flush time.
- Optional: allow encrypting + storing events until consent is regained

```kotlin
override fun logEvent(...) {
    if (!privacyManager.canCollectData()) return
    ...
}
```

---

## 9. Flowchart

```mermaid
flowchart TD
    A[App Calls logEvent()] --> B[Check Consent]
    B -->|Yes| C[Create AnalyticsEvent]
    C --> D[Enqueue to Memory Queue]
    D --> E[Batch Reached or Timer Triggered?]
    E -->|Yes| F[Send Batch via NetworkManager]
    F -->|Success| G[Clear Sent Events]
    F -->|Fail| H[Persist Events to Room DB]
    B -->|No| Z[Drop or Encrypt Event]
```

---

✅ **Done** — This fits into a 1–1.5 hour interview with scope for follow-ups:
- How to integrate WorkManager
- How to encrypt Room DB
- How to make QueueManager lifecycle-aware
- How to support multiple backends with event filtering