---
title: "Room Database完全ガイド — ComposeアプリのローカルDB実装"
emoji: "🗄️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Room**を使ったローカルデータベースの実装方法を、Entity・DAO・Database・ViewModelの構成で解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
plugins {
    id("com.google.devtools.ksp") version "2.0.0-1.0.22"
}

dependencies {
    implementation("androidx.room:room-runtime:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")
    ksp("androidx.room:room-compiler:2.6.1")
}
```

---

## Entity（テーブル定義）

```kotlin
@Entity(tableName = "tasks")
data class TaskEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Int = 0,
    val title: String,
    val description: String = "",
    val isCompleted: Boolean = false,
    val createdAt: Long = System.currentTimeMillis()
)
```

---

## DAO（データアクセス）

```kotlin
@Dao
interface TaskDao {
    @Query("SELECT * FROM tasks ORDER BY createdAt DESC")
    fun observeAll(): Flow<List<TaskEntity>>

    @Query("SELECT * FROM tasks WHERE isCompleted = :completed")
    fun observeByStatus(completed: Boolean): Flow<List<TaskEntity>>

    @Query("SELECT * FROM tasks WHERE id = :id")
    suspend fun getById(id: Int): TaskEntity?

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(task: TaskEntity)

    @Update
    suspend fun update(task: TaskEntity)

    @Delete
    suspend fun delete(task: TaskEntity)

    @Query("DELETE FROM tasks WHERE isCompleted = 1")
    suspend fun deleteCompleted()
}
```

**ポイント**:
- `Flow<List<T>>` でリアルタイム監視（`suspend`不要）
- 単発取得は `suspend fun`

---

## Database

```kotlin
@Database(entities = [TaskEntity::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun taskDao(): TaskDao

    companion object {
        @Volatile
        private var INSTANCE: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "app_database"
                ).build().also { INSTANCE = it }
            }
        }
    }
}
```

---

## Repository

```kotlin
class TaskRepository(private val dao: TaskDao) {
    fun observeAll(): Flow<List<TaskEntity>> = dao.observeAll()

    fun observeByStatus(completed: Boolean): Flow<List<TaskEntity>> =
        dao.observeByStatus(completed)

    suspend fun add(title: String, description: String = "") {
        dao.insert(TaskEntity(title = title, description = description))
    }

    suspend fun toggleComplete(task: TaskEntity) {
        dao.update(task.copy(isCompleted = !task.isCompleted))
    }

    suspend fun delete(task: TaskEntity) {
        dao.delete(task)
    }
}
```

---

## ViewModel

```kotlin
class TaskViewModel(private val repository: TaskRepository) : ViewModel() {
    val tasks: StateFlow<List<TaskEntity>> = repository.observeAll()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun addTask(title: String) {
        viewModelScope.launch {
            repository.add(title)
        }
    }

    fun toggleComplete(task: TaskEntity) {
        viewModelScope.launch {
            repository.toggleComplete(task)
        }
    }

    fun deleteTask(task: TaskEntity) {
        viewModelScope.launch {
            repository.delete(task)
        }
    }
}
```

---

## Compose UI

```kotlin
@Composable
fun TaskListScreen(viewModel: TaskViewModel) {
    val tasks by viewModel.tasks.collectAsStateWithLifecycle()
    var showDialog by remember { mutableStateOf(false) }

    Scaffold(
        floatingActionButton = {
            FloatingActionButton(onClick = { showDialog = true }) {
                Icon(Icons.Default.Add, "追加")
            }
        }
    ) { padding ->
        LazyColumn(
            contentPadding = padding,
            modifier = Modifier.fillMaxSize()
        ) {
            items(tasks, key = { it.id }) { task ->
                TaskItem(
                    task = task,
                    onToggle = { viewModel.toggleComplete(task) },
                    onDelete = { viewModel.deleteTask(task) }
                )
            }
        }
    }

    if (showDialog) {
        AddTaskDialog(
            onDismiss = { showDialog = false },
            onConfirm = { title ->
                viewModel.addTask(title)
                showDialog = false
            }
        )
    }
}
```

---

## マイグレーション

```kotlin
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE tasks ADD COLUMN priority INTEGER NOT NULL DEFAULT 0")
    }
}

Room.databaseBuilder(context, AppDatabase::class.java, "app_database")
    .addMigrations(MIGRATION_1_2)
    .build()
```

---

## TypeConverter（カスタム型）

```kotlin
class Converters {
    @TypeConverter
    fun fromStringList(value: List<String>): String = value.joinToString(",")

    @TypeConverter
    fun toStringList(value: String): List<String> =
        if (value.isEmpty()) emptyList() else value.split(",")
}

@Database(entities = [TaskEntity::class], version = 1)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    // ...
}
```

---

## まとめ

- `@Entity` → テーブル、`@Dao` → クエリ、`@Database` → DB本体
- `Flow<List<T>>` でリアルタイム監視
- KSPでコンパイル時コード生成（kapt非推奨）
- `Migration`でスキーマ変更を安全に
- `@TypeConverters`でカスタム型を保存

---

8種類のAndroidアプリテンプレート（Room Database設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [DataStore完全ガイド](https://zenn.dev/myougatheaxo/articles/android-datastore-guide-2026)
- [オフラインファースト設計](https://zenn.dev/myougatheaxo/articles/android-offline-first-2026)
- [ViewModel完全ガイド](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
