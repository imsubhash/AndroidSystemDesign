
# ✅ Offline-First TODO App – Android (System Design)

## 📌 Requirements Recap

- Support offline creation/update/deletion of TODOs.
- Sync TODOs with remote server (eventually consistent).
- Filter tasks based on **specific day**.
- Show latest status from local DB, then sync in background.
- Handle conflicts (e.g., last write wins).

---

## 🧠 Core Concepts

- **Room DB**: Local source of truth.
- **Flow**: Observability for UI state.
- **WorkManager**: Background sync for offline events.
- **Repository Pattern**: Abstraction over DB + Network.
- **ViewModel**: Prepares UI state.

---

## 🧱 Entity Layer (Room)

```kotlin
@Entity(tableName = "todos")
data class TodoEntity(
    @PrimaryKey val id: String = UUID.randomUUID().toString(),
    val title: String,
    val description: String? = null,
    val dueDateMillis: Long, // used for filtering by day
    val isCompleted: Boolean = false,
    val isSynced: Boolean = false,
    val lastModified: Long = System.currentTimeMillis()
)
```

---

## 🧩 DAO Layer

```kotlin
@Dao
interface TodoDao {

    @Query("SELECT * FROM todos WHERE dueDateMillis BETWEEN :start AND :end")
    fun getTodosForDay(start: Long, end: Long): Flow<List<TodoEntity>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun upsert(todo: TodoEntity)

    @Query("SELECT * FROM todos WHERE isSynced = 0")
    suspend fun getPendingSyncTodos(): List<TodoEntity>
}
```

---

## 🔌 Retrofit API (Sample)

```kotlin
interface TodoApiService {
    @POST("/todos")
    suspend fun syncTodo(@Body todo: TodoEntity): Response<Unit>

    @GET("/todos")
    suspend fun fetchTodos(): List<TodoEntity>
}
```

---

## 🧰 Repository

```kotlin
class TodoRepository @Inject constructor(
    private val dao: TodoDao,
    private val api: TodoApiService
) {
    fun getTodosForDay(dayMillis: Long): Flow<List<TodoEntity>> {
        val start = dayMillis.startOfDay()
        val end = dayMillis.endOfDay()
        return dao.getTodosForDay(start, end)
    }

    suspend fun addOrUpdateTodo(todo: TodoEntity) {
        val updated = todo.copy(lastModified = nowMillis(), isSynced = false)
        dao.upsert(updated)
    }

    suspend fun syncPendingTodos() {
        val pending = dao.getPendingSyncTodos()
        pending.forEach { todo ->
            try {
                api.syncTodo(todo)
                dao.upsert(todo.copy(isSynced = true))
            } catch (_: Exception) {
                // Handle retry via WorkManager
            }
        }
    }
}
```

---

## 📦 Sync Worker (WorkManager)

```kotlin
class TodoSyncWorker(
    context: Context,
    workerParams: WorkerParameters,
    private val repo: TodoRepository
) : CoroutineWorker(context, workerParams) {

    override suspend fun doWork(): Result {
        return try {
            repo.syncPendingTodos()
            Result.success()
        } catch (e: Exception) {
            Result.retry()
        }
    }
}
```

Schedule worker:

```kotlin
WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "sync_todos",
    ExistingPeriodicWorkPolicy.KEEP,
    PeriodicWorkRequestBuilder<TodoSyncWorker>(15, TimeUnit.MINUTES).build()
)
```

---

## 🧠 ViewModel

```kotlin
@HiltViewModel
class TodoViewModel @Inject constructor(
    private val repository: TodoRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(TodoUiState())
    val uiState: StateFlow<TodoUiState> = _uiState.asStateFlow()

    init {
        loadTodosForDate(_uiState.value.selectedDate)
    }

    fun loadTodosForDate(date: LocalDate) {
        _uiState.update { it.copy(selectedDate = date, isLoading = true) }

        viewModelScope.launch {
            repository.getTodosForDay(date.toMillis())
                .catch { e -> _uiState.update { it.copy(error = e.message) } }
                .collect { todos ->
                    _uiState.update { it.copy(todos = todos, isLoading = false, error = null) }
                }
        }
    }

    fun markTodoComplete(todo: TodoEntity, completed: Boolean) {
        viewModelScope.launch {
            repository.addOrUpdateTodo(todo.copy(isCompleted = completed))
        }
    }
}
```

---

## 📦 UI State (used in ViewModel)

```kotlin
data class TodoUiState(
    val selectedDate: LocalDate = LocalDate.now(),
    val todos: List<TodoEntity> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null
)
```

---

## ✅ Summary

| Concern               | Approach                       |
|-----------------------|--------------------------------|
| Offline Support       | Room DB                        |
| Sync Mechanism        | WorkManager + Repository       |
| Server Conflict       | Last write wins (timestamp)    |
| Daily Filtering       | Query using dueDate timestamps |
| Reactive UI Updates   | Kotlin Flow                    |
| Decoupled Architecture| MVVM + Repository              |
