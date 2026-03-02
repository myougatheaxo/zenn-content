---
title: "Cloud Firestore + Compose連携ガイド — CRUD/リアルタイム同期"
emoji: "☁️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "firebase"]
published: true
---

## この記事で学べること

**Cloud Firestore**のCRUD操作とComposeでのリアルタイム同期を解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation(platform("com.google.firebase:firebase-bom:33.0.0"))
    implementation("com.google.firebase:firebase-firestore-ktx")
}
```

---

## データモデル

```kotlin
data class Task(
    val id: String = "",
    val title: String = "",
    val completed: Boolean = false,
    val createdAt: Long = System.currentTimeMillis()
)
```

---

## FirestoreRepository

```kotlin
class TaskRepository(
    private val db: FirebaseFirestore = Firebase.firestore
) {
    private val collection = db.collection("tasks")

    // リアルタイム監視
    fun getTasks(): Flow<List<Task>> = callbackFlow {
        val listener = collection
            .orderBy("createdAt", Query.Direction.DESCENDING)
            .addSnapshotListener { snapshot, error ->
                if (error != null) {
                    close(error)
                    return@addSnapshotListener
                }
                val tasks = snapshot?.documents?.map { doc ->
                    doc.toObject(Task::class.java)?.copy(id = doc.id)
                        ?: Task()
                } ?: emptyList()
                trySend(tasks)
            }
        awaitClose { listener.remove() }
    }

    // 追加
    suspend fun addTask(title: String) {
        collection.add(Task(title = title)).await()
    }

    // 更新
    suspend fun updateTask(id: String, completed: Boolean) {
        collection.document(id).update("completed", completed).await()
    }

    // 削除
    suspend fun deleteTask(id: String) {
        collection.document(id).delete().await()
    }
}
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
        viewModelScope.launch {
            repository.addTask(title)
        }
    }

    fun toggleTask(id: String, completed: Boolean) {
        viewModelScope.launch {
            repository.updateTask(id, completed)
        }
    }

    fun deleteTask(id: String) {
        viewModelScope.launch {
            repository.deleteTask(id)
        }
    }
}
```

---

## Compose画面

```kotlin
@Composable
fun TaskListScreen(viewModel: TaskViewModel = hiltViewModel()) {
    val tasks by viewModel.tasks.collectAsStateWithLifecycle()
    var newTaskTitle by remember { mutableStateOf("") }

    Scaffold(
        topBar = { TopAppBar(title = { Text("タスク一覧") }) }
    ) { padding ->
        Column(Modifier.padding(padding)) {
            // 入力欄
            Row(Modifier.padding(16.dp)) {
                OutlinedTextField(
                    value = newTaskTitle,
                    onValueChange = { newTaskTitle = it },
                    label = { Text("新しいタスク") },
                    modifier = Modifier.weight(1f)
                )
                IconButton(
                    onClick = {
                        if (newTaskTitle.isNotBlank()) {
                            viewModel.addTask(newTaskTitle)
                            newTaskTitle = ""
                        }
                    }
                ) {
                    Icon(Icons.Default.Add, "追加")
                }
            }

            // タスクリスト
            LazyColumn {
                items(tasks, key = { it.id }) { task ->
                    TaskItem(
                        task = task,
                        onToggle = { viewModel.toggleTask(task.id, !task.completed) },
                        onDelete = { viewModel.deleteTask(task.id) }
                    )
                }
            }
        }
    }
}
```

---

## まとめ

- `addSnapshotListener`をFlowに変換でリアルタイム同期
- `collection.add()`/`document().update()`/`document().delete()`でCRUD
- `toObject()`でFirestoreドキュメント→データクラス変換
- `orderBy`+`Direction.DESCENDING`でソート
- `stateIn(WhileSubscribed(5000))`で画面非表示時にリスナー解除
- オフライン対応はFirestoreが自動キャッシュ

---

8種類のAndroidアプリテンプレート（データ管理実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Firebase Auth連携](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-auth-2026)
- [Room DB マイグレーション](https://zenn.dev/myougatheaxo/articles/android-compose-room-migration-2026)
- [Flow+Lifecycle](https://zenn.dev/myougatheaxo/articles/android-compose-flow-lifecycle-2026)
