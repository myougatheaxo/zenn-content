---
title: "Pull-to-Refresh実装ガイド — Composeでスワイプ更新を実装する"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

リストを下に引っ張って更新する**Pull-to-Refresh**をMaterial3 Composeで実装する方法を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.compose.material3:material3:1.3.0")
}
```

---

## 基本実装

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun RefreshableList(viewModel: TaskViewModel) {
    val tasks by viewModel.tasks.collectAsStateWithLifecycle()
    val isRefreshing by viewModel.isRefreshing.collectAsStateWithLifecycle()

    PullToRefreshBox(
        isRefreshing = isRefreshing,
        onRefresh = { viewModel.refresh() }
    ) {
        LazyColumn(Modifier.fillMaxSize()) {
            items(tasks, key = { it.id }) { task ->
                TaskItem(task)
            }
        }
    }
}
```

---

## ViewModel

```kotlin
class TaskViewModel(private val repository: TaskRepository) : ViewModel() {
    private val _isRefreshing = MutableStateFlow(false)
    val isRefreshing: StateFlow<Boolean> = _isRefreshing.asStateFlow()

    val tasks: StateFlow<List<Task>> = repository.observeTasks()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun refresh() {
        viewModelScope.launch {
            _isRefreshing.value = true
            try {
                repository.refreshFromNetwork()
            } catch (e: Exception) {
                // エラー処理
            } finally {
                _isRefreshing.value = false
            }
        }
    }
}
```

---

## カスタムインジケーター

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun CustomRefreshList(viewModel: TaskViewModel) {
    val isRefreshing by viewModel.isRefreshing.collectAsStateWithLifecycle()
    val state = rememberPullToRefreshState()

    PullToRefreshBox(
        isRefreshing = isRefreshing,
        onRefresh = { viewModel.refresh() },
        state = state,
        indicator = {
            PullToRefreshDefaults.Indicator(
                modifier = Modifier.align(Alignment.TopCenter),
                isRefreshing = isRefreshing,
                state = state,
                color = MaterialTheme.colorScheme.primary
            )
        }
    ) {
        LazyColumn(Modifier.fillMaxSize()) {
            // コンテンツ
        }
    }
}
```

---

## 空リストの場合

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun EmptyRefreshableList(viewModel: TaskViewModel) {
    val tasks by viewModel.tasks.collectAsStateWithLifecycle()
    val isRefreshing by viewModel.isRefreshing.collectAsStateWithLifecycle()

    PullToRefreshBox(
        isRefreshing = isRefreshing,
        onRefresh = { viewModel.refresh() }
    ) {
        if (tasks.isEmpty() && !isRefreshing) {
            // 空状態の表示
            Box(
                Modifier.fillMaxSize(),
                contentAlignment = Alignment.Center
            ) {
                Column(horizontalAlignment = Alignment.CenterHorizontally) {
                    Icon(
                        Icons.Default.Inbox,
                        null,
                        Modifier.size(64.dp),
                        tint = MaterialTheme.colorScheme.outline
                    )
                    Spacer(Modifier.height(16.dp))
                    Text("データがありません")
                    Text(
                        "下に引っ張って更新",
                        style = MaterialTheme.typography.bodySmall
                    )
                }
            }
        } else {
            LazyColumn(Modifier.fillMaxSize()) {
                items(tasks, key = { it.id }) { task ->
                    TaskItem(task)
                }
            }
        }
    }
}
```

---

## まとめ

- `PullToRefreshBox`で簡単に実装
- ViewModelで`isRefreshing`状態を管理
- `finally`ブロックで確実にリフレッシュ終了
- 空リスト時もPull-to-Refreshを有効に
- カスタムインジケーターで色を変更可能

---

8種類のAndroidアプリテンプレート（Pull-to-Refresh対応可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [LazyColumn完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-lazycolumn-guide-2026)
- [Retrofit入門](https://zenn.dev/myougatheaxo/articles/android-retrofit-api-2026)
- [オフラインファースト設計](https://zenn.dev/myougatheaxo/articles/android-offline-first-2026)
