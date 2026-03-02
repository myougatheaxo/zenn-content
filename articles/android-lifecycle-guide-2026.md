---
title: "Androidライフサイクル完全ガイド — Activity・ViewModel・Composeの関係"
emoji: "🔄"
type: "tech"
topics: ["android", "kotlin", "lifecycle", "jetpackcompose"]
published: true
---

## この記事で学べること

Androidアプリの**ライフサイクル**を正しく理解すれば、メモリリークやクラッシュを防げます。Activity、ViewModel、Composeそれぞれのライフサイクルを解説します。

---

## Activityのライフサイクル

```
onCreate → onStart → onResume
                          ↓
                     [アプリ使用中]
                          ↓
onPause → onStop → onDestroy
```

| コールバック | タイミング |
|-------------|-----------|
| `onCreate` | Activity生成時（初期化） |
| `onStart` | 画面に表示される直前 |
| `onResume` | ユーザーが操作可能になった |
| `onPause` | 別の画面が前面に来た |
| `onStop` | 完全に見えなくなった |
| `onDestroy` | Activityが破棄される |

---

## ComposeでのライフサイクルAware

```kotlin
@Composable
fun LifecycleAwareScreen() {
    val lifecycleOwner = LocalLifecycleOwner.current

    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_RESUME -> {
                    // 画面が前面に来た（センサー開始など）
                }
                Lifecycle.Event.ON_PAUSE -> {
                    // 画面が背面に行った（センサー停止など）
                }
                else -> {}
            }
        }

        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose {
            lifecycleOwner.lifecycle.removeObserver(observer)
        }
    }
}
```

---

## ViewModelのライフサイクル

```
Activity onCreate ─────────────────────── Activity onDestroy
                                              (設定変更でないとき)
ViewModel created ─────────────────────── ViewModel onCleared
         ↑                                       ↑
    画面回転しても生き残る              画面を完全に離れたとき
```

```kotlin
class HomeViewModel : ViewModel() {
    // 画面回転しても保持される
    private val _items = MutableStateFlow<List<Item>>(emptyList())
    val items: StateFlow<List<Item>> = _items.asStateFlow()

    override fun onCleared() {
        super.onCleared()
        // リソース解放（ViewModel破棄時）
    }
}
```

---

## LaunchedEffect（Composeのライフサイクル）

```kotlin
@Composable
fun TimerScreen() {
    var seconds by remember { mutableIntStateOf(0) }

    // このComposableがCompositionに入ったときに開始
    // Compositionから出たとき（画面遷移など）に自動キャンセル
    LaunchedEffect(Unit) {
        while (true) {
            delay(1000)
            seconds++
        }
    }

    Text("経過: ${seconds}秒")
}
```

| Effect | 用途 |
|--------|------|
| `LaunchedEffect` | Composition時にCoroutine起動 |
| `DisposableEffect` | Composition/Decomposition時のセットアップ/クリーンアップ |
| `SideEffect` | 毎回のRecomposition時に実行 |

---

## collectAsStateWithLifecycle

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel) {
    // ❌ バックグラウンドでもFlowを購読し続ける
    val users by viewModel.users.collectAsState()

    // ✅ 画面が見えているときだけ購読
    val users by viewModel.users.collectAsStateWithLifecycle()
}
```

```kotlin
// build.gradle.kts
implementation("androidx.lifecycle:lifecycle-runtime-compose:2.8.0")
```

`collectAsStateWithLifecycle()`は**画面が非表示のときに自動で購読を停止**します。バッテリー消費を抑えられます。

---

## プロセス再生成（Process Death）

```kotlin
// ❌ remember — プロセス再生成で消える
var query by remember { mutableStateOf("") }

// ✅ rememberSaveable — プロセス再生成でも復元
var query by rememberSaveable { mutableStateOf("") }
```

| シナリオ | remember | rememberSaveable | ViewModel |
|---------|----------|-----------------|-----------|
| Recomposition | 保持 | 保持 | 保持 |
| 画面回転 | リセット | 保持 | 保持 |
| プロセス再生成 | リセット | 保持 | リセット |
| 画面遷移（戻る） | リセット | リセット | リセット |

---

## まとめ

- Activityライフサイクル: onCreate→onResume→onPause→onDestroy
- ViewModelは画面回転でも生存（設定変更に強い）
- `collectAsStateWithLifecycle()`でバッテリー最適化
- `rememberSaveable`でプロセス再生成に対応
- `DisposableEffect`でリソースのクリーンアップ

---

8種類のAndroidアプリテンプレート（正しいライフサイクル管理実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ViewModel + StateFlow入門](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
- [Composeの状態管理入門](https://zenn.dev/myougatheaxo/articles/compose-state-management-2026)
- [Kotlin Coroutines & Flow入門](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-flow-2026)
