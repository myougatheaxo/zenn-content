---
title: "Room + Flow CRUD完全ガイド — リアクティブなDB操作"
emoji: "🗄️"
type: "tech"
topics: ["android", "kotlin", "room", "flow"]
published: true
---

## この記事で学べること

**Room + Flow**のCRUD操作（リアクティブなクエリ、Upsert、一括操作、Compose連携）を解説します。

---

## Entity定義

```kotlin
@Entity(tableName = "tasks")
data class TaskEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val title: String,
    val description: String = "",
    val isCompleted: Boolean = false,
    val priority: Int = 0, // 0=低, 1=中, 2=高
    val createdAt: Long = System.currentTimeMillis(),
    val updatedAt: Long = System.currentTimeMillis()
)
```

---

## DAO（Flow対応）

```kotlin
@Dao
interface TaskDao {

    // 全件取得（リアクティブ）
    @Query("SELECT * FROM tasks ORDER BY priority DESC, createdAt DESC")
    fun observeAll(): Flow<List<TaskEntity>>

    // フィルター付き
    @Query("SELECT * FROM tasks WHERE isCompleted = :completed ORDER BY createdAt DESC")
    fun observeByStatus(completed: Boolean): Flow<List<TaskEntity>>

    // 検索
    @Query("SELECT * FROM tasks WHERE title LIKE '%' || :query || '%' ORDER BY createdAt DESC")
    fun search(query: String): Flow<List<TaskEntity>>

    // 件数
    @Query("SELECT COUNT(*) FROM tasks WHERE isCompleted = 0")
    fun observePendingCount(): Flow<Int>

    // 単件取得（suspend）
    @Query("SELECT * FROM tasks WHERE id = :id")
    suspend fun getById(id: Long): TaskEntity?

    // 挿入
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(task: TaskEntity): Long

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(tasks: List<TaskEntity>)

    // Upsert (Room 2.5+)
    @Upsert
    suspend fun upsert(task: TaskEntity)

    // 更新
    @Update
    suspend fun update(task: TaskEntity)

    // 完了トグル
    @Query("UPDATE tasks SET isCompleted = NOT isCompleted, updatedAt = :now WHERE id = :id")
    suspend fun toggleCompleted(id: Long, now: Long = System.currentTimeMillis())

    // 削除
    @Delete
    suspend fun delete(task: TaskEntity)

    @Query("DELETE FROM tasks WHERE isCompleted = 1")
    suspend fun deleteCompleted(): Int

    @Query("DELETE FROM tasks")
    suspend fun deleteAll()
}
```

---

## Repository

```kotlin
class TaskRepository @Inject constructor(private val dao: TaskDao) {

    fun observeAll(): Flow<List<Task>> = dao.observeAll().map { entities ->
        entities.map { it.toDomain() }
    }

    fun observeByStatus(completed: Boolean) = dao.observeByStatus(completed).map { entities ->
        entities.map { it.toDomain() }
    }

    fun search(query: String) = dao.search(query).map { entities ->
        entities.map { it.toDomain() }
    }

    val pendingCount: Flow<Int> = dao.observePendingCount()

    suspend fun create(title: String, description: String, priority: Int): Long {
        return dao.insert(TaskEntity(title = title, description = description, priority = priority))
    }

    suspend fun toggleCompleted(id: Long) = dao.toggleCompleted(id)

    suspend fun delete(id: Long) {
        dao.getById(id)?.let { dao.delete(it) }
    }

    suspend fun deleteCompleted() = dao.deleteCompleted()
}

// マッピング
fun TaskEntity.toDomain() = Task(
    id = id,
    title = title,
    description = description,
    isCompleted = isCompleted,
    priority = priority,
    createdAt = createdAt
)
```

---

## ViewModel

```kotlin
@HiltViewModel
class TaskViewModel @Inject constructor(
    private val repository: TaskRepository
) : ViewModel() {

    private val _filter = MutableStateFlow(TaskFilter.ALL)

    val tasks: StateFlow<List<Task>> = _filter.flatMapLatest { filter ->
        when (filter) {
            TaskFilter.ALL -> repository.observeAll()
            TaskFilter.ACTIVE -> repository.observeByStatus(false)
            TaskFilter.COMPLETED -> repository.observeByStatus(true)
        }
    }.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    val pendingCount = repository.pendingCount
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), 0)

    fun setFilter(filter: TaskFilter) {
        _filter.value = filter
    }

    fun addTask(title: String, description: String = "", priority: Int = 0) {
        viewModelScope.launch { repository.create(title, description, priority) }
    }

    fun toggleTask(id: Long) {
        viewModelScope.launch { repository.toggleCompleted(id) }
    }

    fun deleteTask(id: Long) {
        viewModelScope.launch { repository.delete(id) }
    }
}

enum class TaskFilter { ALL, ACTIVE, COMPLETED }
```

---

## Compose UI

```kotlin
@Composable
fun TaskListScreen(viewModel: TaskViewModel = hiltViewModel()) {
    val tasks by viewModel.tasks.collectAsStateWithLifecycle()
    val pendingCount by viewModel.pendingCount.collectAsStateWithLifecycle()

    Scaffold(
        topBar = {
            TopAppBar(title = { Text("タスク ($pendingCount 件未完了)") })
        },
        floatingActionButton = {
            FloatingActionButton(onClick = { /* 追加ダイアログ */ }) {
                Icon(Icons.Default.Add, null)
            }
        }
    ) { padding ->
        LazyColumn(Modifier.padding(padding)) {
            items(tasks, key = { it.id }) { task ->
                TaskItem(
                    task = task,
                    onToggle = { viewModel.toggleTask(task.id) },
                    onDelete = { viewModel.deleteTask(task.id) },
                    modifier = Modifier.animateItem()
                )
            }
        }
    }
}
```

---

## まとめ

| 操作 | アノテーション | 戻り値 |
|------|--------------|--------|
| SELECT (リアクティブ) | `@Query` | `Flow<List<T>>` |
| SELECT (1回) | `@Query` | `suspend T?` |
| INSERT | `@Insert` | `suspend Long` |
| UPSERT | `@Upsert` | `suspend` |
| UPDATE | `@Update` | `suspend` |
| DELETE | `@Delete` / `@Query` | `suspend` |

- `Flow<List<T>>`でデータ変更時に自動再通知
- `flatMapLatest`でフィルター変更時にFlowを切り替え
- `Upsert`で存在すれば更新、なければ挿入
- `animateItem()`でリスト操作時のアニメーション

---

8種類のAndroidアプリテンプレート（Room CRUD実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Roomリレーション](https://zenn.dev/myougatheaxo/articles/android-compose-room-relations-2026)
- [Roomマイグレーション](https://zenn.dev/myougatheaxo/articles/android-compose-room-migration-2026)
- [Paging3+Room](https://zenn.dev/myougatheaxo/articles/android-compose-paging3-compose-2026)
