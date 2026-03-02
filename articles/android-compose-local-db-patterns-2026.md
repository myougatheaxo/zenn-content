---
title: "ローカルDB設計パターン — Room + Repository + ViewModelの最適構成"
emoji: "🗄️"
type: "tech"
topics: ["android", "kotlin", "room", "architecture"]
published: true
---

## この記事で学べること

Room Databaseを使った**Repository + ViewModel**のクリーンな設計パターンを解説します。

---

## Entity設計

```kotlin
@Entity(tableName = "tasks")
data class TaskEntity(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val title: String,
    val description: String = "",
    val isCompleted: Boolean = false,
    val priority: Int = 0,
    val createdAt: Long = System.currentTimeMillis(),
    val updatedAt: Long = System.currentTimeMillis()
)

// 複合インデックス
@Entity(
    tableName = "task_tags",
    primaryKeys = ["taskId", "tagId"],
    foreignKeys = [
        ForeignKey(
            entity = TaskEntity::class,
            parentColumns = ["id"],
            childColumns = ["taskId"],
            onDelete = ForeignKey.CASCADE
        )
    ],
    indices = [Index("tagId")]
)
data class TaskTagCrossRef(
    val taskId: Int,
    val tagId: Int
)
```

---

## DAO

```kotlin
@Dao
interface TaskDao {
    @Query("SELECT * FROM tasks ORDER BY priority DESC, createdAt DESC")
    fun observeAll(): Flow<List<TaskEntity>>

    @Query("SELECT * FROM tasks WHERE isCompleted = :completed ORDER BY createdAt DESC")
    fun observeByStatus(completed: Boolean): Flow<List<TaskEntity>>

    @Query("SELECT * FROM tasks WHERE title LIKE '%' || :query || '%'")
    fun search(query: String): Flow<List<TaskEntity>>

    @Query("SELECT * FROM tasks WHERE id = :id")
    suspend fun getById(id: Int): TaskEntity?

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(task: TaskEntity): Long

    @Update
    suspend fun update(task: TaskEntity)

    @Query("UPDATE tasks SET isCompleted = :completed, updatedAt = :now WHERE id = :id")
    suspend fun updateStatus(id: Int, completed: Boolean, now: Long = System.currentTimeMillis())

    @Delete
    suspend fun delete(task: TaskEntity)

    @Query("DELETE FROM tasks WHERE isCompleted = 1")
    suspend fun deleteCompleted(): Int

    @Query("SELECT COUNT(*) FROM tasks WHERE isCompleted = 0")
    fun observePendingCount(): Flow<Int>
}
```

---

## Repository

```kotlin
class TaskRepository(private val dao: TaskDao) {
    fun observeAll(): Flow<List<Task>> =
        dao.observeAll().map { entities ->
            entities.map { it.toDomain() }
        }

    fun observeByStatus(completed: Boolean): Flow<List<Task>> =
        dao.observeByStatus(completed).map { entities ->
            entities.map { it.toDomain() }
        }

    fun search(query: String): Flow<List<Task>> =
        dao.search(query).map { entities ->
            entities.map { it.toDomain() }
        }

    suspend fun add(title: String, description: String, priority: Int): Long {
        val entity = TaskEntity(
            title = title,
            description = description,
            priority = priority
        )
        return dao.insert(entity)
    }

    suspend fun toggleComplete(id: Int, completed: Boolean) {
        dao.updateStatus(id, completed)
    }

    suspend fun delete(task: Task) {
        dao.delete(task.toEntity())
    }
}

// Domain ↔ Entity 変換
fun TaskEntity.toDomain() = Task(id, title, description, isCompleted, priority, createdAt)
fun Task.toEntity() = TaskEntity(id, title, description, isCompleted, priority, createdAt)
```

---

## ViewModel

```kotlin
class TaskViewModel(private val repository: TaskRepository) : ViewModel() {

    private val _filterState = MutableStateFlow(FilterState())
    val filterState: StateFlow<FilterState> = _filterState.asStateFlow()

    val tasks: StateFlow<List<Task>> = _filterState
        .flatMapLatest { filter ->
            when {
                filter.query.isNotBlank() -> repository.search(filter.query)
                filter.showCompleted != null -> repository.observeByStatus(filter.showCompleted)
                else -> repository.observeAll()
            }
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    val pendingCount: StateFlow<Int> = repository.observePendingCount()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), 0)

    fun addTask(title: String, description: String = "", priority: Int = 0) {
        viewModelScope.launch {
            repository.add(title, description, priority)
        }
    }

    fun toggleComplete(id: Int, completed: Boolean) {
        viewModelScope.launch {
            repository.toggleComplete(id, completed)
        }
    }

    fun setFilter(query: String = "", showCompleted: Boolean? = null) {
        _filterState.value = FilterState(query, showCompleted)
    }
}

data class FilterState(
    val query: String = "",
    val showCompleted: Boolean? = null
)
```

---

## まとめ

- Entity: `@Entity` + `@PrimaryKey(autoGenerate = true)`
- DAO: `Flow<List<T>>`で変更監視、`suspend`で書き込み
- Repository: Entity→Domain変換レイヤー
- ViewModel: `flatMapLatest`でフィルター連動
- `stateIn`でFlowをStateFlowに変換
- `OnConflictStrategy.REPLACE`でUPSERT

---

8種類のAndroidアプリテンプレート（Room + Repository設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room Database完全ガイド](https://zenn.dev/myougatheaxo/articles/android-room-database-2026)
- [Coroutines & Flowガイド](https://zenn.dev/myougatheaxo/articles/kotlin-coroutines-flow-2026)
- [Hilt依存性注入ガイド](https://zenn.dev/myougatheaxo/articles/android-hilt-dependency-injection-2026)
