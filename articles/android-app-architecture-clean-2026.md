---
title: "Clean Architecture入門 — Androidアプリの3層設計パターン"
emoji: "🏛️"
type: "tech"
topics: ["android", "kotlin", "architecture", "cleanarchitecture"]
published: true
---

## この記事で学べること

GoogleのAndroidアーキテクチャガイドに沿った**3層設計（UI層・Domain層・Data層）**を解説します。

---

## 3層アーキテクチャ

```
┌──────────────────────────┐
│       UI Layer           │  ← Screen, ViewModel
│  (Compose + ViewModel)   │
├──────────────────────────┤
│      Domain Layer        │  ← UseCase（オプション）
│  (UseCase / Interactor)  │
├──────────────────────────┤
│       Data Layer         │  ← Repository, DataSource
│  (Repository + Source)   │
└──────────────────────────┘
```

---

## Data Layer

```kotlin
// Entity（Room）
@Entity(tableName = "tasks")
data class TaskEntity(
    @PrimaryKey val id: String,
    val title: String,
    val isCompleted: Boolean,
    val createdAt: Long
)

// DAO
@Dao
interface TaskDao {
    @Query("SELECT * FROM tasks ORDER BY createdAt DESC")
    fun observeAll(): Flow<List<TaskEntity>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(task: TaskEntity)

    @Delete
    suspend fun delete(task: TaskEntity)
}

// Repository
class TaskRepository(private val dao: TaskDao) {
    fun observeTasks(): Flow<List<Task>> =
        dao.observeAll().map { entities ->
            entities.map { it.toTask() }
        }

    suspend fun addTask(title: String) {
        val entity = TaskEntity(
            id = UUID.randomUUID().toString(),
            title = title,
            isCompleted = false,
            createdAt = System.currentTimeMillis()
        )
        dao.insert(entity)
    }

    suspend fun toggleComplete(task: Task) {
        dao.insert(task.copy(isCompleted = !task.isCompleted).toEntity())
    }
}
```

---

## Domain Layer（UseCase）

```kotlin
class GetTasksUseCase(private val repository: TaskRepository) {
    operator fun invoke(): Flow<List<Task>> =
        repository.observeTasks()
}

class AddTaskUseCase(private val repository: TaskRepository) {
    suspend operator fun invoke(title: String) {
        if (title.isBlank()) throw IllegalArgumentException("タイトルは必須です")
        repository.addTask(title)
    }
}

class ToggleTaskUseCase(private val repository: TaskRepository) {
    suspend operator fun invoke(task: Task) {
        repository.toggleComplete(task)
    }
}
```

**Domain層はオプション**。ビジネスロジックが複雑な場合のみ導入。シンプルなアプリではViewModelから直接Repositoryを呼んでOK。

---

## UI Layer

```kotlin
class TaskViewModel(
    private val getTasks: GetTasksUseCase,
    private val addTask: AddTaskUseCase,
    private val toggleTask: ToggleTaskUseCase
) : ViewModel() {

    val tasks: StateFlow<List<Task>> = getTasks()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun onAddTask(title: String) {
        viewModelScope.launch {
            try {
                addTask(title)
            } catch (e: IllegalArgumentException) {
                // バリデーションエラー処理
            }
        }
    }

    fun onToggleTask(task: Task) {
        viewModelScope.launch {
            toggleTask(task)
        }
    }
}
```

---

## ファイル構成

```
app/src/main/java/com/example/myapp/
├── data/
│   ├── local/
│   │   ├── TaskDao.kt
│   │   ├── TaskEntity.kt
│   │   └── AppDatabase.kt
│   └── repository/
│       └── TaskRepository.kt
├── domain/
│   ├── model/
│   │   └── Task.kt
│   └── usecase/
│       ├── GetTasksUseCase.kt
│       ├── AddTaskUseCase.kt
│       └── ToggleTaskUseCase.kt
└── ui/
    ├── screen/
    │   └── TaskScreen.kt
    └── viewmodel/
        └── TaskViewModel.kt
```

---

## 依存の方向

```
UI → Domain → Data
  (ViewModel)  (UseCase)  (Repository)
```

**上位が下位に依存**。Data層はUI層を知らない。Domain層はどの層にも依存しない（純粋Kotlin）。

---

## 使い分けガイド

| アプリ規模 | 推奨構成 |
|-----------|---------|
| 画面2〜3個 | ViewModel + Repository（Domain層なし） |
| 画面5〜10個 | ViewModel + Repository + 一部UseCase |
| 画面10個以上 | フル3層 + マルチモジュール |

---

## まとめ

- **Data層**: Room DAO + Repository
- **Domain層**: UseCase（ビジネスロジック集約、オプション）
- **UI層**: Compose + ViewModel
- 依存の方向: UI → Domain → Data
- 小規模アプリはDomain層を省略してOK

---

8種類のAndroidアプリテンプレート（Clean Architecture設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [MVVM Architecture完全ガイド](https://zenn.dev/myougatheaxo/articles/android-mvvm-architecture-2026)
- [ViewModel + StateFlow入門](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
- [DI — Manual vs Hilt vs Koin](https://zenn.dev/myougatheaxo/articles/android-di-manual-vs-hilt-2026)
