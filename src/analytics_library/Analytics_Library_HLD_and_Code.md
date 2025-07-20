# 📊 Android Analytics Library – High-Level Design (HLD)

---

## 1. Requirements

### ✅ Functional Requirements:
- Track user events such as button clicks, screen views, and custom events.
- Provide support for different types of events (e.g., user properties, session tracking).
- Send the collected analytics data to a backend server.
- Allow configurable analytics endpoints for different environments (e.g., development, staging, production).
- Support batching of events to reduce network requests.
- Enable offline event tracking (storing events when offline and sending them when back online).
- Allow event prioritization (critical vs. non-critical events).

### ❗️Non-Functional Requirements:
- Efficient memory usage and minimal impact on app performance.
- Ensure data privacy and security (handle sensitive user data appropriately).
- Provide a simple API for developers to integrate the library into their apps.
- Support extensibility for adding new event types or destinations.

---

## 2. Core Components

- Public API Interface
- Event Queue Manager
- Storage Manager
- Network Manager
- Session Manager
- Configuration Manager

---

## 3. Interfaces & Data Classes

```kotlin
interface Analytics {
    fun init(token: String)
    fun logEvent(eventName: String, params: Map<String, Any>? = null)
    fun setUserProperty(property: String, value: String)
    fun setUserId(userId: String)
}
```

```kotlin
data class AnalyticsConfig(
    val batchSize: Int = 50,
    val batchIntervalMs: Long = 5*60*60,
    val maxQueueSize: Int = 1000,
    val maxRetryAttempts: Int = 3,
    val providers: List<AnalyticsProvider> = emptyList()
)
```

```kotlin
data class AnalyticsEvent(
    val name: String,
    val params: Map<String, Any>?,
    val timestamp: Long = System.currentTimeMillis(),
    val sessionId: String?,
    val userId: String?,
    val deviceInfo: DeviceInfo
)
```

---

## 4. Analytics Implementation

```kotlin
class AnalyticsImpl private constructor(
    private val context: Context,
    private val config: AnalyticsConfig
) : Analytics {

    private val queueManager: QueueManager = QueueManagerImpl(context, config)
    private val sessionManager: SessionManager = SessionManagerImpl()
    private val storageManager: StorageManager = StorageManagerImpl(context)
    private val networkManager: NetworkManager = NetworkManagerImpl(config)
    private val privacyManager: PrivacyManager = PrivacyManagerImpl(config.privacySettings)

    private var userId: String? = null
    private val deviceInfo = DeviceInfo(context)

    companion object {
        @Volatile private var instance: AnalyticsImpl? = null

        fun init(context: Context, config: AnalyticsConfig) {
            instance ?: synchronized(this) {
                instance ?: AnalyticsImpl(context, config).also { instance = it }
            }
        }

        fun getInstance(): AnalyticsImpl =
            instance ?: throw IllegalStateException("Analytics must be initialized first")
    }

    override fun logEvent(eventName: String, params: Map<String, Any>?) {
        val event = AnalyticsEvent(
            name = eventName,
            params = params,
            sessionId = sessionManager.getCurrentSessionId(),
            userId = userId,
            deviceInfo = deviceInfo
        )

        queueManager.enqueue(event)
    }

    override fun setUserProperty(property: String, value: String) {
        val userProperty = UserProperty(property, value, userId)
        storageManager.saveUserProperty(userProperty)
    }

    override fun setUserId(userId: String) {
        this.userId = userId
        storageManager.saveUserId(userId)
    }

    override fun startSession() {
        sessionManager.startSession()
    }

    override fun endSession() {
        sessionManager.endSession()
    }
}
```

---

## 5. Queue Manager

```kotlin
interface QueueManager {
    fun enqueue(event: AnalyticsEvent)
    fun flush()
}
```

```kotlin
class QueueManagerImpl(
    private val context: Context,
    private val config: AnalyticsConfig
) : QueueManager {

    private val eventQueue = ConcurrentLinkedQueue<AnalyticsEvent>()
    private val coroutineScope = CoroutineScope(Dispatchers.IO + SupervisorJob())
    private val storageManager: StorageManager = StorageManagerImpl(context)
    private val networkManager: NetworkManager = NetworkManagerImpl(config)

    init {
        startBatchProcessor()
        loadPersistedEvents()
    }

    override fun enqueue(event: AnalyticsEvent) {
        if (eventQueue.size >= config.maxQueueSize) {
            eventQueue.poll() // Remove oldest event if queue is full
        }
        eventQueue.offer(event)

        if (eventQueue.size >= config.batchSize) {
            flush()
        }
    }

    override fun flush() {
        coroutineScope.launch {
            val events = mutableListOf<AnalyticsEvent>()
            while (events.size < config.batchSize && !eventQueue.isEmpty()) {
                eventQueue.poll()?.let { events.add(it) }
            }

            if (events.isNotEmpty()) {
                try {
                    networkManager.sendEvents(events)
                } catch (e: Exception) {
                    storageManager.persistEvents(events)
                }
            }
        }
    }

    private fun startBatchProcessor() {
        coroutineScope.launch {
            while (isActive) {
                delay(config.batchIntervalMs)
                flush()
            }
        }
    }

    private fun loadPersistedEvents() {
        coroutineScope.launch {
            val persistedEvents = storageManager.loadPersistedEvents()
            persistedEvents.forEach { enqueue(it) }
            storageManager.clearPersistedEvents()
        }
    }
}
```

---

## 6. Storage Manager

```kotlin
interface StorageManager {
    suspend fun persistEvents(events: List<AnalyticsEvent>)
    suspend fun loadPersistedEvents(): List<AnalyticsEvent>
    suspend fun clearPersistedEvents()
    fun saveUserProperty(property: UserProperty)
    fun saveUserId(userId: String)
}
```

```kotlin
class StorageManagerImpl(private val context: Context) : StorageManager {
    private val database = Room.databaseBuilder(
        context,
        AnalyticsDatabase::class.java,
        "analytics_db"
    ).build()

    override suspend fun persistEvents(events: List<AnalyticsEvent>) {
        database.eventsDao().insertAll(events.map { it.toEntity() })
    }

    override suspend fun loadPersistedEvents(): List<AnalyticsEvent> {
        return database.eventsDao().getAll().map { it.toAnalyticsEvent() }
    }

    override suspend fun clearPersistedEvents() {
        database.eventsDao().deleteAll()
    }

    override fun saveUserProperty(property: UserProperty) {
        database.userPropertiesDao().insert(property.toEntity())
    }

    override fun saveUserId(userId: String) {
        PreferenceManager.getDefaultSharedPreferences(context)
            .edit()
            .putString("user_id", userId)
            .apply()
    }
}
```

---

## 7. Network Manager

```kotlin
interface NetworkManager {
    suspend fun sendEvents(events: List<AnalyticsEvent>)
}

class NetworkManagerImpl(private val config: AnalyticsConfig) : NetworkManager {
    private val retryPolicy = ExponentialBackoffRetry(config.maxRetryAttempts)

    override suspend fun sendEvents(events: List<AnalyticsEvent>) {
        config.providers.forEach { provider ->
            retryPolicy.execute {
                provider.sendEvents(events)
            }
        }
    }
}
```

---

## 8. Session Manager

```kotlin
interface SessionManager {
    fun startSession()
    fun endSession()
    fun getCurrentSessionId(): String?
}

class SessionManagerImpl : SessionManager {
    private var currentSessionId: String? = null
    private var sessionStartTime: Long = 0

    override fun startSession() {
        currentSessionId = UUID.randomUUID().toString()
        sessionStartTime = System.currentTimeMillis()
    }

    override fun endSession() {
        currentSessionId = null
    }

    override fun getCurrentSessionId(): String? = currentSessionId
}
```

---

## 9. Privacy Manager

```kotlin
interface PrivacyManager {
    fun canCollectData(): Boolean
    fun setUserConsent(consent: Boolean)
}

class PrivacyManagerImpl(private val settings: PrivacySettings) : PrivacyManager {
    private var hasUserConsent = false

    override fun canCollectData(): Boolean {
        return hasUserConsent || !settings.requiresExplicitConsent
    }

    override fun setUserConsent(consent: Boolean) {
        hasUserConsent = consent
    }
}
```

---

## 10. Usage Example

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val config = AnalyticsConfig(
            providers = listOf(
                FirebaseAnalyticsProvider(),
                MixpanelProvider()
            ),
            privacySettings = PrivacySettings(requiresExplicitConsent = true)
        )

        Analytics.init(applicationContext, config)

        Analytics.getInstance().logEvent(
            "button_click",
            mapOf(
                "button_id" to "submit_button",
                "screen_name" to "checkout"
            )
        )

        Analytics.getInstance().setUserProperty("user_type", "premium")
    }
}
```