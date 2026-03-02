---
title: "ドラッグ&ドロップ/並び替え完全ガイド — LazyColumn Reorder"
emoji: "🔀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**ドラッグ&ドロップ**（LazyColumn並び替え、DragAndDrop API、hapticフィードバック）を解説します。

---

## LazyColumn並び替え

```kotlin
@Composable
fun ReorderableList(
    viewModel: TaskViewModel = hiltViewModel()
) {
    val tasks by viewModel.tasks.collectAsStateWithLifecycle()
    val lazyListState = rememberLazyListState()

    val reorderState = rememberReorderableLazyListState(lazyListState) { from, to ->
        viewModel.reorder(from.index, to.index)
    }

    LazyColumn(
        state = lazyListState,
        modifier = Modifier.fillMaxSize()
    ) {
        itemsIndexed(tasks, key = { _, task -> task.id }) { index, task ->
            ReorderableItem(reorderState, key = task.id) { isDragging ->
                val elevation by animateDpAsState(if (isDragging) 8.dp else 0.dp)

                Card(
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(horizontal = 16.dp, vertical = 4.dp)
                        .shadow(elevation),
                    colors = CardDefaults.cardColors(
                        containerColor = if (isDragging)
                            MaterialTheme.colorScheme.primaryContainer
                        else MaterialTheme.colorScheme.surface
                    )
                ) {
                    Row(
                        Modifier.padding(16.dp),
                        verticalAlignment = Alignment.CenterVertically
                    ) {
                        Icon(
                            Icons.Default.DragHandle,
                            contentDescription = "並び替え",
                            modifier = Modifier.detectReorder(reorderState)
                        )
                        Spacer(Modifier.width(12.dp))
                        Text(task.title, Modifier.weight(1f))
                    }
                }
            }
        }
    }
}
```

---

## ViewModel（並び替えロジック）

```kotlin
@HiltViewModel
class TaskViewModel @Inject constructor(
    private val repository: TaskRepository
) : ViewModel() {

    private val _tasks = MutableStateFlow<List<Task>>(emptyList())
    val tasks = _tasks.asStateFlow()

    fun reorder(fromIndex: Int, toIndex: Int) {
        _tasks.update { list ->
            list.toMutableList().apply {
                add(toIndex, removeAt(fromIndex))
            }
        }
    }

    fun saveOrder() {
        viewModelScope.launch {
            _tasks.value.forEachIndexed { index, task ->
                repository.updateOrder(task.id, index)
            }
        }
    }
}
```

---

## Modifier.dragAndDropでView間ドラッグ

```kotlin
@Composable
fun DragDropExample() {
    var droppedText by remember { mutableStateOf("ここにドロップ") }

    Row(
        Modifier.fillMaxWidth().padding(16.dp),
        horizontalArrangement = Arrangement.SpaceEvenly
    ) {
        // ドラッグ元
        Box(
            modifier = Modifier
                .size(100.dp)
                .background(MaterialTheme.colorScheme.primary, RoundedCornerShape(8.dp))
                .dragAndDropSource {
                    detectTapGestures(onLongPress = {
                        startTransfer(
                            DragAndDropTransferData(
                                ClipData.newPlainText("label", "ドラッグされたデータ")
                            )
                        )
                    })
                },
            contentAlignment = Alignment.Center
        ) {
            Text("ドラッグ", color = Color.White)
        }

        // ドロップ先
        Box(
            modifier = Modifier
                .size(100.dp)
                .background(MaterialTheme.colorScheme.secondary, RoundedCornerShape(8.dp))
                .dragAndDropTarget(
                    shouldStartDragAndDrop = { true },
                    target = remember {
                        object : DragAndDropTarget {
                            override fun onDrop(event: DragAndDropEvent): Boolean {
                                val clip = event.toAndroidDragEvent().clipData
                                droppedText = clip.getItemAt(0).text.toString()
                                return true
                            }
                        }
                    }
                ),
            contentAlignment = Alignment.Center
        ) {
            Text(droppedText, color = Color.White, textAlign = TextAlign.Center)
        }
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| リスト並び替え | `ReorderableLazyList` |
| ドラッグ元 | `dragAndDropSource` |
| ドロップ先 | `dragAndDropTarget` |
| 触覚FB | `performHapticFeedback()` |

- `ReorderableLazyList`で直感的な並び替え
- `dragAndDropSource/Target`でView間ドラッグ
- アニメーションで視覚的フィードバック
- 並び替え結果はDBに永続化

---

8種類のAndroidアプリテンプレート（ドラッグ&ドロップ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-transition-2026)
- [LazyColumn最適化](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-column-tips-2026)
- [Room Flow CRUD](https://zenn.dev/myougatheaxo/articles/android-compose-room-flow-crud-2026)
