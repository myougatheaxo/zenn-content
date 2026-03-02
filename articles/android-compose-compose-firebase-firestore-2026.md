---
title: "Compose Firebase Firestore完全ガイド — CRUD/リアルタイム同期/クエリ"
emoji: "🔥"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "firebase"]
published: true
---

## この記事で学べること

**Compose Firebase Firestore**（CRUD操作、リアルタイムリスナー、クエリ、Compose統合）を解説します。

---

## 基本CRUD

```kotlin
class TaskRepository @Inject constructor() {
    private val db = Firebase.firestore
    private val tasksRef = db.collection("tasks")

    fun getTasks(): Flow<List<Task>> = callbackFlow {
        val listener = tasksRef.orderBy("createdAt", Query.Direction.DESCENDING)
            .addSnapshotListener { snapshot, error ->
                if (error != null) { close(error); return@addSnapshotListener }
                val tasks = snapshot?.documents?.mapNotNull { it.toObject<Task>()?.copy(id = it.id) } ?: emptyList()
                trySend(tasks)
            }
        awaitClose { listener.remove() }
    }

    suspend fun addTask(task: Task): String {
        val doc = tasksRef.add(task.copy(createdAt = Timestamp.now())).await()
        return doc.id
    }

    suspend fun updateTask(id: String, updates: Map<String, Any>) {
        tasksRef.document(id).update(updates).await()
    }

    suspend fun deleteTask(id: String) {
        tasksRef.document(id).delete().await()
    }
}

data class Task(
    val id: String = "",
    val title: String = "",
    val completed: Boolean = false,
    val createdAt: Timestamp? = null
)
```

---

## ViewModel

```kotlin
@HiltViewModel
class TaskViewModel @Inject constructor(
    private val repository: TaskRepository
) : ViewModel() {

    val tasks = repository.getTasks()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun addTask(title: String) {
        viewModelScope.launch { repository.addTask(Task(title = title)) }
    }

    fun toggleComplete(task: Task) {
        viewModelScope.launch {
            repository.updateTask(task.id, mapOf("completed" to !task.completed))
        }
    }

    fun deleteTask(id: String) {
        viewModelScope.launch { repository.deleteTask(id) }
    }
}
```

---

## Compose UI

```kotlin
@Composable
fun TaskListScreen(viewModel: TaskViewModel = hiltViewModel()) {
    val tasks by viewModel.tasks.collectAsStateWithLifecycle()
    var newTaskTitle by remember { mutableStateOf("") }

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        Row(Modifier.fillMaxWidth()) {
            OutlinedTextField(value = newTaskTitle, onValueChange = { newTaskTitle = it },
                modifier = Modifier.weight(1f), placeholder = { Text("新しいタスク") })
            IconButton(onClick = { viewModel.addTask(newTaskTitle); newTaskTitle = "" },
                enabled = newTaskTitle.isNotBlank()) { Icon(Icons.Default.Add, "追加") }
        }

        LazyColumn(Modifier.fillMaxSize(), verticalArrangement = Arrangement.spacedBy(4.dp)) {
            items(tasks, key = { it.id }) { task ->
                ListItem(
                    headlineContent = { Text(task.title, textDecoration = if (task.completed) TextDecoration.LineThrough else null) },
                    leadingContent = { Checkbox(checked = task.completed, onCheckedChange = { viewModel.toggleComplete(task) }) },
                    trailingContent = { IconButton(onClick = { viewModel.deleteTask(task.id) }) { Icon(Icons.Default.Delete, "削除") } }
                )
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `addSnapshotListener` | リアルタイム同期 |
| `callbackFlow` | Flow変換 |
| `add/update/delete` | CRUD操作 |
| `orderBy/whereEqualTo` | クエリ |

- `callbackFlow`でFirestoreリスナーをFlow化
- リアルタイム同期で即座にUI反映
- `await()`でsuspend対応
- `toObject<T>()`でKotlinデータクラスに自動変換

---

8種類のAndroidアプリテンプレート（Firebase対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Firebase Auth](https://zenn.dev/myougatheaxo/articles/android-compose-compose-firebase-auth-2026)
- [Firebase Messaging](https://zenn.dev/myougatheaxo/articles/android-compose-compose-firebase-messaging-2026)
- [Flow CallbackFlow](https://zenn.dev/myougatheaxo/articles/android-compose-flow-callback-flow-2026)
