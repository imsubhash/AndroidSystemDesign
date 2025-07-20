# 📊 Android Analytics Library – System Design

---

## 1. Business Idea

Create a lightweight, extensible Android analytics library to help app teams log events (e.g., button clicks, screen views), store and batch them, and send to configurable analytics endpoints like Firebase or Mixpanel—while being **privacy-compliant**, **offline-resilient**, and **easy to integrate**.

---

## 2. Requirements Clarification

### ✅ Functional Requirements
- `logEvent()` with custom parameters  
- `setUserProperty()` and `setUserId()`  
- Support for screen views and session tracking  
- Batching + retry with exponential backoff  
- Offline support: persist events  
- Configurable endpoints, batch size, and interval  
- Privacy toggle: explicit user consent required  

### ❗️Non-Functional Requirements
- Thread-safe, low memory, no UI jank  
- Extensible (support 3rd-party providers)  
- Pluggable storage/network layers  
- Background-safe (WorkManager optional)  
- Secure: no PII leakage; privacy-compliant  

---

## 3. Mathematical Model (Optional)

Assume:
- 1 user = 20 events/day  
- MAU = 1M  
- Total = **20M events/day**  
- Avg event size ≈ 512 bytes → ≈ **10 GB/day**

Batching 50 events → 25KB per batch →  
= **400,000 network calls/day** vs 20M unbatched  
✅ **~98% fewer calls**

---

## 4. Client–Server Split

| Client (Android SDK)             | Server (Firebase/Mixpanel/Custom) |
|----------------------------------|------------------------------------|
| `logEvent()`, batching, privacy  | Receives event payloads            |
| Queueing, storage, retry logic   | Auth, validation, aggregation      |
| ConfigManager                    | Rules, thresholds, reporting       |

---

## 5. API Design (Client)

```kotlin
interface Analytics {
    fun init(token: String, config: AnalyticsConfig)
    fun logEvent(eventName: String, params: Map<String, Any>? = null)
    fun setUserProperty(property: String, value: String)
    fun setUserId(userId: String)
}
```

### `AnalyticsConfig`
```kotlin
data class AnalyticsConfig(
    val batchSize: Int = 50,
    val batchIntervalMs: Long = 30000L,
    val maxQueueSize: Int = 1000,
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
- **SessionManager** → UUID, timestamps  
- **PrivacyManager** → `canCollectData()`  
- **ConfigManager** → batch size, endpoints  

---

## 7. Deep Dive: **QueueManager**

### 🎯 Responsibilities
- Accepts events
- Enqueues in memory
- Flushes in batch (size/time)
- Persists if failure (Room fallback)

### 🔁 Event Lifecycle

1. `logEvent()` → `QueueManager.enqueue()`  
2. If full → evict oldest  
3. Flush triggers:
   - `queue.size >= batchSize`
   - `batchInterval` timer  
4. On flush:
   - Try network → if success → clear  
   - Else → save batch to Room  

---

## 8. Trade-offs

| Strategy             | Pros                  | Cons                      |
|----------------------|-----------------------|---------------------------|
| In-memory queue      | Fast, async           | Lost on crash             |
| Room fallback        | Reliable, retry-safe  | Slightly slower I/O       |
| Exponential backoff  | Resilient             | Delayed delivery          |

---

## 9. Tricky Case: **Consent Revoked at Runtime**

### ❓ Scenario  
User opens app → events logged → revokes consent.

### ✅ Design Decisions
- Events logged before consent are **not flushed**.
- `PrivacyManager.canCollectData()` is checked **on log & flush**.
- Optional: Encrypt & store events till consent returns.

```kotlin
override fun logEvent(eventName: String, params: Map<String, Any>?) {
    if (!privacyManager.canCollectData()) return
    queueManager.enqueue(Event(eventName, params))
}
```

---

## 10. Flowchart – Event Logging Lifecycle
![Screenshot 2025-07-20.png](Screenshot%202025-07-20.png)

```
